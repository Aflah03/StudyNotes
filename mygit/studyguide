# ugit — DIY Git in Python: Interview Study Guide

> A build-your-own-Git project. You reimplemented Git's core plumbing in ~600 lines of Python
> across `ugit/data.py`, `ugit/base.py`, `ugit/diff.py`, `ugit/remote.py`, and `ugit/cli.py`.
> This guide walks the commit progression, explains what happens **under the hood**, and ends
> with likely interviewer questions + answers.

---

## 0. The 30-second pitch (memorize this)

> "I built a small clone of Git from scratch in Python called **ugit**. It implements Git's
> real internal model — a **content-addressable object store** (blobs, trees, commits addressed
> by SHA-1 hash), **references** (HEAD, branches, tags) as pointers into that store, a **staging
> area / index**, and higher-level commands like `commit`, `log`, `branch`, `checkout`, `diff`,
> `merge` (three-way + fast-forward), and even `fetch`/`push` between local repos. It taught me
> what Git actually does under the hood instead of just memorizing commands."

### The architecture in one picture

```
        cli.py                 base.py                    data.py
   (argument parsing,     (Git "porcelain":          (Git "plumbing":
    user-facing commands)  commit, log, checkout,     object store + refs
          |                merge, branches...)        on disk)
          |                       |                          |
   parse_args() ---> base.commit() ---> data.hash_object() --+--> .ugit/objects/<sha1>
                          |          \-> data.update_ref() ---+--> .ugit/refs/... , .ugit/HEAD
                          |
                     diff.py (compare_trees, diff via `diff`, merge via `diff3`)
```

Git's own terms map cleanly:
- **plumbing** = low-level internals → `data.py`
- **porcelain** = user-facing commands → `base.py` + `cli.py`

---

## 1. The core idea: Git is a content-addressable key-value store

The single most important sentence for the interview:

> **Git stores everything as objects, and the *key* of each object is the SHA-1 hash of its own contents.**

This means:
- Identical content → identical hash → stored once (automatic deduplication).
- Content can never be silently corrupted: if a byte changes, the hash changes, so the address changes.
- You never "name" an object; you compute its address from what's inside it. This is **content-addressable storage**.

There are 3 object **types**, all stored the same way:

| Type     | Represents        | Contents                                         |
|----------|-------------------|--------------------------------------------------|
| `blob`   | a file's bytes    | the raw file content                             |
| `tree`   | a directory       | a list of `(type, oid, name)` entries            |
| `commit` | a snapshot in time| a tree oid + parent oid(s) + message             |

"oid" = **object id** = the SHA-1 hash.

---

## 2. Commit-by-commit progression

I've grouped the ~60 commits into 10 logical phases. For each phase: **what was added, the code, and what happens under the hood.**

---

### PHASE 1 — Project skeleton + the object store (plumbing)

**Commits:** `ugit: DIY Git in Python` → `cli: Add argument parser` → `init` → `hash-object` → `cat-file` → `data: Add types to objects`

#### `init` — create the repo
```python
# data.py
GIT_DIR = None            # set to '.ugit' at runtime
def init ():
    os.makedirs (GIT_DIR)                 # .ugit/
    os.makedirs (f'{GIT_DIR}/objects')    # .ugit/objects/  <- the object database
```
Under the hood: real Git creates `.git/`; ugit creates `.ugit/`. The `objects/` folder is the whole database.

#### `hash-object` — write a file into the object store
```python
# data.py
def hash_object (data, type_='blob'):
    obj = type_.encode () + b'\x00' + data          # prepend "blob\0"
    oid = hashlib.sha1 (obj).hexdigest ()           # address = SHA-1 of (type + content)
    with open (f'{GIT_DIR}/objects/{oid}', 'wb') as out:
        out.write (obj)
    return oid
```

**This is the heart of Git.** Walk through it:
1. Prepend a type tag and a null byte: `blob\x00<file bytes>`.
2. SHA-1 the whole thing → a 40-char hex string, e.g. `a1b2c3...`.
3. Save the bytes to `.ugit/objects/a1b2c3...`. The **filename is the hash**.

