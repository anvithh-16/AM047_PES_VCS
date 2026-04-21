# PES-VCS — Version Control System from Scratch

> A Git-inspired local version control system implementing content-addressable storage, staging area, tree objects, and commit history.

---

## 1. Team Information

| Name | SRN |
|------|-----|
| Anvith Vegi  | PES1UG24AM047 |

---

## 2. Build and Run Instructions

### Prerequisites

```bash
sudo apt update
sudo apt install -y gcc build-essential libssl-dev
```

### Build Project

```bash
make clean
make
```

### Run Tests

#### Phase 1 — Object Storage

```bash
make test_objects
./test_objects
```

#### Phase 2 — Tree Objects

```bash
make test_tree
./test_tree
```

### Run PES Commands

```bash
./pes init
echo "hello" > file1.txt
echo "world" > file2.txt
./pes add file1.txt file2.txt
./pes status
./pes commit -m "Initial commit"
./pes log
```

---

## 3. Demo Screenshots

### Phase 1 — Object Storage Foundation

| ID | Description | Screenshot |
|----|-------------|------------|
| 1A | `./test_objects` output — all tests passing | <img width="1390" height="508" alt="image" src="https://github.com/user-attachments/assets/711ea732-b225-4816-ae3d-10e2f9b2015b" /> |
| 1B | `find .pes/objects -type f` — sharded directory structure | <img width="1384" height="402" alt="image" src="https://github.com/user-attachments/assets/8a00b65d-790b-4a52-8c4a-60558f7d43b6" /> |

### Phase 2 — Tree Objects

| ID | Description | Screenshot |
|----|-------------|------------|
| 2A | `./test_tree` output — all tests passing | <img width="1834" height="276" alt="image" src="https://github.com/user-attachments/assets/f592f2c3-3298-4a00-be71-7125edbdc07f" /> |
| 2B | `xxd` dump of a raw tree object (first 20 lines) | <img width="1834" height="220" alt="image" src="https://github.com/user-attachments/assets/5814ab53-6d28-48de-9370-2fce3742ba1b" /> |

### Phase 3 — Staging Area (Index)

| ID | Description | Screenshot |
|----|-------------|------------|
| 3A | `pes init` → `pes add` → `pes status` sequence | <img width="750" height="722" alt="Screenshot 2026-04-21 at 12 30 37 PM" src="https://github.com/user-attachments/assets/8cb7c5de-fed1-47e2-8969-f0c06d6ab0f4" /> |
| 3B | `cat .pes/index` — human-readable index file | <img width="805" height="75" alt="image" src="https://github.com/user-attachments/assets/9d259074-6b99-4be2-8c54-f8b37f4ec632" /> |

### Phase 4 — Commits and History

| ID | Description | Screenshot |
|----|-------------|------------|
| 4A | `pes log` — three commits with hashes, authors, timestamps | <img width="939" height="730" alt="image" src="https://github.com/user-attachments/assets/dd5089df-9e7b-4761-a6d9-8c60b71db451" /> |
| 4B | `find .pes -type f \| sort` — object store growth | <img width="714" height="231" alt="image" src="https://github.com/user-attachments/assets/363b21de-157d-4d5a-a8a3-99715211689a" /> |
| 4C | `cat .pes/refs/heads/main` and `cat .pes/HEAD` | <img width="663" height="70" alt="image" src="https://github.com/user-attachments/assets/b47e55d4-1851-4557-8ecd-cef6ce8d96e5" /> |

### Final Integration Test

<img width="978" height="673" alt="image" src="https://github.com/user-attachments/assets/643181e1-9390-41ec-bef8-c86b60a29d22" />

<img width="721" height="665" alt="image" src="https://github.com/user-attachments/assets/26fd7340-8f5e-4a61-9fb4-3418aa6fabce" />


---

## 4. Engineering Analysis

### 4.1 Object Storage Design

Content-addressable storage using SHA-256 ensures three key properties:

- **Deduplication** — identical file contents are stored exactly once
- **Integrity** — any corruption is detectable by re-hashing
- **Immutability** — changing content produces a different hash, a different object

Objects are sharded under `.pes/objects/XX/YYY...` using the first two hex characters to avoid oversized directories.

### 4.2 Tree Structure

Tree objects represent directory snapshots. Each entry records a file mode, name, and the SHA-256 hash of the referenced blob or subtree. Nested paths (e.g., `src/main.c`) are handled recursively — a `src` subtree object is created and referenced from the root tree.

### 4.3 Index (Staging Area)

The index (`.pes/index`) is a plain text file tracking staged files with the format:

```
<mode-octal>  <64-char-hex-hash>  <mtime>  <size>  <path>
```

