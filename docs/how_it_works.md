# How fuse-archive Works

fuse-archive mounts archive files (`.tar`, `.zip`, `.7z`, `.tar.gz`, `.tar.gpg`, etc.) as
read-only FUSE file systems. A user runs one command and the archive appears as a directory
tree they can browse with any normal tool. No extraction to disk; the archive file itself is
never modified.

This document explains the internals in depth.

---

## Overview

The lifecycle of a mount has two phases:

1. **Load phase** — the archive is opened, every entry is scanned, and an in-memory
   virtual file system (VFS) tree is built. File _content_ may be cached to a temporary
   file at this point (full-cache mode) or left for later (lazy/no-cache modes).
2. **Serve phase** — `fuse_main()` takes over. The process daemonises and handles
   filesystem calls from the kernel via FUSE callbacks until the mount point is unmounted.

```
fuse-archive archive.tar.gz /mnt/point
       │
       ├─ main() ──────── parses options
       │                  opens archive file (fd kept open)
       │                  Tree::Load() ── Reader reads all entries ──► Node tree in RAM
       │                  creates mount point directory if needed
       │
       └─ fuse_main() ── daemonises
                         kernel ◄──► FUSE callbacks (getattr, read, readdir, …)
                                     ↓
                                   Tree / Node / Reader
```

---

## Key Data Structures

### `ArchiveDescriptor` (`lib/common.h`)

One instance per archive file passed on the command line. Holds:

- `path` — the file path argument
- `fd` — an open `O_RDONLY` file descriptor kept alive for the entire process
- `size` — file size in bytes, used for progress reporting
- `format` — detected `ArchiveFormat` (e.g. RAW, or whatever libarchive reports)
- `filter_count` — how many compression/encoding filters are active (e.g. 1 for `.tar.gz`)
- `is_seekable_format` — true for ZIP/7z, which require random access to decompressed data
- `name_without_extension` — the basename stripped of archive extensions; used for the VFS
  node name when mounting multiple archives

All `Reader` instances for the same archive share a pointer back to this descriptor.

### `Node` (`lib/node.h`)

Every file, directory, symlink, and special file in the VFS is a `Node`. Notable fields:

| Field                  | Purpose                                                                                |
| ---------------------- | -------------------------------------------------------------------------------------- |
| `index_within_archive` | 1-based position in the archive; used to seek the Reader back to this entry            |
| `descriptor`           | which `ArchiveDescriptor` (archive file) contains this entry                           |
| `cache_offset`         | byte offset into the cache file where this entry's data starts; `-1` if not yet cached |
| `cached_size`          | bytes of this file written to the cache file so far                                    |
| `size`                 | total decompressed size                                                                |
| `hardlink_target`      | non-null for hard links; points at the canonical node                                  |
| `reader`               | a `Reader::Ptr` pinned to this node during lazy caching                                |
| `holes`                | sparse-file hole descriptors (pairs of `[from, to)` byte ranges)                       |
| `attributes`           | extended attributes extracted from the archive                                         |
| `by_parent`            | intrusive slist hook — links this node into its parent's `children` list               |
| `by_path`              | intrusive unordered-set hook — links this node into `Tree::nodes_by_path_`             |

The `parent` + `name` fields establish the tree structure. The `by_path` index gives O(1)
average-case lookup of any node by its full path string.

Nodes are allocated with `new` during the load phase and freed when the `Tree` destructor
runs. The intrusive containers do not own the allocations; `NodesByPath::~NodesByPath`
calls `clear_and_dispose(std::default_delete<Node>())` to delete them.

### `Tree` (`lib/tree.h`, `lib/tree.cc`)

The central object. Owns:

- `nodes_by_path_` — a Boost.Intrusive unordered set of all nodes, keyed by their full
  path hash (a Boost `hash_combine` over each character). This is the primary lookup
  index used by every FUSE callback.
- `root_` — raw pointer to the root `/` node.
- `archives_` — the `ArchiveDescriptor` list, kept alive for the life of the process.
- `recycled_readers_` — a Boost.Intrusive list of `Reader` objects that have finished
  an I/O request and been returned to a pool rather than destroyed.
- `cache_fd_` — the single shared temporary file used by all cached entries.
- `cache_size_` — current allocated size of the cache file (grows monotonically).
- `password_` — optional passphrase for encrypted archives.
- `options_` — the parsed `Options` struct (caching strategy, masks, feature flags).