```
   file "hello.txt" = "hi\n"
            |
   "blob\0hi\n"  --SHA1-->  af5626b...
            |
   .ugit/objects/af5626b...   (contents: b"blob\0hi\n")
```

> **Interview note:** Real Git does two extra things: it prepends a *length* header
> (`blob 3\0hi\n`) and **zlib-compresses** the object before writing. It also shards the
> directory (`objects/af/5626b...`) so one folder isn't full of millions of files. ugit
> skips compression and sharding for simplicity. Modern Git can also use SHA-256.

#### `cat-file` — read an object back
```python
def get_object (oid, expected='blob'):
    with open (f'{GIT_DIR}/objects/{oid}', 'rb') as f:
        obj = f.read ()
    type_, _, content = obj.partition (b'\x00')     # split off the "blob\0" header
    type_ = type_.decode ()
    if expected is not None:
        assert type_ == expected, f'Expected {expected}, got {type_}'
    return content
```
`hash-object` and `cat-file` are inverses. Adding the **type** (`data: Add types to objects`)
lets `get_object` verify it read the kind of thing the caller expected — a blob vs a tree vs a commit.

---

### PHASE 2 — Trees: snapshotting a whole directory

**Commits:** `write-tree: List files` → `Ignore .ugit files` → `Hash the files` → `Write tree objects` → `read-tree: Extract tree` → `Delete existing stuff before reading`

A **tree** is Git's way of representing a directory. A tree object is just text, one line per entry:

```
blob 5b1f1e...  README.md
blob af5626...  hello.txt
tree 9c2a4d...  src
```
Each line is `<type> <oid> <name>`. A `tree` entry points at a *sub-tree* (a subdirectory) — that's the recursion, and it's why Git can represent an entire nested directory as one root hash.

#### write-tree (recursive)
```python
def write_tree_recursive (tree_dict):
    entries = []
    for name, value in tree_dict.items ():
        if type (value) is dict:
            type_ = 'tree'
            oid = write_tree_recursive (value)   # recurse into subdirectory
        else:
            type_ = 'blob'
            oid = value                          # already-hashed file
        entries.append ((name, oid, type_))
    tree = ''.join (f'{type_} {oid} {name}\n' for name, oid, type_ in sorted (entries))
    return data.hash_object (tree.encode (), 'tree')   # the tree is itself an object
```
Under the hood — snapshotting `.`:
```
   .
   ├── hello.txt   --hash-->  blob af5626
   └── src/
       └── a.py    --hash-->  blob 11aa22

   tree(src)  = "blob 11aa22 a.py\n"                        -> tree 9c2a4d
   tree(root) = "blob af5626 hello.txt\ntree 9c2a4d src\n"  -> tree 4f8e01
```
The single hash `4f8e01` now uniquely identifies the **entire directory tree**. Two directories
with identical content produce the same tree hash — dedup again.

#### read-tree — the reverse
`get_tree` walks the tree object recursively back into a `{path: oid}` map, then
`_checkout_index`/`_empty_current_directory` **deletes the working directory contents and rewrites
them** from the objects. This is why `read-tree`/`checkout` can restore an old snapshot exactly.

> **Interview note:** `is_ignored()` prevents ugit from snapshotting its own `.ugit/` folder —
> the same reason Git never versions `.git/`.

---

### PHASE 3 — Commits + HEAD

**Commits:** `commit: Create commit` → `Record hash of last commit to HEAD` → `commit: set parent to HEAD`

A **commit object** wraps a tree with metadata:
```python
def commit (message):
    commit  = f'tree {write_tree ()}\n'          # snapshot pointer
    HEAD = data.get_ref ('HEAD').value
    if HEAD:
        commit += f'parent {HEAD}\n'             # link to previous commit
    commit += '\n'
    commit += f'{message}\n'
    oid = data.hash_object (commit.encode (), 'commit')
    data.update_ref ('HEAD', data.RefValue (symbolic=False, value=oid))  # move HEAD forward
    return oid
```
A commit object looks like:
```
tree 4f8e01...
parent 8823bc...

Fix the login bug
```
The `parent` line is what makes history a **linked list** (actually a DAG once merges appear).
**HEAD** is a pointer to "where I am now." Every commit: (1) snapshot the tree, (2) point the new
commit's `parent` at the old HEAD, (3) move HEAD to the new commit.

