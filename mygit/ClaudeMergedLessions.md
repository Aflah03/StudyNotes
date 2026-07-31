# ugit — How Merge Works (Step-by-Step Lessons)

A slow, incremental walkthrough of how `merge` is implemented in the ugit tutorial,
written to match this repo's own conventions (`data.Refvalue`, `.value`,
`data.get_ref('HEAD').value`, `iter_commits_and_parents`, etc.).

> **Status of this repo when these lessons were written:** merge is *not yet*
> implemented. The code is at the *refs / branches / `k` visualization* stage.
> These lessons teach the code you will add.

## The road map

| Lesson | Topic | State |
|--------|-------|-------|
| 0 | What a merge *produces* (the mental model) | ✅ covered |
| 1 | `compare_trees`: lining up files across trees | ✅ covered |
| 2 | `get_merge_base`: finding the common ancestor | ✅ covered |
| 3 | Fast-forward merge (the easy type) | ✅ covered |
| 4 | Three-way content merge (`merge_blobs`, `merge_trees`) | ✅ covered |
| 5 | Writing the merged tree to disk (`read_tree_merged`) | ✅ covered |
| 6 | The two-parent merge commit (`commit`, `get_commit`, `MERGE_HEAD`) | ⏳ not yet |
| 7 | Wiring the CLI + a full end-to-end run | ⏳ not yet |

**The two types of merge:**
1. **Fast-forward** — HEAD is an ancestor of the other commit (no divergence). Just slide HEAD forward. No merge commit, no content merging.
2. **Three-way merge** — the branches diverged. Find the common ancestor, merge file contents 3 ways, and create a merge commit with **two parents**.

---

## ⚠️ A bug to fix in your existing code

Your namedtuple field is `value` (lowercase) — `data.py:11`:

```python
Refvalue = namedtuple('Refvalue', ['symbolic', 'value'])
```

But your existing `commit` and `checkout` call it with a **capital V**:
`data.Refvalue(symbolic=False, Value=oid)` at `base.py:24` and `base.py:32`.

Tested: `Value=` throws `TypeError: unexpected keyword argument 'Value'`. Those two
lines will crash when they run. **Always spell it `value=` (lowercase).** Fix the two
existing lines too (side quest).

---

## Lesson 0 — What does a merge actually *produce*?

In ugit, a **commit** points to two things (`base.py:10`):

```
tree   <-- a snapshot of ALL your files at that moment
parent <-- the commit that came before it
```

So history is a chain, each commit pointing back to one parent:

```
O <---- A <---- B      (each arrow = "my parent is...")
```

Now imagine two branches started from the same commit `O` and worked separately:

```
        A <---- B        <- "main" branch did commits A then B
       /
O <---
       \
        C                <- "feature" branch did commit C
```

Both `B` and `C` came from `O` but don't know about each other. **Merging combines
B's snapshot and C's snapshot back into one.**

The result is a new commit `M` that is special in one way: **it has TWO parents.**

```
        A <---- B <------ M
       /                 /
O <---                  /
       \               /
        C <-----------
```

`M` points back to *both* `B` and `C`. Its tree is the combined snapshot.

**Lock in two things:**
1. A merge's job is to build **one tree** out of the two branches' trees (plus the
   original `O` tree — that's why it's a *three*-way merge).
2. The merge commit is a normal commit that happens to list **two parents**.

The commit that is an ancestor of **both** B and C is `O` — remember it for Lesson 2.

---

## Lesson 1 — `compare_trees`: lining files up side by side

### Why we need it