The tree is created once before `fuse_main()` and lives for the duration of the mount.
It is passed to `fuse_main()` as `private_data` and recovered in every FUSE callback via
`fuse_get_context()->private_data`.

### `Reader` (`lib/reader.h`, `lib/reader.cc`)

A Reader wraps one `archive_read` libarchive handle pointed at one archive file,
positioned at one entry. It provides:

- Sequential iteration via `NextEntry()`
- Forward-seeking via `AdvanceIndex()` / `AdvanceOffset()`
- A 256 KiB rolling buffer that allows serving slightly out-of-order reads without
  re-decompressing from the start
- A `CacheEntryData()` method that streams an entry's content into the cache file

Readers are expensive to create (libarchive must decompress from the beginning of the
archive to reach entry N). Therefore, they are **recycled**: when a FUSE `read` call
finishes, the `Reader` is not destroyed but placed in `Tree::recycled_readers_`. The next
`read` that needs the same archive at a nearby offset can reuse it.

---

## Phase 1: Loading the Archive

`Tree::Load()` in `tree.cc` drives this phase.

### 1.1 Opening Archive Files

Each archive path is opened with `open(O_RDONLY)`. The file descriptor is kept open until
the process exits (or until full-cache mode has cached everything, at which point it is
closed early). fuse-archive never writes to the archive file.

### 1.2 Format and Filter Detection

`Reader::SetFormat()` in `reader.cc` is called when the first `Reader` is constructed.
It inspects the filename extension chain (e.g. `.tar.gz` → filter `gz`, format `tar`)
and configures libarchive accordingly.

**Filter extensions** (`.gz`, `.bz2`, `.xz`, `.zst`, `.gpg`, `.br`, `.lz4`, …) are
handled by `Reader::SetFilter()`. Most map directly to `ARCHIVE_FILTER_*` constants.
Filters without native libarchive support (`.gpg`, `.pgp`, `.asc`, `.base64`, `.brotli`,
`.grzip`, `.lzop`) are handled by `archive_read_append_filter_program()`, which forks
an external subprocess and pipes the archive data through it.

> **macOS note:** On macOS, libarchive spawns external filter programs via `posix_spawnp`
> with `POSIX_SPAWN_CLOEXEC_DEFAULT` and a null environment. Non-platform (e.g. Homebrew)
> binaries invoked this way fail silently. For GPG, fuse-archive writes a small shell
> script to `/tmp/fuse_archive_gpg_XXXXXX.sh` at startup and uses that as the filter
> command instead. `/bin/sh` is a macOS platform binary that works correctly, and it
> `exec`s the real `gpg` binary. The script is unlinked on process exit.

**Format extensions** (`.tar`, `.zip`, `.7z`, `.rar`, `.cpio`, `.xar`, `.warc`, …) map to
`archive_read_support_format_*` calls. Some combinations (e.g. `.tar.gz`, `.tar.bz2`) are
recognised as a unit via `SetCompressedTarFormat()`, which configures both filter and
format in one step.

If no recognised extension is found and bidding is enabled (`-o bidding`), fuse-archive
enables all filters and formats and lets libarchive's **bidding** system pick the right
one by looking at the magic bytes of the file.

**Stacked filters** (e.g. `.tar.gz.gpg` — GPG wrapping gzip wrapping tar) are supported
up to `max_filter_count` layers (default 1, configurable via `-o maxfilters=N`).

### 1.3 Iterating Entries

```
while (r->NextEntry()) {
    CheckRawArchive(*r);
    ProcessEntry(*r, path, local_root);
}
ResolveHardlinks();
```

`NextEntry()` calls `archive_read_next_header()`. For each entry,
`Tree::ProcessEntry()` decides what to do:

- **Hard link** — recorded in `Tree::hardlinks_` for deferred resolution after all
  entries are seen (targets might appear later in the archive).
- **Skipped types** — special files, symlinks, or directories may be skipped
  depending on `-o nospecials`, `-o nosymlinks`, `-o nodirs` options.
- **Directory** — `GetOrCreateDirNode()` ensures the directory and all missing
  ancestors exist as `Node` objects.
- **Regular file** — a new `Node` is created, attached to its parent directory,
  inserted into `nodes_by_path_`, and its content is handled per the caching strategy.