```
   HEAD
    |
    v
   C3 --parent--> C2 --parent--> C1 --parent--> (none)
   |tree           |tree          |tree
   v               v              v
  snapshot3      snapshot2      snapshot1
```

> Real Git commits also store `author`, `committer`, and timestamps. ugit keeps just tree +
> parent + message to stay minimal.

---

### PHASE 4 — Reading history: `log`, `checkout`, and `get_commit`

**Commits:** `log: Implement` → `log: Add oid parameter` → `checkout: Read tree and move HEAD`

`get_commit` parses a commit object back into a `namedtuple(tree, parents, message)`:
```python
for line in itertools.takewhile (operator.truth, lines):   # read header until blank line
    key, value = line.split (' ', 1)
    if key == 'tree':   tree = value
    elif key == 'parent': parents.append (value)
message = '\n'.join (lines)                                 # the rest is the message
```
`log` starts at HEAD and follows `parent` links backward, printing each commit — exactly `git log`.

`checkout` = "go to this commit": read its tree into the working directory **and** move HEAD.

---

### PHASE 5 — References: tags, then a general ref system

**Commits:** `tag: Implement` → `tag: Generalize HEAD to refs` → `Create the tag ref` → `Resolve name to oid in argparse` → `Try different directories when searching for a ref` → `cli: pass HEAD by default`

A **ref** is just a file under `.ugit/refs/` containing an oid. This is the big conceptual unlock:

> **HEAD, branches, and tags are all the same thing — a small file whose content is a 40-char hash (or a pointer to another ref).**

```
   .ugit/
     HEAD                 -> "ref: refs/heads/master"   (symbolic: points to a branch)
     refs/
       heads/
         master           -> "a1b2c3..."   (a branch: a moving pointer)
         feature          -> "d4e5f6..."
       tags/
         v1.0             -> "a1b2c3..."   (a tag: usually a fixed pointer)
```

`get_oid` is the **name resolver** — it turns a human name into an oid by trying, in order:
```python
refs_to_try = [f'{name}', f'refs/{name}', f'refs/tags/{name}', f'refs/heads/{name}']
```
...and if none match, checks whether `name` is already a 40-char hex SHA. That's how `ugit log`,
`ugit checkout v1.0`, `ugit checkout master`, and `ugit checkout <sha>` all work with one function.
Wiring `type=base.get_oid` into argparse means names are resolved to oids automatically at the CLI layer.

---

### PHASE 6 — Visualizing the DAG: the `k` command

**Commits:** `k: Print refs` → `Iterate commits and parents` → `Render graph` → `log: Use iter_commits_and_parents`

