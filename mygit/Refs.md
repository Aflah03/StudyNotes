# ugit Walkthrough — Symbolic Refs & the Ref System

A conversational deep-dive into the `ugit` codebase (a Git-like app, work in progress),
with heavy focus on symbolic refs, how they are read/written, and the checkout/branch
behavior. Captured from a Q&A session.

---

## Q: Teach me what's going on in the ugit directory, especially symbolic refs.

### The big picture

`ugit` is a re-implementation of Git's core, following the classic "ugit" tutorial.
It's a content-addressable object store plus a naming layer on top. Three modules,
cleanly layered:

- **`ugit/data.py`** — the **storage layer**. Knows about the `.ugit` directory, the
  object database, and refs. Nothing above it should touch the filesystem directly.
- **`ugit/base.py`** — the **logic layer**. Trees, commits, tags, branches, history
  traversal. Speaks in terms of `data.*`.
- **`ugit/cli.py`** — the **CLI layer**. `argparse` wiring; every subcommand maps to a
  `base` (or `data`) call.

Everything lives under `.ugit/` (constant `GIT_DIR` in `data.py`):

```
.ugit/
├── HEAD                       # where we are now
├── objects/<oid>              # every blob/tree/commit, keyed by sha1
└── refs/
    ├── heads/<branch>         # branches
    └── tags/<tag>             # tags
```

### The object store

Everything you store is an **object** named by the SHA-1 of its content
(`data.hash_object`):

```python
obj = type_.encode() + b'\x00' + data   # "blob\x00<bytes>"
oid = hashlib.sha1(obj).hexdigest()
```

The type prefix + NUL byte is prepended *before* hashing, so the type is part of the
identity and can be validated on read (`get_object` splits on the NUL and asserts
`type_ == expected`).

Three object types:

- **blob** — file contents.
- **tree** — a directory listing. Text, one entry per line: `{type} {oid} {name}`
  (`write_tree`). `write_tree` recurses the working dir; `read_tree` empties the current
  dir and materializes a tree back to disk.
- **commit** — `tree <oid>`, optional `parent <oid>`, blank line, message (`commit` /
  `get_commit`).

History is just commits pointing at parents; `iter_commits_and_parents` does a
breadth-first walk (a `deque` + `visited` set) so `log` and `k` can traverse it.

---

## Symbolic refs — the core concept

A **ref** ("reference") is a human-friendly name that points at an OID, so you say
`master` instead of `b30cf1f2…`. In ugit a ref is literally a small file under `.ugit/`
whose contents are the thing it points to.

There are **two kinds of ref**, and this is the whole idea:

1. **Direct ref** — the file contains a 40-char OID.
   ```
   .ugit/refs/heads/new_branch   →   b30cf1f212dcc27bbc5b3121488ae726911d84e8
   ```
2. **Symbolic ref** — the file contains `ref: <another-ref-name>`. It points at *another
   ref*, not at an object.
   ```
   .ugit/HEAD   →   ref: refs/heads/new_branch
   ```

The point of a symbolic ref is indirection: `HEAD` says "I am wherever
`refs/heads/new_branch` is." When you commit, the branch moves forward and `HEAD`
automatically follows, because `HEAD` never named the OID directly — it named the branch.

This is modeled by one small namedtuple:

```python
Refvalue = namedtuple('Refvalue', ['symbolic', 'value'])
```

- `symbolic=True`  → `value` is a **ref name** (e.g. `refs/heads/new_branch`)
- `symbolic=False` → `value` is an **OID** (e.g. `b30cf1f2…`)

### The engine: `_get_ref_internal`

Every read and write goes through one function:

```python
def _get_ref_internal(ref, deref):
    ref_path = f'{GIT_DIR}/{ref}'
    value = None
    if os.path.isfile(ref_path):
        with open(ref_path) as f:
            value = f.read().strip()          # note the .strip() — trailing newline safety

    symbolic = bool(value) and value.startswith('ref:')
    if symbolic:
        value = value.split(':', 1)[1].strip()   # 'ref: refs/heads/x' -> 'refs/heads/x'
        if deref:
            return _get_ref_internal(value, deref=True)   # follow the chain

    return ref, Refvalue(symbolic=symbolic, value=value)
```

Two things to notice:

**(a) It returns a tuple `(ref, Refvalue)`.** The first element is *which ref file we
ended up at*; the second is the value. That first element is the secret to writes.

**(b) The `deref` flag decides whether to follow the pointer.**

- `deref=True` (default): when the ref is symbolic, it recurses into the target and keeps
  going until it hits a direct ref. So `get_ref('HEAD')` on a symbolic HEAD returns the
  **final OID**, and the returned `ref` name is the concrete branch it landed on
  (`refs/heads/new_branch`).
- `deref=False`: it stops at the first ref and returns the symbolic value itself. So you
  learn "HEAD points to `refs/heads/new_branch`" *without* resolving to an OID.

The two public wrappers just pick which half of the tuple they want:

```python
def get_ref(ref, deref=True):
    return _get_ref_internal(ref, deref)[1]     # the Refvalue
```

### Why writes need `deref` too — `update_ref`

```python
def update_ref(ref, value, deref=True):
    ref = _get_ref_internal(ref, deref)[0]      # <-- resolve WHICH file to write
    assert value.value
    if value.symbolic:
        value = f'ref: {value.value}'
    else:
        value = value.value
    ref_path = f'{GIT_DIR}/{ref}'
    os.makedirs(os.path.dirname(ref_path), exist_ok=True)
    with open(ref_path, 'w') as f:
        f.write(value)
```

It uses the **first** element of the tuple (`[0]`), the resolved ref name. So if `HEAD`
is symbolic → `refs/heads/new_branch`, then `update_ref('HEAD', <oid>)` doesn't overwrite
`HEAD`; it **writes through** to `refs/heads/new_branch`. That is exactly how, in real
Git, `git commit` while on a branch advances the branch and leaves `HEAD` as
`ref: refs/heads/…`.

When you *do* want to move `HEAD` itself (e.g. switching branches), you pass `deref=False`
so resolution stops at `HEAD` and writes there.

### Worked examples

**Example 1 — reading a symbolic HEAD (chained deref)**

```
.ugit/HEAD                    →  ref: refs/heads/master
.ugit/refs/heads/master       →  b30cf1f2…
```

`get_ref('HEAD')` (deref=True):
1. read `HEAD` → `ref: refs/heads/master`, symbolic=True, target=`refs/heads/master`
2. recurse → read `refs/heads/master` → `b30cf1f2…`, symbolic=False
3. returns `Refvalue(symbolic=False, value='b30cf1f2…')`, resolved name `refs/heads/master`

`get_ref('HEAD', deref=False)`:
- stops at step 1 → `Refvalue(symbolic=True, value='refs/heads/master')`

**Example 2 — commit writes *through* HEAD to the branch**

`update_ref('HEAD', Refvalue(False, newoid))`:
- `_get_ref_internal('HEAD', deref=True)[0]` → `'refs/heads/master'`
- writes `newoid` into `refs/heads/master`
- `HEAD` file untouched → still `ref: refs/heads/master`. Branch advanced, HEAD followed.

**Example 3 — switching branches moves HEAD itself**

`update_ref('HEAD', Refvalue(True, 'refs/heads/feature'), deref=False)`:
- resolution stops at `HEAD`
- `value.symbolic` True → writes literal `ref: refs/heads/feature` into `HEAD`.

**Example 4 — `get_oid`, the two-pass existence check**

```python
if data.get_ref(ref, deref=False).value:   # does this ref exist at all?
    return data.get_ref(ref).value          # now fully deref to the OID
```

The first call uses `deref=False` so a symbolic ref registers as "exists" even if its
target is missing; the second resolves the real OID.

**Example 5 — how `k` visualizes refs**

`k` iterates with `deref=False` on purpose:

```python
for refname, ref in data.iter_refs(deref=False):
    dot += f'"{refname}" -> "{ref.value}"\n'
    if not ref.symbolic:
        oids.add(ref.value)
```

Because it doesn't deref, a symbolic `HEAD` draws an arrow to `refs/heads/master`
(ref→ref), and only *direct* refs seed the commit-graph walk.

### Edge cases & gotchas

1. **The live repo has a *detached* HEAD, not a symbolic one.** In `new/.ugit`, `HEAD`
   contains a raw OID, not `ref: …`. `commit` and `checkout` currently write `HEAD` as a
   **non-symbolic** ref. The symbolic-ref machinery is built in `data.py`, but `base.py`
   isn't using it for HEAD yet.