### 1.4 Collision Handling

Archive entries are not required to have unique paths. When two entries map to the same
VFS path, `Tree::RenameIfCollision()` appends ` (1)`, ` (2)`, etc. before the file
extension (e.g. `file.txt` → `file (1).txt`). If a non-directory entry occupies a path
that also needs to be a directory (because another entry is inside it), the file is
renamed first, and the directory is created in its place.

### 1.5 Hard Link Resolution

`Tree::ResolveHardlinks()` runs after all entries are processed. It resolves each
deferred hard link by looking up the target path in `nodes_by_path_`. The behaviour
depends on the `-o nohardlinks` option:

- **With hardlinks (default):** the new node shares the same inode number as the target;
  `target->nlink` is incremented. The new node's `hardlink_target` pointer is set.
- **Without hardlinks:** each hard link gets its own inode and is treated as an
  independent copy.

### 1.6 Caching Strategies

Three modes are supported, selected with `-o fullcache` (default), `-o lazycache`, or
`-o nocache`.

#### Full cache (default)

During `ProcessEntry()`, `Reader::CacheEntryData()` streams the entire decompressed
content of every file into `cache_fd_`. The file's `Node` stores `cache_offset` (byte
offset in the cache file) and `size`. After loading, the archive file descriptor is
closed; all subsequent reads are satisfied from the cache file via `pread()`.

The cache file is a temporary file created with `CreateCacheFile()` (`tmpfile()` or
`memfd_create()` on Linux). It lives as long as the mount.

**ZIP / seekable formats:** ZIP files store metadata at the _end_ of the file and require
random access. Under full-cache mode, fuse-archive first decompresses the entire ZIP
into a cache file, then re-opens _that cache file_ as the archive, so libarchive can seek
within it freely.

**Sparse files:** When writing to the cache file, `FileDescriptor::WriteBytesAndSkipHoles()`
detects runs of NUL bytes and punches holes using `lseek(fd, n, SEEK_DATA/SEEK_HOLE)` or
`fallocate(FALLOC_FL_PUNCH_HOLE)`. The `Node::holes` vector records the hole ranges so
that `lseek(SEEK_DATA)` / `lseek(SEEK_HOLE)` via the `Seek` FUSE callback returns
correct offsets, and `statfs` block counts are accurate.

#### Lazy cache

No caching happens during load. The first `read()` on a file triggers
`Tree::CacheUpTo(node, offset + len)`, which decompresses and writes to the cache file
up to the requested byte. A `Reader` is kept pinned to the node (`node->reader`) while
caching is in progress and returned to `recycled_readers_` when the file is closed.

The cache file is shared across all files, so once a byte range has been cached it can
be served by `pread()` without touching the archive again.

#### No cache

Every `read()` decompresses on the fly by calling `Tree::GetReader()` to find or create
a `Reader` positioned at the right archive entry and offset. The rolling buffer in
`Reader` allows the kernel's occasional out-of-order reads to be served cheaply, but
large backwards seeks require re-opening the archive from the beginning.

This mode is memory-efficient but performs poorly for random access patterns and is
explicitly warned against for filtered archives (`.tar.gz`, etc.) because every seek
triggers full re-decompression from the start.

### 1.7 Directory Trimming

When `-o trim` is active (the default), `Tree::Trim()` collapses chains of single-child
directories after loading. Archives that wrap content in a top-level directory
(`archive/file1`, `archive/file2`, …) get that redundant wrapper removed, so the mount
point shows `file1`, `file2` directly. The operation de-indexes the inner directory nodes,
moves their children up, and deletes the now-empty intermediate nodes.

### 1.8 Multiple Archives and Merging

When multiple archive paths are given, they are loaded in order. By default (`-o merge`),
all entries appear merged at the root of the mount point. With `-o nomerge`, each archive
gets its own sub-directory named after the archive file (without extension). Collision
handling works the same way across archives.

---

## Phase 2: Serving the File System

After loading, `fuse_main()` is called. It daemonises the process and starts the FUSE
event loop, calling the handlers registered via `GetFuseOperations()` in `fuse_ops.cc`.

The handlers are all read-only; fuse-archive never modifies anything.

### FUSE Callbacks