`k` (named after `gitk`) emits **Graphviz DOT** and pipes it to `dot` to draw the commit graph:
```python
for refname, ref in data.iter_refs (deref=False):
    dot += f'"{refname}" -> "{ref.value}"\n'        # ref -> commit it points to
for oid in base.iter_commits_and_parents (oids):
    for parent in commit.parents:
        dot += f'"{oid}" -> "{parent}"\n'           # commit -> its parent(s)
```
`iter_commits_and_parents` is a **breadth-first walk of the commit DAG** using a `deque` and a
`visited` set (so shared ancestors aren't visited twice). It's reused by `log`, `k`, and merge logic.

---

### PHASE 7 — Symbolic refs, RefValue, and real branches

**Commits:** `branch: Create new branch` → `data: Implement symbolic refs` → `Create RefValue container` → `Dereference refs when reading/writing` → `Don't always dereference (for k)` → `Write symbolic refs` → `checkout: Switch branches` → `init: Set HEAD to master` → `status: Print current branch` → `branch: Show all branches`

This phase makes branches behave like real Git. Two kinds of ref, captured by one container:
```python
RefValue = namedtuple ('RefValue', ['symbolic', 'value'])
```
- **Direct ref:** value is an oid. (`refs/heads/master → a1b2c3`)
- **Symbolic ref:** value points at *another ref*. Stored on disk as `ref: refs/heads/master`.
  `HEAD` is normally symbolic — it points at the current branch, not directly at a commit.

`_get_ref_internal` follows symbolic refs recursively when `deref=True`:
```python
symbolic = value.startswith ('ref:')
if symbolic:
    value = value.split (':', 1)[1].strip ()
    if deref:
        return _get_ref_internal (value, deref=True)   # HEAD -> refs/heads/master -> oid
```

**Why this matters — committing on a branch.** Because HEAD is `ref: refs/heads/master`, calling
`update_ref('HEAD', oid)` with `deref=True` *writes through* HEAD and updates `refs/heads/master`
instead. So the branch pointer advances automatically while HEAD keeps pointing at the branch:

```
   BEFORE commit          AFTER commit
   HEAD ─► master ─► C1    HEAD ─► master ─► C2 ─► C1
```

**Detached HEAD.** `checkout` decides:
```python
if is_branch (name):
    HEAD = RefValue (symbolic=True,  value=f'refs/heads/{name}')  # attached to branch
else:
    HEAD = RefValue (symbolic=False, value=oid)                   # detached at a commit
data.update_ref ('HEAD', HEAD, deref=False)   # note deref=False: write HEAD itself
```
Checking out a branch → HEAD is symbolic (commits move the branch). Checking out a raw commit/tag
→ HEAD holds the oid directly = **detached HEAD** (commits wouldn't move any branch). `deref=False`
is essential here: we want to overwrite HEAD *itself*, not the branch it currently points to.

`get_branch_name` reads HEAD with `deref=False` and returns the branch only if HEAD is symbolic —
that's how `status` prints "On branch master" vs "HEAD detached at a1b2c3".

---

### PHASE 8 — Diffing and `show`, `reset`

**Commits:** `log: Show refs at each commit` → `reset: Move HEAD` → `show: Print message / changed files / diff` → `diff: Compare working tree to a commit` → `status: Show changed files`

The diff engine (`diff.py`) is built on one primitive, `compare_trees`, which aligns any number of
`{path: oid}` trees by path:
```python
def compare_trees (*trees):
    entries = defaultdict (lambda: [None] * len (trees))
    for i, tree in enumerate (trees):
        for path, oid in tree.items ():
            entries[path][i] = oid
    for path, oids in entries.items ():
        yield (path, *oids)     # (path, oid_in_tree0, oid_in_tree1, ...)
```
From this everything follows:
- **`iter_changed_files`** — if a path's oid differs between two trees, classify it:
  `new file` (missing on the left), `deleted` (missing on the right), else `modified`.
- **`diff_trees` / `diff_blobs`** — for each changed path, write both blob versions to temp files
  and shell out to the Unix `diff -u` to produce a real unified diff.
- **`show`** diffs a commit against its **first parent** → "what changed in this commit."
- **`reset`** just moves HEAD to an arbitrary commit (this is `git reset --soft`; ugit doesn't
  touch the working tree).

```
   tree A ─┐
           ├─► compare_trees ─► per-path (oid_A, oid_B) ─► changed? ─► diff / status
   tree B ─┘
```

---

### PHASE 9 — Merging: merge-base, three-way, fast-forward

**Commits:** `merge: Create command` → `Merge in working directory` → `Support multiple parents` → `data: Delete refs` → `Record both parents` → `Iter over MERGE_HEAD` → `merge-base: common ancestor` → `Three-way merge` → `Fast-forward merge`

**merge-base** = the most recent common ancestor of two commits:
```python
def get_merge_base (oid1, oid2):
    parents1 = set (iter_commits_and_parents ({oid1}))     # all ancestors of oid1
    for oid in iter_commits_and_parents ({oid2}):          # walk oid2's ancestors
        if oid in parents1:
            return oid                                     # first shared one = merge base
```

**Fast-forward:** if the merge base *is* HEAD, then the other branch is strictly ahead — nothing to
merge, just slide the pointer forward:
```python
if merge_base == HEAD:
    read_tree (c_other.tree, update_working=True)
    data.update_ref ('HEAD', RefValue (symbolic=False, value=other))
    print ('Fast-forward merge, no need to commit')
    return
```
```
   FAST-FORWARD:   master ─► C1 ─► C2      (HEAD==C1, other==C2)  ⇒  master ─► C2
```

**Three-way merge** when both branches diverged. It uses **base / HEAD / other** and the Unix
`diff3 -m` tool per file:
```python
def merge_trees (t_base, t_HEAD, t_other):
    tree = {}
    for path, o_base, o_HEAD, o_other in compare_trees (t_base, t_HEAD, t_other):
        tree[path] = data.hash_object (merge_blobs (o_base, o_HEAD, o_other))
    return tree
```
`merge_blobs` runs `diff3 -m HEAD BASE MERGE_HEAD`, which emits the file with `<<<<<<<` conflict
markers when both sides changed the same region — exactly Git's conflict markers.

```
   THREE-WAY:            C_base (common ancestor)
                        /      \
                   C_HEAD      C_other
                        \      /
                     merge commit  (has TWO parents)
```
The merge writes `MERGE_HEAD`, merges into the working tree, and asks you to commit. The next
`commit` sees `MERGE_HEAD` exists, records **two `parent` lines**, then deletes `MERGE_HEAD`. That's
why `iter_commits_and_parents` handles multiple parents (first parent first, others later) and why
`iter_refs` includes `MERGE_HEAD`.

---

### PHASE 10 — Remotes: `fetch` and `push`

**Commits:** `data: Allow git directory change` → `fetch: ...` (4 commits) → `push: ...` (3 commits)

Remotes in ugit are just **other directories on disk** (no network). The key helper is a context
manager that temporarily repoints `GIT_DIR` at the remote:
```python
@contextmanager
def change_git_dir (new_dir):
    global GIT_DIR
    old_dir = GIT_DIR
    GIT_DIR = f'{new_dir}/.ugit'
    yield
    GIT_DIR = old_dir
```

**fetch:** read the remote's refs, then copy every object reachable from those refs that we don't
already have, then record them under `refs/remote/...`:
```python
for oid in base.iter_objects_in_commits (refs.values ()):
    data.fetch_object_if_missing (oid, remote_path)     # copy only what's missing
```
`iter_objects_in_commits` walks commits → their trees → sub-trees → blobs, yielding every object id
in history. This is the "reachability" traversal — the same idea Git uses to decide what to transfer.

**push:** the mirror image, with two safety points:
```python
# Don't allow force push: local must be a descendant of what's on the remote
assert not remote_ref or base.is_ancestor_of (local_ref, remote_ref)

# Send only objects the remote is missing (set difference over reachable objects)
objects_to_push = local_objects - remote_objects
```
```
   local objects  {A,B,C,D}
   remote objects {A,B}          ⇒  push only {C,D}
```
The force-push guard (`is_ancestor_of`) is exactly why real Git rejects a non-fast-forward push with
"Updates were rejected because the remote contains work that you do not have."

---

### PHASE 11 — The staging area (the index) — *the most recent work*

**Commits:** `add: Record added files in the index` → `add: Allow adding a directory` → `write-tree: Write from the index` → `read-tree: Read into index` → `status: Show staged and non-staged` → `diff: Add --cached and take index into account`

Before this phase, ugit committed straight from the working directory. These commits introduce the
**index** (a.k.a. staging area) — the thing that makes `git add` meaningful.

The index is a single JSON file `.ugit/index` mapping `{path: oid}`, accessed via a context manager
that loads on enter and saves on exit:
```python
@contextmanager
def get_index ():
    index = {}
    if os.path.isfile (f'{GIT_DIR}/index'):
        with open (f'{GIT_DIR}/index') as f:
            index = json.load (f)
    yield index                                  # caller mutates the dict
    with open (f'{GIT_DIR}/index', 'w') as f:
        json.dump (index, f)                     # persisted automatically
```

`add` hashes files into the object store and records them in the index. `write-tree` now builds the
tree **from the index**, not the working directory — so only staged content is committed. It first
"unflattens" the flat `{path:oid}` index into nested dicts, then recurses to build tree objects.

**The three trees.** Once the index exists, Git (and ugit) constantly compares three snapshots:

```
   ┌─────────────┐   git add    ┌───────────┐   git commit   ┌──────────┐
   │ Working Tree│ ───────────► │   Index   │ ─────────────► │  HEAD    │
   │ (your files)│              │ (staged)  │                │ (commit) │
   └─────────────┘              └───────────┘                └──────────┘
```

`status` now shows two sections by diffing adjacent pairs:
```python
# "Changes to be committed"      = HEAD  vs index
diff.iter_changed_files (base.get_tree (HEAD_tree), base.get_index_tree ())
# "Changes not staged for commit"= index vs working tree
diff.iter_changed_files (base.get_index_tree (), base.get_working_tree ())
```
And `diff --cached` shows HEAD-vs-index (what a commit *would* record) while plain `diff` shows
index-vs-working-tree (what you haven't staged yet). This is precisely how real `git status` and
`git diff` / `git diff --cached` behave.

---

## 3. The data model, all together

```
                        REFS (pointers, mutable)
   HEAD ──► refs/heads/master ──┐
                                 ▼
   ┌──────────────────────  commit C2  ──────────────────────┐
   │ tree:   T_root                                           │
   │ parent: C1                                               │
   │ message:"..."                                            │
   └───────────────┬──────────────────────────────────────────┘
                   ▼
              tree T_root
              ├─ blob  hello.txt
              └─ tree  src/
                    └─ blob a.py

   OBJECTS (immutable, content-addressed by SHA-1): blobs + trees + commits
   INDEX (staging area): flat {path: oid}, the "next" tree being built
```
- **Objects are immutable and permanent** (addressed by content).
- **Refs are mutable pointers** (branches move; HEAD moves).
- **A branch is a pointer; a commit is a snapshot; a tree is a directory; a blob is a file.**

---

## 4. Likely interviewer questions (with answers)

### About *your* project
1. **"Walk me through what happens when you run `ugit commit`."**
   → write-tree from the index (creating tree objects), build a commit object pointing at that tree
   and at the current HEAD as parent, hash-store it, then move HEAD (which writes through to the
   current branch because HEAD is a symbolic ref).

2. **"Why is it called content-addressable? What's the benefit?"**
   → The address is the SHA-1 of the content. Benefits: automatic dedup (same content stored once),
   integrity (any change changes the hash), and cheap "is this the same?" checks (compare hashes).

3. **"How does ugit know what changed for `status`?"**
   → It builds three `{path: oid}` maps — HEAD tree, index, working tree — and compares them
   pairwise. Different oid for a path = modified; missing on one side = added/deleted.

4. **"How does branching work with only files?"**
   → A branch is a one-line file under `refs/heads/` holding a commit hash. HEAD is a symbolic ref
   pointing at the current branch. Committing moves the branch because writing HEAD dereferences to it.

5. **"How did you implement merge / detect conflicts?"**
   → Find the merge base (common ancestor via a DAG walk), then a three-way merge per file using
   `diff3`, which inserts conflict markers when both sides changed the same lines. Fast-forward when
   the base equals HEAD. The merge commit records two parents.

6. **"What did you leave out vs real Git?"**
   → zlib compression, object directory sharding, packfiles/delta compression, author/timestamp
   metadata, a binary index format, real network transport (SSH/HTTPS), the reflog, and rebase.

### General Git internals
7. **What are the four Git object types?** → blob, tree, commit, and **tag** (annotated tags are
   objects; ugit's tags are lightweight — just refs).
8. **Difference between a lightweight and an annotated tag?** → Lightweight = a ref pointing at a
   commit (what ugit does). Annotated = a full tag *object* with tagger, date, message, signature.
9. **What is HEAD? What is a detached HEAD?** → HEAD is a pointer to your current location, normally
   a symbolic ref to a branch. Detached = HEAD points directly at a commit; new commits belong to no
   branch and can be lost.
10. **`git reset --soft` vs `--mixed` vs `--hard`?** → soft moves HEAD only; mixed (default) moves
    HEAD + resets the index; hard also overwrites the working tree. ugit's `reset` is `--soft`.
11. **`git fetch` vs `git pull`?** → fetch downloads objects/refs only; pull = fetch + merge (or rebase).
12. **`merge` vs `rebase`?** → merge preserves history and creates a merge commit with two parents;
    rebase replays your commits onto a new base, producing a linear history (rewrites commit hashes).
13. **Fast-forward vs three-way merge?** → FF when one branch is a direct ancestor of the other (just
    move the pointer). Three-way when both diverged (needs a merge commit).
14. **What's the staging area / index for?** → It lets you compose a commit deliberately — stage
    some changes, leave others — instead of committing the entire working tree.
15. **How does Git guarantee integrity?** → Every object is addressed by the hash of its content, and
    commits/trees embed the hashes of what they reference, forming a Merkle-DAG (a Merkle tree/chain).
16. **Why SHA-1, and is it a security risk?** → Historically SHA-1 for addressing (not security).
    After the SHAttered collision, Git added collision detection and is migrating to SHA-256.
17. **What's a packfile?** → Git compresses many loose objects into a single packfile with delta
    compression (storing diffs between similar objects) for storage/transfer efficiency. ugit uses
    only loose, uncompressed objects.
18. **What does `git cherry-pick` do?** → Applies the diff of a single commit onto the current branch
    as a new commit.
19. **What is the reflog and why is it useful?** → A local log of where HEAD/branches have pointed;
    lets you recover "lost" commits after a bad reset/rebase. ugit has none.
20. **How is a commit hash computed — does the message affect it?** → Yes. The hash covers the tree,
    parents, author/committer + timestamps, and message. Change any of them and the hash changes,
    which is why rebasing rewrites hashes.

### GitHub / workflow (often asked alongside)
21. **Difference between Git and GitHub?** → Git is the distributed version-control tool (local);
    GitHub is a hosting platform adding pull requests, issues, access control, CI/CD, code review.
22. **What is a pull request?** → A request to merge one branch/fork into another, with review and
    discussion — a GitHub feature, not a Git concept.
23. **`git merge` a PR vs squash vs rebase merge on GitHub?** → merge commit keeps all commits +
    a merge node; squash collapses the PR into one commit; rebase replays commits linearly with no
    merge commit.
24. **How do you resolve a merge conflict?** → Open the conflicted files, pick/combine the code
    between the `<<<<<<<`/`=======`/`>>>>>>>` markers, remove the markers, `git add`, then commit.
25. **What is a fork vs a clone?** → A fork is a server-side copy of a repo under your account
    (GitHub); a clone is a local copy on your machine.
26. **`origin` and `upstream`?** → Conventional remote names — `origin` is your clone's source,
    `upstream` is typically the original repo you forked from.
27. **What's in `.gitignore` and how does it work?** → Patterns for paths Git shouldn't track; Git
    skips untracked files matching them. (ugit's analog is `is_ignored`, hardcoded to skip `.ugit`.)
28. **Feature-branch / Git-flow / trunk-based — what workflow do you use?** → Be ready to describe
    one: e.g. branch per feature → PR → review → CI → squash-merge to `main`.

---

## 5. Quick command cheat-sheet (ugit → what it demonstrates)

| ugit command                    | Demonstrates                                    |
|---------------------------------|-------------------------------------------------|
| `ugit init`                     | creating the object database + refs             |
| `ugit hash-object <f>`          | content-addressable write                       |
| `ugit cat-file <oid>`           | content-addressable read                        |
| `ugit add <f>` / `write-tree`   | the index / staging area, tree objects          |
| `ugit commit -m ...`            | commit objects, parent links, moving HEAD       |
| `ugit log` / `k`                | walking the commit DAG                          |
| `ugit branch` / `checkout`      | refs, symbolic refs, detached HEAD              |
| `ugit tag`                      | lightweight tags as refs                        |
| `ugit status` / `diff [--cached]`| three-tree comparison (HEAD/index/working)     |
| `ugit merge` / `merge-base`     | merge base, three-way merge, fast-forward       |
| `ugit fetch` / `push`           | object reachability, missing-object transfer    |

---

### Final tip for the interview
Lead with the **content-addressable object store** idea, then the **three object types**, then
**refs as pointers**, then the **index/three-tree model**. If you can draw the commit→tree→blob
diagram and explain that "a branch is just a file with a hash in it," you'll demonstrate real
understanding of Git internals — which is exactly why this project is impressive on a resume.