Writes use the atomic temp-file-then-rename pattern so a crash mid-write never leaves the index in a corrupt state.

### 4.4 Commit System

Each commit stores a pointer to the root tree, an optional parent commit hash, author metadata, a Unix timestamp, and the commit message. The chain of parent pointers forms the complete history linked list.

---

## 5. Analysis Questions

### Q5.1 — Branching and Checkout

To implement `pes checkout <branch>`:

1. Read the target branch file at `.pes/refs/heads/<branch>` to get the commit hash.
2. Parse that commit to get its root tree.
3. Walk the tree recursively and update every file in the working directory to match.
4. Rewrite `.pes/index` to match the new tree's contents.
5. Update `.pes/HEAD` to `ref: refs/heads/<branch>`.

The complexity comes from handling **conflicts**: if the user has uncommitted changes to a file that also differs between branches, a naive checkout would silently overwrite their work. The operation must detect and refuse this case.

### Q5.2 — Dirty Working Directory Detection

For each tracked file (every entry in the index):

1. Read the stored blob hash from the index.
2. Read the corresponding blob hash from the target branch's tree.
3. If they differ, the file changes between branches — now check the working directory.
4. Re-hash the working copy and compare it to the **current** index hash. If they differ, the file has been locally modified.
5. If the file is both locally modified **and** differs between branches, refuse checkout to avoid data loss.

This requires no extra data beyond the index and object store — no diff tool needed.

### Q5.3 — Detached HEAD

In detached HEAD state, `HEAD` contains a raw commit hash instead of `ref: refs/heads/<branch>`. New commits are created and chained correctly, but **no branch pointer is updated** — so once you switch away, those commits become unreachable and invisible to `pes log`.

Recovery options:
- Run `git reflog` (or a PES equivalent) if the hashes were recorded.
- Create a new branch pointing directly at the detached commit: `git branch recover-work <hash>`.
- If the hash is lost, a GC sweep would eventually delete those objects.

### Q6.1 — Garbage Collection

**Algorithm — mark and sweep:**

1. **Mark phase:** Start from every branch tip (all files in `.pes/refs/heads/`). For each commit, recursively follow parent pointers and parse each tree, adding every reachable commit, tree, and blob hash to a `HashSet`.
2. **Sweep phase:** Walk every file in `.pes/objects/`. For any object whose hash is not in the reachable set, delete it.

The right data structure is a **hash set** (e.g., a C `uthash` table or a bitset indexed by truncated hash) for O(1) membership checks during the sweep.

**Estimate for 100,000 commits, 50 branches:** assuming ~5 tree objects and ~10 blobs per commit, you'd visit roughly 100,000 + 500,000 + 1,000,000 ≈ **1.6 million objects** in the mark phase, then scan all files in the sweep phase.

### Q6.2 — GC Race Condition

Consider this interleaving:

1. A commit operation computes a blob hash and writes the blob to `.pes/objects/`.
2. GC runs its mark phase — it traverses all reachable commits. The new blob is not yet referenced by any commit or tree, so it is **not marked**.
3. GC's sweep phase deletes the "unreachable" blob.
4. The commit operation now tries to create a tree and commit pointing to the deleted blob. The object store is now corrupt.

**How Git avoids this:** Git's GC (`git gc`) uses a **grace period** — it only deletes objects that have been unreachable for longer than a configurable threshold (default: 2 weeks). A newly written object is always younger than the threshold, so it is safe. Git also writes a lock file during GC to prevent concurrent `git gc` runs, and object writes themselves are atomic (temp-file+rename), so a partial write is never visible.

---

## 6. Design Decisions and Tradeoffs

| Component | Design Choice | Tradeoff | Justification |
|-----------|--------------|----------|---------------|
| Object storage | SHA-256 content addressing | Minor hashing overhead per write | Strong integrity guarantees; automatic deduplication |
| Index format | Human-readable text file | Slower than binary format | Easy to debug with `cat`; matches lab scope |
| Tree building | Recursive subtree construction | More complex code | Correctly handles arbitrary directory nesting |
| All writes | Atomic temp-file + rename | Extra I/O (one extra file per write) | Crash-safe; no half-written state ever visible |

---

## 7. Conclusion

PES-VCS is a minimal but architecturally faithful reimplementation of Git's core engine. It demonstrates how a few Unix primitives — SHA-256 hashing, atomic file rename, and linked on-disk structures — combine to produce a reliable, deduplicated, and tamper-evident version control system. Every design choice maps directly to an OS concept covered in the lab: content-addressable storage, filesystem atomicity, directory sharding, and pointer-based history traversal.