| Callback    | Implementation | Notes                                                                            |
| ----------- | -------------- | -------------------------------------------------------------------------------- |
| `getattr`   | `GetAttr`      | Calls `Node::GetStat()` — assembles `struct stat` from node fields               |
| `readlink`  | `ReadLink`     | Returns `Node::symlink` string                                                   |
| `open`      | `Open`         | Allocates a `FileHandle{node, reader}` stored in `fi->fh`; increments `fd_count` |
| `read`      | `Read`         | Core I/O path; see below                                                         |
| `release`   | `Release`      | Frees `FileHandle`; decrements `fd_count`                                        |
| `opendir`   | `OpenDir`      | Stores `Node*` in `fi->fh`; sets `fi->cache_readdir = true` on FUSE 3            |
| `readdir`   | `ReadDir`      | Iterates `Node::children`, calls filler with stat if FUSE_READDIR_PLUS           |
| `statfs`    | `StatFs`       | Reports `Tree::GetBlockCount()` and `GetInodeCount()`                            |
| `getxattr`  | `GetXattr`     | Looks up `Node::attributes` via the `HashedString` interning table               |
| `listxattr` | `ListXattr`    | Concatenates NUL-terminated attribute names into the buffer                      |
| `init`      | `Init`         | Sets `cfg->use_ino = true`, `cfg->nullpath_ok = true`                            |
| `lseek`     | `Seek`         | Implements `SEEK_DATA` / `SEEK_HOLE` via `Node::SparseSeek()`                    |

### The `Read` Path in Detail

`Read` in `fuse_ops.cc` is the most complex callback. The path depends on the cache mode:

```
Read(offset, len)
  │
  ├─ [Full cache]   pread(cache_fd, buf, len, node.cache_offset + offset)
  │
  ├─ [Lazy cache]   Tree::CacheUpTo(node, offset + len)
  │                 pread(cache_fd, buf, len, node.cache_offset + offset)
  │
  └─ [No cache]     Reuse or create a Reader at (index, offset)
                    Reader::Read(offset, buf)
```

**Reader reuse logic (no-cache):** Each `FileHandle` owns a `Reader::Ptr`. The reader is
reused if the requested offset is:

- within the current position (can serve from rolling buffer), or
- ahead by no more than `rolling_buffer_size` (256 KiB) beyond the current position.

If neither condition holds (large forward skip or any backward seek), the reader is
released, and `Tree::GetReader()` searches `recycled_readers_` for the best warm reader
— one positioned at or before the target entry/offset in the same archive. If no suitable
warm reader exists, a new one is constructed from scratch (O(entries) decompression cost
to reach the target entry).

**Rolling buffer:** `Reader::rolling_buffer` is a 256 KiB circular buffer. Each
`archive_read_data()` call fills it. Reads within `[offset - rolling_buffer_size,
offset_within_entry]` are served with `memcpy` from the buffer without re-decompressing.
This handles the kernel's occasional out-of-order reads efficiently.

---

## Path Lookup

Every FUSE callback that takes a `const char* path` calls `Tree::FindNode(path)`, which
resolves to a single hash-map lookup:

```cpp
nodes_by_path_.find(HashedStringView(path), hash_function, key_eq)
```

The hash is computed by `Node::ComputePathHash()` at insertion time using
`boost::hash_combine` over each character of the full path. The equality check compares
the stored `path_length` and `path_hash` first, then falls back to reconstructing the
full path string (`Node::GetPath()`) for definitive comparison.

Because `fi->fh` carries a `Node*` or `FileHandle*` directly after `open`/`opendir`,
subsequent `read`/`readdir`/`release` calls skip the hash lookup entirely and use the
pointer directly (`cfg->nullpath_ok = true` allows FUSE to pass a null path in those
cases).

---

## Reader Lifecycle and Recycling

Creating a `Reader` opens a new `archive_read` handle and seeks to entry 1. Advancing to
entry N costs O(N) decompression work for non-seekable formats (tar, cpio, etc.).

To amortise this cost, `Tree::recycled_readers_` acts as a free list. When a `Reader::Ptr`
is released, its custom deleter (`Reader::Recycler::operator()`) places the reader into
`recycled_readers_` instead of calling `delete`. Readers accumulate in the list (up to an
internal cap) so that the next request at a nearby position can take one from the list.

