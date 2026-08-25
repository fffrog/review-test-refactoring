# File Routing

Authoritative rules for classifying a target file into a module. Every
review request goes through this routing before any knowledge is loaded.

## Procedure

1. **Scope check.** Only PyTorch test files are in scope: paths under
   `test/**` or `torch/testing/**`. Anything else is out of scope — stop.

2. **Directory routing (authoritative).** Match the path against the
   directory patterns in the module table below. A match routes the file
   immediately — directory contents are never enumerated in any list. This
   step handles the bulk of files (everything under `test/distributed/**`,
   `test/inductor/**`, `test/jit/**`, ...).

3. **Exact membership (authoritative for files outside the directories).**
   Check the path against each module's `test-list.txt`. These lists hold
   only files no directory pattern covers: top-level files (e.g.
   `test/test_jit.py`) and explicit exceptions. Paths are stored
   repo-relative, one per line.

4. **Conflict.** A path claimed by both a directory pattern and a list
   entry, or listed in more than one module's `test-list.txt`, is a routing
   conflict. Resolve by the most specific module (see the module table),
   then fix the lists so the conflict cannot recur.

5. **Default.** An unlisted file that no directory pattern claims is `core`.
   If its content clearly belongs to another module (e.g. a new
   `test/test_inductor_*` file), confirm the route with the user and record
   it in that module's `test-list.txt` so the decision is deterministic
   next time.

## Module Table

| Module | Directories (authoritative) | Additional detection | Specificity |
|--------|------------------------------|----------------------|-------------|
| `distributed` | `test/distributed/**` | `modules/distributed/test-list.txt` (files outside the directory; currently empty) | high |
| `graph` | `test/jit/**`, `test/fx/**`, `test/dynamo/**`, `test/inductor/**`, `test/export/**`, `test/onnx/**`, `test/functorch/**`, `test/higher_order_ops/**`, `test/lazy/**`, `test/cpp/**`, `test/cpython/**` | `modules/graph/test-list.txt` (top-level files and exceptions) | high |
| `core` | everything else under `test/**` and `torch/testing/**` | default — no list | low |

The directory patterns route whole directories; the `test-list.txt` files
hold only what the patterns cannot express.

**Modules are parallel.** "Default" in the table above is a routing
catch-all only — it does not make core the baseline for other modules'
workflows or knowledge. Each module's review workflow is complete and
standalone.

## Adding a Module

A module is worth creating only when it needs a different review workflow or
different review rules than an existing module — not just a different
directory.

To register a new module `<name>`:

1. Create `reference/modules/<name>/` with `README.md`, `workflow.md`,
   `pitfalls.md` (mirror the structure of an existing module). Write all
   content in English — every file in this skill is English-only.
2. Add its `test-list.txt` (may start empty; it holds only files outside
   the module's directory patterns).
3. Add a row to the module table above and update the routing table in
   `SKILL.md`.
4. If the new module overlaps `graph`, prefer the finer-grained module for
   the overlap and remove those paths from `graph/test-list.txt`.

## Migration Sources

- Lists: `~/.claude/skills/pytorch-test-refactoring/reference/distributed/test_list.txt`,
  `~/.claude/skills/pytorch-test-refactoring/reference/graph/test_list.txt`
  (copied in place at migration, later slimmed to top-level files only —
  directory patterns now route everything else).
- Field detection logic: `~/.claude/skills/pytorch-test-refactoring/flow.py`.
