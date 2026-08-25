# File Routing

Authoritative rules for classifying a target file into a module. Every
review request goes through this routing before any knowledge is loaded.

## Procedure

1. **Scope check.** Only PyTorch test files are in scope: paths under
   `test/**` or `torch/testing/**`. Anything else is out of scope — stop.

2. **Exact membership (authoritative).** Check the path against each module's
   `test-list.txt`:

   ```
   path in modules/<m>/test-list.txt  ->  module <m>
   ```

   Paths are stored repo-relative (`test/distributed/test_c10d_common.py`),
   one per line.

3. **Conflict.** A path listed in more than one module's `test-list.txt` is a
   routing conflict. Resolve by the most specific module (see the module
   table below), then fix the lists so the conflict cannot recur: remove the
   path from the less specific module's list.

4. **Path-pattern fallback (unlisted files only).** If no list matches, use
   the path patterns in the module table as a heuristic signal. Confirm a
   non-default route with the user before finalizing it, then record the path
   in that module's `test-list.txt` so the decision is deterministic next
   time.

5. **Default.** An unlisted file that no pattern claims is `core`.

## Module Table

| Module | Scope (path patterns, fallback only) | Detection | Specificity |
|--------|---------------------------------------|-----------|-------------|
| `distributed` | `test/distributed/**` | `modules/distributed/test-list.txt` | high |
| `graph` | `test/jit/**`, `test/fx/**`, `test/dynamo/**`, `test/inductor/**`, `test/export/**`, `test/onnx/**`, `test/functorch/**`, `test/higher_order_ops/**`, `test/lazy/**`, `test/cpp/**` | `modules/graph/test-list.txt` | high |
| `core` | everything else under `test/**` and `torch/testing/**` | default — no list | low |

The `test-list.txt` files are the source of truth; the path patterns exist
only to route files not yet listed.

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
   `pitfalls.md` (mirror the structure of an existing module).
2. Add its `test-list.txt` (may start empty; entries are added as files get
   routed there).
3. Add a row to the module table above and update the routing table in
   `SKILL.md`.
4. If the new module overlaps `graph`, prefer the finer-grained module for
   the overlap and remove those paths from `graph/test-list.txt`.

## Migration Sources

- Lists: `~/.claude/skills/pytorch-test-refactoring/reference/distributed/test_list.txt`,
  `~/.claude/skills/pytorch-test-refactoring/reference/graph/test_list.txt`
  (already copied in place).
- Field detection logic: `~/.claude/skills/pytorch-test-refactoring/flow.py`.