`Tree::GetReader()` searches the list for the _best_ reader: one that is ahead of or at
the target `(index_within_archive, offset_within_entry)` and as close to it as possible.
For archives without compression filters, only readers at exactly the right entry index
are considered (a forward skip within the same entry is cheap). For filtered archives, any
reader at or before the target is acceptable (a forward skip across entries is still
cheaper than re-opening from the start).

---

## Sparse File Support

Many archive entries contain long runs of NUL bytes (e.g. disk images, database files).
fuse-archive detects and preserves these as sparse regions:

1. **Detection during caching:** `FileDescriptor::WriteBytesAndSkipHoles()` scans each
   write for NUL runs. It punches holes using `fallocate(FALLOC_FL_PUNCH_HOLE)` on Linux
   or `lseek(SEEK_DATA/SEEK_HOLE)` on macOS. Each hole's `[from, to)` byte range is
   appended to `Node::holes`.

2. **`SEEK_DATA` / `SEEK_HOLE`:** The `Seek` FUSE callback calls `Node::SparseSeek()`,
   which binary-searches `Node::holes` to return the correct next data or hole position.
   This allows tools like `cp --sparse=always` or `rsync` to reproduce the sparse layout.

3. **Block count:** `Node::GetBlockCount()` deducts `saved_blocks` (blocks saved by holes)
   from the raw block count. `statfs` reflects the reduced disk usage.

---

## Extended Attributes

When `-o noxattrs` is not set, `Tree::ProcessEntry()` calls `archive_entry_xattr_reset()`
and iterates over each xattr key/value pair stored in the archive entry, appending them
to `Node::attributes`.

Attribute keys are interned via `GetOrCreateUnique()`, which maintains a global
`HashedStringSet` of `HashedString` objects. All nodes that share the same xattr key name
point to the same `HashedString*`. This eliminates redundant string copies for common
keys (e.g. `system.posix_acl_access`) across millions of files.

The `GetXattr` and `ListXattr` FUSE callbacks serve requests directly from the node's
`Attributes` vector without touching the archive.

---

## Password / Passphrase Handling

For encrypted archives (7z, ZIP with AES, GPG), fuse-archive reads the passphrase from
standard input at load time. `Reader::ReadPassword()` is registered as the libarchive
passphrase callback via `archive_read_set_passphrase_callback()`. It prompts on stdout,
suppresses terminal echo via `tcsetattr(TCSANOW)`, and reads one line from stdin.

The passphrase is applied to the libarchive handle for built-in format decryption (ZIP,
7z). GPG decryption is handled differently: it is delegated entirely to the external `gpg`
subprocess via `archive_read_append_filter_program()`; libarchive does not pass any
passphrase to it. The `gpg` process connects to `gpg-agent` for key material independently.

---

## Logging

fuse-archive uses a custom logger (`LOG(LEVEL)` macros in `util.h`) that writes to
`syslog`. The macro expands to a temporary `Logger` object whose destructor flushes the
line. The log level is set once at startup from the verbosity flags (`-v` / `-vv`).

The `should_print_progress` flag on `Reader` triggers periodic `LOG(INFO)` lines during
the load phase when the archive is large. A `Beat` throttler (in `util.h`) ensures
progress messages are emitted at most once per second.

---

## Platform Notes

| Area                          | Linux                                      | macOS                                                                                          |
| ----------------------------- | ------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| Cache file creation           | `memfd_create()` or `tmpfile()`            | `tmpfile()`                                                                                    |
| Sparse hole punching          | `fallocate(FALLOC_FL_PUNCH_HOLE)`          | `lseek(SEEK_DATA/SEEK_HOLE)`                                                                   |
| SEEK_DATA / SEEK_HOLE in FUSE | Supported                                  | Not supported (skipped in tests)                                                               |
| FUSE version                  | libfuse 3.x preferred                      | macFUSE (FUSE 2.x API)                                                                         |
| GPG filter subprocess         | Direct binary via `execvp` + inherited env | Shell script wrapper to work around `posix_spawnp` + `POSIX_SPAWN_CLOEXEC_DEFAULT` + null envp |
| GPG/crypto in libarchive      | nettle or openssl                          | CommonCrypto (no OpenPGP support — external `gpg` used instead)                                |

---

> [!CAUTION]
> This document was generated with the assistance of Claude Code (AI). It accurately
> reflects the source code as of the date it was written, but may become out of date
> as the codebase evolves.