2. **Live bug: `Value=` vs `value=`.** All four writers in `base.py` construct the
   namedtuple with a capital-V keyword: `data.Refvalue(symbolic=False, Value=oid)`. But
   the field is lowercase `value`. `namedtuple` raises
   `TypeError: __new__() got an unexpected keyword argument 'Value'`. So **`commit`,
   `checkout`, `create_tag`, and `create_branch` currently crash.** The refs in
   `new/.ugit` were created by an earlier version before this typo.

3. **Dangling symbolic ref → `symbolic` collapses to False.** If `HEAD` = `ref:
   refs/heads/ghost` and the target doesn't exist, full deref recurses into the missing
   file, reads `None`, and returns `Refvalue(symbolic=False, value=None)`. Callers that
   care about "is HEAD symbolic?" must use `deref=False`.

4. **`assert value.value` forbids empty writes.** You can never point a ref at
   `None`/empty — so "delete a ref" isn't supported through `update_ref`.

5. **`.strip()` everywhere is load-bearing.** Files are written without trailing
   newlines, and every read strips. Commit `f914979` ("Fix trailing spaces… using
   .strip()") fixed exactly this class of bug.

6. **Chained symbolic refs work** (ref→ref→ref→oid) via unconditional recursion. There's
   **no cycle guard** — a self-pointing symbolic ref recurses to `RecursionError`.

7. **Windows path separators leak into ref names.** In `iter_refs`, `os.walk` +
   `os.path.relpath` yield `root` with backslashes, then it builds `f'{root}/{name}'` →
   `refs\heads/new_branch`. Works because Windows accepts mixed separators, but ref names
   are inconsistent. `is_ignored` already normalizes `\` → `/` and could be reused.

---

## Q: How are symbolic refs written into (disk)?

Symbolic refs are written by the same function as direct refs — `update_ref`. What makes
the file *symbolic* is entirely decided by the `Refvalue` you hand it.

```python
def update_ref(ref, value, deref=True):
    ref = _get_ref_internal(ref, deref)[0]        # 1. decide WHICH file
    assert value.value
    if value.symbolic:                             # 2. decide WHAT to write
        value = f'ref: {value.value}'
    else:
        value = value.value
    ref_path = f'{GIT_DIR}/{ref}'
    os.makedirs(os.path.dirname(ref_path), exist_ok=True)
    with open(ref_path, 'w') as f:                 # 3. write plain text
        f.write(value)
```

Three decisions:

1. **Which file** — `_get_ref_internal(ref, deref)[0]` returns the resolved ref name. To
   write a symbolic ref you almost always pass **`deref=False`**, so resolution stops at
   the ref you named and doesn't follow any existing pointer.
2. **What bytes** — the only place symbolic-ness becomes concrete:
   - `value.symbolic == True` → writes literal **`ref: <target>`** (that prefix is what
     makes it symbolic on read).
   - `value.symbolic == False` → writes the raw OID.
3. **Write** — plain `open(..., 'w').write(value)`. No newline (why reads `.strip()`).

**Example — making HEAD symbolic:**

```python
data.update_ref(
    'HEAD',
    data.Refvalue(symbolic=True, value='refs/heads/master'),
    deref=False,          # essential: write HEAD itself, don't follow it
)
```

- `_get_ref_internal('HEAD', deref=False)[0]` → `'HEAD'`
- `value.symbolic` True → `value = 'ref: refs/heads/master'`
- writes that string into `.ugit/HEAD`

**Why `deref=False` matters:** if you passed `deref=True` while HEAD was already symbolic
(pointing at `refs/heads/old`), resolution would land on `refs/heads/old` and you'd
overwrite **the branch file** with `ref: refs/heads/master` — a branch that is itself
symbolic, almost never what you want.

Rule of thumb:
- **Writing a symbolic ref** (moving HEAD to point at a branch) → `deref=False`.
- **Writing a value through a symbolic ref** (committing → advance branch) → `deref=True`
  with a non-symbolic `Refvalue`.

**The catch:** nobody currently writes symbolic refs — every `update_ref` in `base.py`
passes `symbolic=False` (and carries the `Value=` typo). That's why the live `HEAD` holds
a raw OID.

---

## Q: What do `get_oid` and `get_commit` return now?

### `get_oid(name)`

Returns a **string: a 40-char OID** (or raises `AssertionError`). Never a `Refvalue`.

```python
def get_oid(name):
    if name == '@':
        name = 'HEAD'
    refs_to_try = [
        f'{name}',
        f'refs/{name}',
        f'refs/tags/{name}',
        f'refs/heads/{name}',
    ]
    for ref in refs_to_try:
        if data.get_ref(ref, deref=False).value:   # existence check (no deref)
            return data.get_ref(ref).value          # resolved OID (full deref)
    is_hex = all(c in string.hexdigits for c in name)
    if len(name) == 40 and is_hex:
        return name
    assert False, f'Unknow name {name}'
```

- **A name that resolves as a ref** (`@`, `HEAD`, a tag, a branch, or full `refs/...`) →
  the **OID string** it points at. `deref=False` tests existence; the second call fully
  dereferences.
- **A raw 40-char hex string** → returned **unchanged**.
- **Anything else** → `AssertionError: Unknow name <name>`.

Used as `type=oid` on `object`/`oid`/`tree`/`start_point` args so `base` functions receive
plain OIDs.

⚠️ Edge case: a **dangling symbolic ref** passes the `deref=False` check but the full-deref
call returns `value=None` → `get_oid` returns `None`.

### `get_commit(oid)`

Returns a **`Commit` namedtuple**: `Commit(tree, parent, message)`.

```python
Commit = namedtuple('Commit', ['tree', 'parent', 'message'])

def get_commit(oid):
    parent = None
    commit = data.get_object(oid, 'commit').decode()
    lines = iter(commit.splitlines())
    for line in itertools.takewhile(operator.truth, lines):   # stop at blank line
        key, value = line.split(' ', 1)
        if key == 'tree':
            tree = value.strip()
        elif key == 'parent':
            parent = value.strip()
        else:
            assert False, f'Unknow field {key}'
    message = '\n'.join(lines)
    return Commit(tree=tree, parent=parent, message=message)
```

- **`tree`** — OID of the commit's tree object (always present).
- **`parent`** — OID of the parent commit, or **`None`** for the first commit.
- **`message`** — everything after the first blank line, rejoined with `\n`.

`takewhile(operator.truth, lines)` consumes header lines until the empty line (falsy →
stops), leaving the shared iterator positioned at the message body.

Robustness note: `tree` is never pre-initialized (unlike `parent = None`), so a commit
lacking a `tree` line would raise `UnboundLocalError`. In practice `commit()` always
writes one.

---

## Q: What does `get_oid()` return when I pass a branch name?

The **OID string** the branch points at.

`get_oid('new_branch')` against `refs/heads/new_branch → b30cf1f2…`:

1. Not `@`, so `name` stays `'new_branch'`.
2. Tries each candidate, stops at the first existing file:
   ```
   'new_branch'              → .ugit/new_branch              (no file)
   'refs/new_branch'         → .ugit/refs/new_branch         (no file)
   'refs/tags/new_branch'    → .ugit/refs/tags/new_branch    (no file)
   'refs/heads/new_branch'   → .ugit/refs/heads/new_branch   ✅ exists
   ```
3. Existence check truthy → returns `data.get_ref('refs/heads/new_branch').value`.
4. Branch files hold a raw OID → returned verbatim.

**Result:** `'b30cf1f212dcc27bbc5b3121488ae726911d84e8'`.

Notes:
- You don't type the `refs/heads/` prefix — the `refs_to_try` list finds it.
- **Ordering matters:** `refs/tags/{name}` is tried *before* `refs/heads/{name}`. A name
  that is both a tag and a branch resolves to the **tag**.
- Branches are direct refs — no dereferencing needed.
- Unknown, non-hex name → `AssertionError: Unknow name`.

---

## Q: What hash is `b30cf1f2…`? A commit?

**Yes — it's a commit object.** A branch ref (`refs/heads/…`) always stores a **commit**
OID.

How you know from the code:
- Branch OIDs trace back to `base.commit()`, which does `hash_object(commit.encode(),
  'commit')` — the `'commit'` type is baked into the object's identity.
- `get_commit` calls `data.get_object(oid, 'commit')`, which **asserts** the stored type
  is `commit`. A branch pointing at a non-commit would blow up on read.

The type ladder:

- **branch / tag / HEAD** → point at a **commit** OID
- **commit** → contains a `tree` line → points at a **tree** OID
- **tree** → lines pointing at **blob** OIDs (files) and nested **tree** OIDs (subdirs)
- **blob** → raw file bytes

So `b30cf1f2…` (commit) → its `tree <oid>` → tree entries → blobs.

(Tags *could* point at any object, but `create_tag` is only ever given commit OIDs, so in
this codebase tags point at commits too.)

---

## Q: The tutorial "checkout: Switch branches" step — explain the new code.

### The core idea

`checkout` needs a decision it couldn't make before:

- checkout **a branch** → HEAD becomes **symbolic** (`ref: refs/heads/<name>`), so future
  commits advance the branch.
- checkout **anything else** (tag, raw OID) → HEAD points **directly** at the commit OID
  → *detached HEAD*.

To decide, `checkout` must know the **original name the user typed**, not just the
resolved OID. That ties all three edits together.

### Change 1 — the CLI stops pre-resolving (key enabler)

```diff
- checkout_parser.add_argument ('oid', type=oid)
+ checkout_parser.add_argument ('commit')
```

Previously `type=oid` made argparse run `get_oid` **at parse time**, so `base.checkout`
got an already-resolved OID and the name `branch1` was gone. Dropping `type=oid` passes
the **raw string** so `checkout` can inspect it.

```diff
- base.checkout (args.oid)
+ base.checkout (args.commit)
```

Other commands (`log`, `tag`, `branch`) keep `type=oid` — only `checkout` needs the name.

### Change 2 — `is_branch` helper

```python
def is_branch(branch):
    return data.get_ref(f'refs/heads/{branch}').value is not None
```

"Is there a ref file at `refs/heads/<name>`?" A tag or raw OID returns `False`.

### Change 3 — `checkout` rewritten

```diff
- def checkout (oid):
+ def checkout (name):
+     oid = get_oid (name)
      commit = get_commit (oid)
      read_tree (commit.tree)
-     data.update_ref ('HEAD', data.RefValue (symbolic=False, value=oid))
+
+     if is_branch (name):
+         HEAD = data.RefValue (symbolic=True,  value=f'refs/heads/{name}')
+     else:
+         HEAD = data.RefValue (symbolic=False, value=oid)
+
+     data.update_ref ('HEAD', HEAD, deref=False)
```

1. **`oid = get_oid(name)`** — resolution moved from CLI into `checkout` so it can keep
   `name`.
2. **`get_commit` + `read_tree`** — unchanged: load commit, materialize tree.
3. **Build the right HEAD:** branch → symbolic `ref: refs/heads/<name>`; otherwise →
   direct OID (detached).
4. **`update_ref('HEAD', HEAD, deref=False)`** — crucial `deref=False`: write the `HEAD`
   file itself, don't follow an existing pointer.

### The two scenarios

**`ugit checkout branch1`**
- `is_branch` → True → writes `HEAD = ref: refs/heads/branch1`
- `commit()` uses default `deref=True`, resolves HEAD → `refs/heads/branch1`, writes the
  new OID there → branch1 advances and HEAD (symbolic) follows.

**`ugit checkout 3d8773`** (same commit, by OID)
- `is_branch` → False → writes `HEAD = 3d8773…` directly → **detached HEAD**
- Now a `commit()` writes through HEAD, but HEAD isn't symbolic → new OID goes straight
  into `HEAD`. Only HEAD moves; branch1 stays put. Switching away can "lose" that commit.

### How it maps onto the current codebase

1. **Namedtuple name/keyword.** The tutorial uses `data.RefValue`. This repo defines
   **`Refvalue`** (lowercase `v`), field `value`. Existing callers pass **`Value=oid`**
   (capital `V`) — a bug that raises `TypeError`. Pick one spelling and be consistent:
   ```python
   HEAD = data.Refvalue(symbolic=True, value=f'refs/heads/{name}')
   ```
2. **Current `checkout` lacks `deref=False`** — this step adds it and the branch-aware
   logic.

The `data.py` layer (symbolic flag, `deref` on read/write, `iter_refs`) is **already
built**. This step is purely wiring `base`/`cli` to use symbolic HEAD.

### Edge cases

- **Branch/tag collision:** `checkout` checks `is_branch(name)` first (branch wins), but
  `get_oid`'s `refs_to_try` resolves tags before heads — the two can disagree on
  precedence.
- **`is_branch` does a full deref** (default `deref=True`); a dangling symbolic branch
  would be misclassified. Branches from `create_branch` are always direct, so fine.
- **Short/partial OID:** `get_oid` only accepts a full 40-char hex OID (or a ref name);
  `ugit checkout 3d8773` needs the full 40 chars in this code.

---

## Q: What does the `branch` command do now?

It **creates a new branch** (writes a ref under `refs/heads/`). It does *not* switch to it.

### CLI wiring

```python
branch_parser.add_argument('name')
branch_parser.add_argument('start_point', default='@', type=oid, nargs='?')
```

- **`name`** — branch name (required).
- **`start_point`** — optional; defaults to `'@'` → `HEAD`. `type=oid` resolves it to an
  OID at parse time.

### Handler

```python
def branch(args):
    base.create_branch(args.name, args.start_point)
    print(f'Branch {args.name} created at {args.start_point[:10]}')
```

### The work — `create_branch`

```python
def create_branch(name, oid):
    data.update_ref(f'refs/heads/{name}', data.Refvalue(symbolic=False, Value=oid))
```

Writes a **direct (non-symbolic) ref** at `.ugit/refs/heads/<name>` holding the OID. A
branch is just a file under `refs/heads/` holding a commit OID.

`ugit branch feature` with HEAD at `b30cf1f2…`:
- creates `.ugit/refs/heads/feature → b30cf1f2…`
- prints `Branch feature created at b30cf1f212`
- HEAD untouched — you'd run `ugit checkout feature` to move onto it.

Flags:
1. **`Value=` bug is here too** → `create_branch` currently crashes with `TypeError`.
2. **No validation / no "already exists" check** — will overwrite an existing branch and
   accepts names with `/` or spaces.

---

## Q: Does `ugit checkout <a new branch name>` create a new branch now?

**No.** If the name doesn't already exist, checkout **fails** — it never creates anything.

```python
def checkout(name):
    oid = get_oid(name)          # <-- fails here for an unknown name
    ...
```

`get_oid(name)` tries `name`, `refs/name`, `refs/tags/name`, `refs/heads/name`. For a
brand-new name none exist and it isn't a 40-char hex OID → `assert False, 'Unknow name'`
→ **`AssertionError`**. You never reach `is_branch`, and `is_branch` is only an existence
check, not a "create."

### Difference from real Git

Real Git needs an explicit flag to create-and-switch: **`git checkout -b <newname>`** (or
`git switch -c`). ugit has no `-b`; its `checkout` is switch-only.

### How to actually create a branch in ugit

```
ugit branch feature      # create refs/heads/feature (at HEAD by default)
ugit checkout feature     # now this resolves, so HEAD becomes ref: refs/heads/feature
```

Order matters: `checkout feature` before `branch feature` exists → `AssertionError`. Run
`branch` first.

(To get `git checkout -b` behavior you'd add a flag to the checkout parser and call
`base.create_branch` before setting HEAD — not in the current design or tutorial step.)

---

## Recurring themes / action items

- **Fix the `Value=` → `value=` typo** in all four `base.py` callers (`commit`,
  `checkout`, `create_tag`, `create_branch`) — they currently raise `TypeError`. Or
  rename the namedtuple to `RefValue` and update every call site consistently.
- **Wire up symbolic HEAD** so `checkout <branch>` sets `HEAD = ref: refs/heads/<branch>`
  (`deref=False`) and `commit` writes through it (`deref=True`, the default). The `data.py`
  machinery already supports it.
- **Suggested reading order:** `data.py` (`hash_object`/`get_object` → `_get_ref_internal`
  → `get_ref`/`update_ref`/`iter_refs`) → `base.py` (`write_tree`/`read_tree` →
  `commit`/`get_commit` → `iter_commits_and_parents` → tag/branch helpers) → `cli.py`
  (argparse plumbing).
- **Mental model:** `_get_ref_internal` returns `(where-we-ended-up, what-it-holds)`, and
  `deref` decides whether "where" follows the `ref:` pointers or stops. Reads take the
  value; writes take the location.
