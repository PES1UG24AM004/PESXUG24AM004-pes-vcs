# PES-VCS — Version Control System from Scratch

> A local, Git-style version control system implemented in C for an OS lab. Tracks file changes, manages a staging area, stores snapshots using content-addressable storage (SHA-256), and builds a linked commit history.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Project Structure](#project-structure)
- [Implementation Phases](#implementation-phases)
  - [Phase 1: Object Storage Foundation](#phase-1-object-storage-foundation)
  - [Phase 2: Tree Objects](#phase-2-tree-objects)
  - [Phase 3: The Index (Staging Area)](#phase-3-the-index-staging-area)
  - [Phase 4: Commits and History](#phase-4-commits-and-history)
- [Phase 5 & 6: Analysis Questions](#phase-5--6-analysis-questions)
  - [Q5.1: Branching and Checkout Implementation](#q51-branching-and-checkout-implementation)
  - [Q5.2: Detecting a Dirty Working Directory](#q52-detecting-a-dirty-working-directory)
  - [Q5.3: Detached HEAD State](#q53-detached-head-state)
  - [Q6.1: Garbage Collection Algorithm](#q61-garbage-collection-algorithm)
  - [Q6.2: GC Race Conditions](#q62-gc-race-conditions)

---

## Overview

PES-VCS is a from-scratch implementation of a version control system modeled after Git's core internals. It demonstrates:

- **Content-addressable object storage** using SHA-256 hashing and sharded directories
- **Staged commits** via an `.pes/index` file tracking blob metadata
- **Recursive tree objects** that mirror directory structure
- **Linked commit history** with author metadata and `HEAD`/branch reference pointers
- **Atomic writes** to prevent data corruption on crashes

---

## Features

- `pes init` — Initialize a new repository
- `pes add <files>` — Stage files (hash, store as blob, update index)
- `pes status` — Show staged vs. unstaged changes
- `pes commit -m "<msg>"` — Create a commit from the current index
- `pes log` — Walk and display the commit history

---

## Project Structure

```
.pes/
├── HEAD                  # Points to current branch or commit hash
├── index                 # Staging area (sorted, atomic writes)
├── objects/              # Content-addressable store (sharded by first 2 hex chars)
│   └── ab/
│       └── cdef1234...  # Blob, tree, or commit object
└── refs/
    └── heads/
        └── main          # Branch pointer (stores commit hash)
```

---

## Implementation Phases

### Phase 1: Object Storage Foundation

Built the core storage engine with deduplication and atomic writes. Files are written to a `.tmp` path first, then renamed — ensuring the store is never left in a corrupt state if the process crashes mid-write.

Every object is SHA-256 hashed. The first two characters of the hex digest form a subdirectory; the remainder is the filename. This sharding keeps directory sizes manageable at scale.

```bash
make test_objects
./test_objects
find .pes/objects -type f
```

| Screenshot | Description |
|---|---|
| ![Phase 1A](final-screenshots/1A.png) | Tests passing |
| ![Phase 1B](final-screenshots/1B.png) | Sharded object directory structure |

---

### Phase 2: Tree Objects

Implemented recursive tree construction from a flat list of staged files. Files are grouped by directory prefix, and trees are built bottom-up — matching Git's actual directory tracking behavior. The resulting tree objects are stored in binary format.

```bash
make test_tree
./test_tree
xxd <tree-object-path> | head -20
```

| Screenshot | Description |
|---|---|
| ![Phase 2A](final-screenshots/2A.png) | Tests passing |
| ![Phase 2B](final-screenshots/2B.png) | Raw binary hex dump of a tree object |

---

### Phase 3: The Index (Staging Area)

Built the `.pes/index` tracking system. When a file is staged with `pes add`, it is hashed, saved as a blob object, and registered in the index with its metadata (size, modification time). The index is kept sorted for deterministic behavior and written atomically.

```bash
./pes init
echo "hello" > file1.txt
echo "world" > file2.txt
./pes add file1.txt file2.txt
./pes status
cat .pes/index
```

| Screenshot | Description |
|---|---|
| ![Phase 3A](final-screenshots/3A.png) | `add` and `status` output |
| ![Phase 3B](final-screenshots/3B.png) | Raw text content of the index file |

---

### Phase 4: Commits and History

Tied all components together. The commit process:

1. Builds a tree object from the current index
2. Links it to the parent commit hash
3. Attaches author metadata and a timestamp
4. Serializes the commit object to disk
5. Advances the `HEAD` and branch reference pointer

```bash
./pes commit -m "Initial commit"
echo "Goodbye" > bye.txt
./pes add bye.txt
./pes commit -m "Add farewell"
./pes log
find .pes -type f | sort
cat .pes/refs/heads/main
cat .pes/HEAD
make test-integration
```

| Screenshot | Description |
|---|---|
| ![Phase 4A](final-screenshots/4A.png) | Commit log output |
| ![Phase 4B](final-screenshots/4B.png) | Object store growth after two commits |
| ![Phase 4C](final-screenshots/4C.png) | Reference chain (`HEAD` → branch → commit hash) |
| ![Final](final-screenshots/Final.png) | Integration test passing |

---

## Phase 5 & 6: Analysis Questions

### Q5.1: Branching and Checkout Implementation

To implement a `pes checkout <branch>` command, the system needs to read the commit hash from the target branch file and update the `.pes/HEAD` file to point to it. The complex part is updating the working directory safely: the system must recursively traverse the target commit's tree and mirror those files to the working directory while aggressively ensuring it doesn't accidentally overwrite any uncommitted, unsaved work the user currently has open.

### Q5.2: Detecting a "Dirty" Working Directory

To figure out if a checkout will cause a conflict without rehashing everything, we do a quick three-way check. First, we compare the current branch's tree with the target branch's tree to see if the file actually differs. Then, we look at the `.pes/index` file; if a file's metadata in the index differs from the current branch's baseline, it means the user has modified or staged it. If the file is modified *and* it differs between the two branches, we have a dirty conflict and must refuse the checkout to protect the user's work.

### Q5.3: Detached HEAD State

If you commit while in a detached HEAD state, the commit object successfully saves to the object store, and `HEAD` updates to point directly at that new hash. However, because no branch pointer (like `main`) is updated, the moment you switch to another branch, those new commits become "orphaned" and invisible to standard logs. To get them back, a user has to dig up the exact hash of the orphaned commit and manually create a new branch pointer directed right at it.

### Q6.1: Garbage Collection Algorithm

To clean up a massive repository, a mark-and-sweep algorithm using a Hash Set is the best approach. In the **mark phase**, we start at every branch reference in `.pes/refs/heads/`, walk backwards through all parent commits, and tag every commit, tree, and blob hash we encounter as "reachable" by adding them to the Hash Set (which gives us O(1) lookups). In the **sweep phase**, we loop through the actual `.pes/objects/` folder and delete any file whose hash isn't in our set. For a repo with 100k commits, we'd have to traverse millions of blobs and subtrees during the mark phase, but the sweep itself would be highly efficient.

### Q6.2: GC Race Conditions

Running GC in the background while someone is committing is a recipe for disaster due to race conditions. For example, if a user runs `pes add file.txt`, a blob is generated; if GC sweeps right at that millisecond before `pes commit` runs, it will delete the new blob because it isn't linked to a commit tree yet, completely corrupting the upcoming commit. Git solves this by enforcing a grace period — usually two weeks — meaning the GC will only delete unreachable objects if their file timestamps prove they are genuinely old and abandoned, ignoring freshly minted files entirely.
