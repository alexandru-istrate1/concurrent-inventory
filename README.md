# Concurrent File & Process Inventory System

A set of multi-process systems tools written in C on top of POSIX APIs, built for the Operating Systems course at the Faculty of Computer Science, Iasi. The tools scan the file system and running processes, store snapshots in custom binary databases, and compare them over time.

## Components

### 1. Snapshot databases with record-level locking
- **`fileops_indexer`** — recursively indexes the file system into `index.db` (path, type, size, mtime, inode, djb2 content hash, symlink targets)
- **`proc_snapshot`** — captures running processes from `/proc` into `proc.db` (PID, PPID, state, RSS, CPU time, command line)
- **`db_diff`** — compares two snapshots and reports added, removed and modified entries

Multiple instances can contribute to the same snapshot concurrently. Writers use a **two-level locking scheme** (`fcntl` locks on the header, then on the individual record region), so concurrent upserts never corrupt the database. A snapshot is automatically **sealed** when the last writer exits.

### 2. Manager/worker inventory over shared memory
- **`fileops_manager`** distributes directories to a pool of **`fileops_worker`** processes through a memory-mapped region (`mmap` + `MAP_SHARED`)
- A shared **ring-buffer job queue** plus a **dedicated result channel per worker**
- **POSIX semaphores** for mutual exclusion and **backpressure**: producers block when a queue is full, consumers block when it is empty
- **Deterministic termination**: the scan ends when the queue is empty *and* no job is in progress, after which workers are woken up and exit cleanly
- Per-worker statistics (files, bytes, user/system CPU time via `getrusage()`, wall time)
- The final database is written **atomically**: built in a `.tmp` file, marked complete, then swapped in with `rename()`, so a crash never leaves a truncated database

### 3. Control plane and graceful shutdown (HW5)
Extends the manager/worker system with three separated planes:

- **Data plane** — the shared-memory job queue, result channels and stats from part 2, unchanged
- **Control plane** — each worker reports to the manager through an **anonymous pipe** (fd passed via `--control-fd`, unused ends closed after `fork()`), using a line-based protocol (`JOB_DONE`, `HEARTBEAT`, `ERROR`, `WORKER_EXITING`). Every message is written in a single `write()` of at most `PIPE_BUF` bytes, so messages from different workers never interleave.
- **Signal plane**:
  - `SIGUSR1` prints a live status line (queued/active jobs, files, bytes, workers alive)
  - `SIGINT`/`SIGTERM` trigger a **graceful shutdown**: set the terminate flag, wake blocked workers, send `SIGTERM`, wait for a configurable timeout, escalate to `SIGKILL` if needed, then reap all children
  - `SIGCHLD` reaps workers with `waitpid(..., WNOHANG)` in a loop, so no zombies remain

Signal handlers are **async-signal-safe**: they only set `volatile sig_atomic_t` flags, and all work happens in the main loop.

An interrupted run still produces a structurally valid database marked as **partially complete**, while a finished scan is marked **fully complete**, and both pass `--verify`.

## Binary database format

Each database is a fixed-size header followed by fixed-size records, so record *N* can be accessed directly at `sizeof(header) + N * sizeof(record)`. Files are validated on open (magic number, format version, exact size check) and invalid files are rejected without being modified.

Full specifications are in [`doc/`](doc/):
- [`FORMAT_DB.md`](doc/FORMAT_DB.md) — snapshot database layout and update strategy
- [`T4_DB_FORMAT.md`](doc/T4_DB_FORMAT.md) — inventory database layout
- [`MMAP_PROTOCOL.md`](doc/MMAP_PROTOCOL.md) — shared-memory IPC protocol

## Project structure

```
include/   shared data formats (db_format.h, ipc_format.h, db_lock.h)
src/       tool entry points and locking implementation
tests/     shell test suites (concurrency, recursive build, inventory)
tools/     helper scripts
doc/       format and protocol specifications
HW5/       control plane and signal handling extension
```

## Tech
C, POSIX (`mmap`, semaphores, `fcntl` locks, pipes, signals, `fork`), Bash, Linux


## Authors 
- Istrate Alexandru
- Mindirigiu Cristian