To combine two snapshots we compare them **file by file**. `get_tree` (`base.py:86`)
gives a flat dict `{path: oid}`, e.g. `{'a.txt': 'AAA', 'b.txt': 'BBB'}`. With two such
dicts we need something that "zips" them by filename (one branch may have a file the
other doesn't). That tool is `compare_trees`.

### What it does

You hand it several tree-dicts. It yields, **one path at a time**, that path followed
by *its oid in each tree* (or `None` if a tree lacks the file).

### Tiny concrete example

```python
tree1 = {'a.txt': 'AAA', 'b.txt': 'BBB'}
tree2 = {'a.txt': 'A_changed', 'c.txt': 'CCC'}
```

`compare_trees(tree1, tree2)` yields:

```
('a.txt', 'AAA',   'A_changed')   # in both
('b.txt', 'BBB',   None)          # only tree1
('c.txt', None,    'CCC')         # only tree2
```

Each row is `(path, oid-in-tree1, oid-in-tree2)`. The `None` marks "missing here" —
later that's how we detect files added/deleted on one side.

### The code (`ugit/diff.py` — new file)

```python
from collections import defaultdict
from . import data

def compare_trees(*trees):
    entries = defaultdict(lambda: [None] * len(trees))

    for i, tree in enumerate(trees):
        for path, oid in tree.items():
            entries[path][i] = oid

    for path, oids in entries.items():
        yield (path, *oids)
```

Line by line:

- **`def compare_trees(*trees):`** — `*trees` accepts any number of tree-dicts. For
  `compare_trees(tree1, tree2)`, `trees` is `(tree1, tree2)`. A real merge passes three.
- **`entries = defaultdict(lambda: [None] * len(trees))`** — a dict whose keys are paths
  and whose values are a list with one slot per tree, starting `[None, None]`.
  `defaultdict` auto-creates that `[None, None]` the first time a new path is touched,
  so we never write "if path not in dict".
- **The nested loop** — `enumerate(trees)` gives `i` (tree position = which slot) and
  the tree. For each `path, oid` we set `entries[path][i] = oid`.
- **The yield loop** — makes this a generator (like your `iter_commits_and_parents`).
  `*oids` unpacks the list into the tuple, so `('a.txt', *['AAA','A_changed'])` becomes
  `('a.txt', 'AAA', 'A_changed')`.

### Trace of `compare_trees(tree1, tree2)`

After both loops, `entries` is:

```python
{
  'a.txt': ['AAA', 'A_changed'],
  'b.txt': ['BBB', None],
  'c.txt': [None,  'CCC'],
}
```

Yielded rows:

```
('a.txt', 'AAA', 'A_changed')
('b.txt', 'BBB', None)
('c.txt', None,  'CCC')
```

### How we consume it later

```python
# two-way (diff)
for path, o_from, o_to in compare_trees(t_from, t_to):
    if o_from != o_to:
        ...   # this file differs

# three-way (merge)
for path, o_base, o_HEAD, o_other in compare_trees(t_base, t_HEAD, t_other):
    ...
```

Same function, three trees, four-element rows.

---

## Lesson 2 — `get_merge_base`: finding the common ancestor

The commit `O` where two branches split is the **merge base**. This lesson finds it.

### Reminder: your existing `iter_commits_and_parents` (`base.py:61`)

```python
def iter_commits_and_parents(oids):
    oids = deque(oids)
    visited = set()

    while oids:
        oid = oids.popleft()
        if not oid or oid in visited:
            continue
        visited.add(oid)
        yield oid

        commit = get_commit(oid)
        oids.appendleft(commit.parent)
```

Give it starting commits; it yields them plus every ancestor, **newest first**.

Trace of `iter_commits_and_parents({B})` on history `O <- A <- B`:

| Step | queue | visited | action |
|------|-------|---------|--------|
| 1 | pop `B` | `{B}` | yield **B**, queue `[A]` |
| 2 | pop `A` | `{B,A}` | yield **A**, queue `[O]` |
| 3 | pop `O` | `{B,A,O}` | yield **O**, queue `[None]` |
| 4 | pop `None` | — | skip (empty) |

Yields **B, A, O**. Two things it relies on:
- `visited` prevents processing a commit twice (vital once merge commits rejoin history).
- Newest-first ordering makes the *first* shared commit we find the *closest* one.

### The idea

1. Walk all ancestors of B into a set ("everything B can reach").
2. Walk C's ancestors newest-first; for each, ask "is it in B's set?"
3. The **first** match is the merge base (closest, because of newest-first order).

### The code (`ugit/base.py`)

```python
def get_merge_base(oid1, oid2):
    parents1 = set(iter_commits_and_parents({oid1}))

    for oid in iter_commits_and_parents({oid2}):
        if oid in parents1:
            return oid
```

- **`parents1 = set(iter_commits_and_parents({oid1}))`** — drains the generator into a
  set, e.g. `{B, A, O}`. A set makes the `in` test instant. `{oid1}` is a one-element
  set because the walker expects a collection of start points.
- **The loop + `if oid in parents1: return oid`** — walk `oid2`'s ancestors newest-first
  and return the first that is also an ancestor of `oid1`.

### Trace 1 — simple (`get_merge_base(B, C)`), histories `O<-A<-B` and `O<-C`

`parents1 = {B, A, O}`. Walk C's ancestors (C, O):

| checking | in `{B,A,O}`? | result |
|----------|----------------|--------|
| `C` | no | continue |
| `O` | **yes** | `return O` ✅ |

### Trace 2 — closest wins

```
O <- P <- A <- B      (main, at B)
           \
            C         (feature, at C)
```

`parents1 = {B, A, P, O}`. Walk C's ancestors (C, A, P, O):

| checking | in set? | result |
|----------|---------|--------|
| `C` | no | continue |
| `A` | **yes** | `return A` ✅ (never even checks P or O) |

Newest-first ordering returns the closest shared commit `A`, not the ancient `O`.

### Trace 3 — the fast-forward signal

HEAD at `A`, other at `B`, history `O <- A <- B`. `get_merge_base(B, A)`:

`parents1 = {B, A, O}`. Walk A's ancestors (A, O): first check `A` → in set → `return A`.

The merge base **equals HEAD**. That means HEAD is already an ancestor of the other
commit — you're simply behind. That equality is the trigger for a **fast-forward**
(Lesson 3). When `merge_base != HEAD`, the branches diverged → real three-way merge.

> Note: `get_merge_base` relies on `iter_commits_and_parents` following parents. Today
> the walker follows a single `commit.parent`; in Lesson 6 we upgrade it to follow *all*
> parents, and `get_merge_base` keeps working unchanged.

---

## Lesson 3 — The fast-forward merge (the easy type)

### When it happens

HEAD is at `A`; the other branch moved ahead to `B` built on top of `A`; you made no
commits of your own:

```
O <---- A <---- B
        ^       ^
       HEAD    other
```

`B` already contains everything you have, plus more. "Merging" = just **slide HEAD
forward from A to B**. No new commit, no file-combining. Hence *fast-forward*.

### How we detect it

From Lesson 2: `get_merge_base(B, A)` returns `A`, which equals HEAD.

**If `merge_base == HEAD`, we can fast-forward.** It means HEAD is an ancestor of the
other commit → HEAD has nothing unique → we're behind.

### The three actions

1. Load the other commit's snapshot into the working dir — your `read_tree` (`base.py:122`).
2. Move HEAD to point at the other commit — `data.update_ref('HEAD', ...)`.
3. Print a note. **No new commit, no file-combining.**

### The code (`ugit/base.py`) — fast-forward half of `merge`

```python
def merge(other):
    HEAD = data.get_ref('HEAD').value
    assert HEAD

    merge_base = get_merge_base(other, HEAD)
    c_other = get_commit(other)

    # --- Fast-forward case ---
    if merge_base == HEAD:
        read_tree(c_other.tree)
        data.update_ref('HEAD',
                        data.Refvalue(symbolic=False, value=other))
        print('Fast-forward merge, no need to commit')
        return

    # --- (diverged case: Lessons 4-6) ---
    print('Real three-way merge needed - coming in later lessons')
```

- **`other`** — the commit oid we're merging in (the CLI resolves the branch name to an oid).
- **`HEAD = data.get_ref('HEAD').value`** — where we currently stand (same pattern as `base.py:15`).
- **`assert HEAD`** — can't merge into an empty repo.
- **`merge_base = get_merge_base(other, HEAD)`** — Lesson 2; the "did we diverge?" detector.
- **`c_other = get_commit(other)`** — parse the other commit to reach `c_other.tree`.
- **`if merge_base == HEAD:`** — the fast-forward test.
- **`read_tree(c_other.tree)`** — working dir now matches commit `B`.
- **`data.update_ref('HEAD', data.Refvalue(symbolic=False, value=other))`** — HEAD points at `B`.
  Note lowercase `value=`. `symbolic=False` (storing a real oid).
- **`print` + `return`** — done.

### Trace — `merge(B)` with HEAD at `A`, history `O <- A <- B`

1. `HEAD = A`.
2. `merge_base = get_merge_base(B, A) = A`.
3. `c_other = get_commit(B)` → `treeB`.
4. `merge_base == HEAD` → `A == A` → True.
5. `read_tree(treeB)` → working dir matches `B`.
6. HEAD now points at `B`.

```
O <---- A <---- B
                ^
               HEAD    (slid forward)
```

### Diverged case (preview)

HEAD at `B`, run `merge(C)` on the O→B, O→C picture:
`merge_base = O`; `O == B` is False → falls through to the three-way path (Lessons 4–6).

---

## DOUBT (Lesson 3): symbolic HEAD & `deref` — do we move the branch or detach HEAD?

**Question:** if HEAD is attached to a branch, doesn't `update_ref('HEAD', ...oid...)`
overwrite HEAD's `ref: refs/heads/main` with a bare oid and detach it? Shouldn't we
update the *branch* oid and keep HEAD symbolic?

**Answer: your instinct is exactly right — and your existing code already does it,
thanks to `deref`.**

### Two states of HEAD

- **Attached (normal):** `.ugit/HEAD` contains `ref: refs/heads/main`; the oid lives in
  `.ugit/refs/heads/main`.
- **Detached:** `.ugit/HEAD` contains a raw oid directly.

### `deref` saves us

`update_ref` (`data.py:17`) starts with:

```python
def update_ref(ref, value, deref=True):
    ref = _get_ref_internal(ref, deref)[0]   # resolve which ref to actually write
    ...
```

`deref` defaults to **`True`**. `_get_ref_internal` (`data.py:76`), when `deref=True` and
the ref is symbolic, **follows the pointer down** and returns the *resolved* ref name.
So the ref we actually write to is `refs/heads/main`, **not** `HEAD`.

### Trace (attached HEAD)

Setup: `.ugit/HEAD` = `ref: refs/heads/main`; `.ugit/refs/heads/main` = `AAA`;
fast-forward to `other = BBB`. Run:

```python
data.update_ref('HEAD', data.Refvalue(symbolic=False, value='BBB'))  # deref=True
```

1. `ref = _get_ref_internal('HEAD', True)[0]`
2. reads HEAD → `ref: refs/heads/main` (symbolic) → recurse on `refs/heads/main`
3. reads `refs/heads/main` → `AAA` (not symbolic) → returns `('refs/heads/main', ...)`
4. `[0]` = `'refs/heads/main'`
5. writes `'BBB'` into `.ugit/refs/heads/main`

**End state:** `refs/heads/main` → `BBB` (branch moved); `HEAD` still `ref: refs/heads/main`
(still attached). Exactly what you wanted — no special-case code needed.

### Subtle correction

We don't *set* HEAD symbolic in the merge — HEAD is *already* symbolic and we leave it
alone. We pass `symbolic=False` + the oid because the thing ultimately written (the
branch file) holds an oid.

### Detached case still works

If HEAD is detached, `_get_ref_internal('HEAD', True)` sees it's not symbolic, doesn't
recurse, returns `('HEAD', ...)`, and we write the oid straight into HEAD. Same line
handles both.

### When you'd use `deref=False`

`checkout` to a branch wants to repoint HEAD *itself*:

```python
data.update_ref('HEAD',
                data.Refvalue(symbolic=True, value='refs/heads/otherbranch'),
                deref=False)
```

| Operation | Intent | `deref` |
|-----------|--------|---------|
| `commit`, fast-forward merge | advance the *current branch* | `True` (default) — write through HEAD |
| `checkout` to a branch | repoint HEAD *itself* | `False` — write HEAD literally |

---

## Lesson 4 — The three-way *content* merge (`merge_blobs`, `merge_trees`)

The heart of a real merge. We're inside the `else` path from Lesson 3 (branches diverged).

### Why two versions isn't enough

If a file differs between our version and theirs, with only those two you can't tell
**who changed it**, so you can't safely auto-combine. The fix: bring in a **third**
version, the **base** (common ancestor `O`), the file before either side touched it:

- **base** — the original
- **HEAD** — our version
- **other** — their version

Comparing each side *to the base* answers "who changed this?" That's why it's *three*-way.

### The decision logic (per file)

| Situation | Meaning | Action |
|-----------|---------|--------|
| HEAD == base | we didn't touch it | take **other's** version |
| other == base | they didn't touch it | take **our** version |
| HEAD == other | both made the same change | take either |
| all three differ | both changed it differently | merge line-by-line; overlaps → **conflict** |

The Unix tool **`diff3`** implements this at line granularity in one call.

### Example that just works (non-overlapping edits)

Base:
```
line1
line2
line3
```
Ours (changed line 1):
```
line1-OURS
line2
line3
```
Theirs (changed line 3):
```
line1
line2
line3-THEIRS
```
Merged automatically (no conflict):
```
line1-OURS
line2
line3-THEIRS
```

### Example that conflicts (both change line 2)

Result written with markers:
```
line1
<<<<<<< HEAD
line2-OURS
||||||| BASE
line2
=======
line2-THEIRS
>>>>>>> MERGE_HEAD
line3
```

Read the markers:
- `<<<<<<< HEAD` … `||||||| BASE` → **our** version
- `||||||| BASE` … `=======` → the **original** (base)
- `=======` … `>>>>>>> MERGE_HEAD` → **their** version

A conflict isn't an error that stops everything — it's markers left in the file for a
human to resolve. The merge still produces a file; it's just unfinished.

### `merge_blobs` — merge one file's content (`ugit/diff.py`)

```python
import subprocess
from tempfile import NamedTemporaryFile as Temp

def merge_blobs(o_base, o_HEAD, o_other):
    with Temp() as f_base, Temp() as f_HEAD, Temp() as f_other:
        # Write each version's bytes into its own temp file
        for oid, f in ((o_base, f_base), (o_HEAD, f_HEAD), (o_other, f_other)):
            if oid:
                f.write(data.get_object(oid))
                f.flush()

        with subprocess.Popen(
            ['diff3', '-m',
             '-L', 'HEAD',       f_HEAD.name,
             '-L', 'BASE',       f_base.name,
             '-L', 'MERGE_HEAD', f_other.name,
             ], stdout=subprocess.PIPE) as proc:
            output, _ = proc.communicate()
            assert proc.returncode in (0, 1)

        return output
```

- Takes three **blob oids** (any may be `None`).
- **Why temp files:** `diff3` is an external program; it reads plain files by path, not
  ugit objects. So we extract each version's bytes to a real temp file it can open.
- **`with Temp() as ...`** — `NamedTemporaryFile`; auto-deleted at block end; `f.name`
  is its path.
- **write loop** — pair each oid with its temp file. `if oid:` skips `None` (leaves that
  temp file empty = "file didn't exist here"). `data.get_object(oid)` fetches bytes
  (`data.py:110`). `f.flush()` forces bytes to disk before `diff3` reads them.
- **`diff3 -m`** — merge mode, produce merged output with markers. `-L` labels are the
  marker names (`HEAD`, `BASE`, `MERGE_HEAD`). Order of file args: mine, base, theirs.
  `stdout=PIPE` captures output.
- **`output, _ = proc.communicate()`** — run to completion; `output` = merged bytes.
- **`assert proc.returncode in (0, 1)`** — `0` = clean, `1` = merged with conflicts
  (both success); anything else is a real failure.
- **`return output`** — merged bytes (clean, or with markers).

### `merge_trees` — do it for every file (`ugit/diff.py`)

```python
def merge_trees(t_base, t_HEAD, t_other):
    tree = {}
    for path, o_base, o_HEAD, o_other in compare_trees(t_base, t_HEAD, t_other):
        tree[path] = merge_blobs(o_base, o_HEAD, o_other)
    return tree
```

- The four-element unpacking previewed in Lesson 1.
- For each path, run the three-way content merge, store the **bytes**.
- Returns `{path: merged_bytes}` — the "one combined tree" from Lesson 0, holding raw
  content ready to write to disk (Lesson 5).

Trace on:
```
t_base  : a.txt=AAA, b.txt=BBB
t_HEAD  : a.txt=AAA, b.txt=B2      (we changed b)
t_other : a.txt=A2,  b.txt=BBB     (they changed a)
```
`compare_trees` yields `('a.txt','AAA','AAA','A2')` and `('b.txt','BBB','B2','BBB')`:
- `a.txt`: ours==base → take theirs (`A2`)
- `b.txt`: theirs==base → take ours (`B2`)
Result: `{'a.txt': <A2 bytes>, 'b.txt': <B2 bytes>}`. Both edits preserved automatically.

### Windows reality

`diff3` is a Unix program, not on Windows `PATH` by default (same class of issue as
`dot -Tx11 /dev/stdin` in your `k` command). Two options:

**1. Use the real `diff3`** — bundled with Git for Windows at
`C:\Program Files\Git\usr\bin\diff3.exe`. Add that folder to `PATH` (or call the full
path) and the tutorial code runs unchanged.

**2. Pure-Python `merge_blobs`** (whole-file conflict; no external dependency; literally
the Step-2 table as code):

```python
def merge_blobs(o_base, o_HEAD, o_other):
    base  = data.get_object(o_base).decode()  if o_base  else ''
    head  = data.get_object(o_HEAD).decode()  if o_HEAD  else ''
    other = data.get_object(o_other).decode() if o_other else ''

    if head == base:
        return other.encode()
    if other == base:
        return head.encode()
    if head == other:
        return head.encode()

    return (
        '<<<<<<< HEAD\n'       + head  +
        '\n||||||| BASE\n'     + base  +
        '\n=======\n'          + other +
        '\n>>>>>>> MERGE_HEAD\n'
    ).encode()
```

Trade-off: conflicts on the *whole file* when both sides changed, whereas `diff3` merges
non-overlapping *line* changes and conflicts only on overlapping lines. Same interface
either way, so `merge_trees` and everything downstream don't care which you use.
(A `difflib`-based pure-Python version can do proper line-level merging — ask later.)

---

## DOUBT (Lesson 4/5): what does `merge_blobs` operate on — files or commits?

**Answer: `merge_blobs` works purely on blobs (file contents). It never touches commit
objects, and never touches tree objects either.** Its inputs are three **blob oids**.

Three object types (`data.py:101`):

| Type | Holds | Example |
|------|-------|---------|
| **blob** | contents of one file | `hello\nworld\n` |
| **tree** | a directory listing (`type oid name` lines) | `blob AAA a.txt` |
| **commit** | `tree ...`, `parent ...`, message | `base.py:10` |

Division of labor across three levels:

```
base.merge(other)          <-- deals with COMMITS (reads HEAD & other, finds base commit)
   |  extracts .tree from each commit
   v
merge_trees(t_base, t_HEAD, t_other)   <-- deals with TREES (loops every path)
   |  hands 3 blob oids down per path
   v
merge_blobs(o_base, o_HEAD, o_other)   <-- deals with BLOBS (one file's contents)
```

- **Commits** appear only at the top, in `base.merge`, which pulls `.tree` out of each.
- **Trees** are handled by `merge_trees` + `compare_trees`. `get_tree` (`base.py:86`)
  already flattens a tree object into `{path: blob_oid}`, so they work on paths + blob oids.
- **Blobs** are the only thing `merge_blobs` sees.

Merging *commits* is really merging their trees plus recording two parents (Lesson 6).

---

## Lesson 5 — Writing the merged tree to disk (`read_tree_merged`)

`merge_trees` gave merged **bytes in memory**; now we put them in the working directory.
Best understood next to your existing `read_tree`.

### Your existing `read_tree` (`base.py:122`)

```python
def read_tree(tree_oid):
    _empty_current_directory()
    for path, oid, in get_tree(tree_oid, base_path='./').items():
        os.makedirs(os.path.dirname(path), exist_ok=True)
        with open(path, 'wb') as f:
            f.write(data.get_object(oid))
```

1. Empty the working dir. 2. Get `{path: oid}` for **one** tree. 3. For each file: make
its folder, write it (fetching bytes with `data.get_object(oid)`).

### What changes for a merge — only two things

| | `read_tree` | `read_tree_merged` |
|---|-------------|--------------------|
| Where the dict comes from | `get_tree(one tree)` → `{path: oid}` | `merge_trees(three trees)` → `{path: bytes}` |
| How we get the bytes | `data.get_object(oid)` (look up) | the value **is** the bytes — write directly |

Everything else (empty dir, make folders, `'wb'`) is identical.

### The code (`ugit/base.py`; needs `from . import diff`)

```python
def read_tree_merged(t_base, t_HEAD, t_other):
    _empty_current_directory()
    merged = diff.merge_trees(
        get_tree(t_base,  base_path='./'),
        get_tree(t_HEAD,  base_path='./'),
        get_tree(t_other, base_path='./'),
    )
    for path, blob in merged.items():
        os.makedirs(os.path.dirname(path), exist_ok=True)
        with open(path, 'wb') as f:
            f.write(blob)
```

- **`def read_tree_merged(t_base, t_HEAD, t_other):`** — three **tree oids** (Lesson 6's
  `base.merge` gets these from each commit's `.tree`).
- **`_empty_current_directory()`** — same as `read_tree`.
- **`merge_trees(...)`** — turn each tree oid into `{path: blob_oid}` via `get_tree`
  (`base_path='./'` matches `read_tree`), then merge all three → `{path: merged_bytes}`.
  Each value is already finished bytes (clean or with conflict markers).
- **write loop** — make the folder, write the bytes. **No `data.get_object` here** —
  `blob` already *is* the bytes. That's the one real change from `read_tree`.

### Trace

```
t_base  : a.txt=AAA, b.txt=BBB
t_HEAD  : a.txt=AAA, b.txt=B2
t_other : a.txt=A2,  b.txt=BBB
```
1. empty dir. 2. `merge_trees` → `{'./a.txt': <A2 bytes>, './b.txt': <B2 bytes>}`.
3. write both. End: `a.txt` has their change, `b.txt` has ours — both combined on disk.
(If both had edited the same lines, the written file would contain conflict markers.)

### The crucial thing we did NOT do

After `read_tree_merged`, the merged files are on disk, but:
- no new blob objects created for the merged content,
- no tree object, no commit, HEAD unmoved.

Deliberate: the merged content lives only as ordinary files, so **you** can resolve any
conflict markers and then run a normal `commit`. At commit time, your existing
`write_tree` (`base.py:129`) scans the working dir, `hash_object`s every file (turning
your resolved content into real blob/tree objects then), and builds the snapshot. That's
why `merge_trees` could return plain bytes instead of oids — nothing needs an oid until
commit, and commit builds them from the working directory anyway.

### Picture so far (diverged path)

```
base.merge(other)
  |- get_merge_base -> base commit          (Lesson 2)
  |- not fast-forward                       (Lesson 3 test failed)
  \- read_tree_merged(base, HEAD, other)    (Lesson 5) -> merged files on disk
        \- merge_trees -> merge_blobs        (Lesson 4)
```

Missing: recording a **commit with two parents** + the `MERGE_HEAD` bookkeeping. That's
**Lesson 6**, where `base.merge` gets its second half.

---

## Still to come

### Lesson 6 — The two-parent merge commit (`MERGE_HEAD`)

Will cover:
- Writing `MERGE_HEAD` in `base.merge` (the diverged branch) so the *next* commit knows
  it has a second parent.
- `commit` reading `MERGE_HEAD` and adding a second `parent` line, then deleting it.
- Changing `get_commit` to store `parents` (a list) instead of a single `parent`.
- Upgrading `iter_commits_and_parents` to follow *all* parents.
- Adding `data.delete_ref`.

### Lesson 7 — Wiring the CLI + full end-to-end run

- `merge` subcommand in `cli.py`.
- Updating `log` and `k` to use `commit.parents`.
- A complete fast-forward run and a complete three-way (with conflict) run.

---

## Summary — everything merge touches

| File | Add / change | Why |
|------|--------------|-----|
| `ugit/diff.py` (new) | `compare_trees`, `merge_trees`, `merge_blobs` | Align paths across trees; three-way content merge |
| `ugit/base.py` | `read_tree_merged`, `merge`, `get_merge_base` | Orchestrate merge; decide FF vs three-way |
| `ugit/base.py` | `commit` (+MERGE_HEAD 2nd parent), `get_commit`→`parents`, `iter_commits_and_parents`→all parents | Record & traverse merge commits (Lesson 6) |
| `ugit/data.py` | `delete_ref` | Clear `MERGE_HEAD` after committing (Lesson 6) |
| `ugit/cli.py` | `merge` subcommand; update `log`/`k` to use `parents` | User command; correct graph/history (Lesson 7) |

**The two merge types:**
1. **Fast-forward** — `merge_base == HEAD`; no divergence, advance HEAD (via `deref` this
   moves the current *branch* and keeps HEAD attached). No merge commit.
2. **Three-way merge** — branches diverged; find common ancestor, merge file contents
   three ways (base/HEAD/other), write `MERGE_HEAD`, and the next `commit` becomes a
   two-parent merge commit.
