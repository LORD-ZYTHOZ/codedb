---
description: File a GitHub issue with a mandatory failing test case that proves the bug or feature gap exists
---

# File Issue with Failing Test

When filing a new GitHub issue for codedb2, you **must** include a concrete failing test that demonstrates the problem. Issues without a reproducing test will not be accepted.

## Steps

### 1. Identify the bug or feature gap

Clearly define:
- **What** is broken or missing
- **Where** in the codebase it lives (module, function, line range)
- **Why** it matters (crash, wrong result, missing coverage, etc.)

### 2. Write a failing test in `src/tests.zig`

Add a new `test "..."` block to `src/tests.zig` that:
- Exercises the exact code path that is broken or missing
- **Fails** on the current `main` branch (compile error, assertion failure, or unexpected result)
- Is minimal — only tests the specific issue, not a large integration scenario
- Follows the existing test style (uses `std.testing`, arena allocators for Explorer tests, proper `defer` cleanup)

Example pattern for an Explorer bug:

```zig
test "issue-XX: <short description of the bug>" {
    var arena = std.heap.ArenaAllocator.init(testing.allocator);
    defer arena.deinit();
    var exp = Explorer.init(arena.allocator());

    // Setup: index a file that triggers the bug
    try exp.indexFile("example.rs", "<minimal source that triggers it>");

    // Assert: the expected behavior that currently fails
    const outline = exp.getOutlineOwned("example.rs", testing.allocator);
    if (outline) |o| {
        defer testing.allocator.free(o);
        // ... specific assertions that FAIL today ...
        try testing.expect(o.symbols.len > 0);
    } else {
        return error.TestUnexpectedResult;
    }
}
```

### 3. Verify the test fails

Run:

```bash
zig build test 2>&1 | grep "issue-XX"
```

// turbo
Confirm the new test **fails**. If it passes, the issue is either already fixed or the test doesn't target the right behavior — revisit step 2.

### 4. Create the GitHub issue

Use `gh issue create` with this structure:

```
Title: <module>: <concise description>

## Problem

<1-2 sentences describing the bug or gap>

## Failing Test

\`\`\`zig
<paste the exact test block from step 2>
\`\`\`

## Expected

<What should happen when the test passes>

## Fix

<Brief guidance on where the fix should go — file, function, approach>
```

Required labels:
- `bug` for defects, omit for feature gaps
- `priority:p0` for crashes/data-loss, `priority:p2` for correctness, omit for enhancements

Example:

```bash
gh issue create \
  --title "explore: parseRustLine misses pub(super) async fn" \
  --label "bug" \
  --body "## Problem
\`parseRustLine\` does not recognize \`pub(super) async fn\` declarations, so Rust functions with this visibility+async combo are silently dropped from outlines.

## Failing Test
\`\`\`zig
test \"issue-XX: pub(super) async fn not parsed\" {
    var arena = std.heap.ArenaAllocator.init(testing.allocator);
    defer arena.deinit();
    var exp = Explorer.init(arena.allocator());
    try exp.indexFile(\"lib.rs\", \"pub(super) async fn handler() -> Result<()> {\");
    const outline = exp.getOutline(\"lib.rs\");
    try testing.expect(outline != null);
    try testing.expect(outline.?.symbols.items.len == 1);
    try testing.expectEqualStrings(\"handler\", outline.?.symbols.items[0].name);
}
\`\`\`

## Expected
The function \`handler\` appears in the outline with kind \`function\`.

## Fix
Add \`pub(super) async fn\` to the \`is_decl\` check in \`parseRustLine\` (src/explore.zig)."
```

### 5. Commit the failing test on a branch

```bash
git checkout -b issue-XX-failing-test
git add src/tests.zig
git commit -m "test: add failing test for issue #XX"
git push origin issue-XX-failing-test
```

Then reference the branch in the issue body or as a comment so the fixer can start from a known-failing state.

## Rules

- **No issue without a test.** If you cannot write a test that fails, the issue is not well-defined enough to file.
- **One test per issue.** Keep the scope tight. If you find multiple bugs, file separate issues.
- **Test name must include the issue number** (e.g., `test "issue-42: ..."`). Update it after `gh issue create` returns the number.
- **Do not fix the bug in the same PR as the failing test.** The test commit proves the bug exists; the fix comes in a separate commit or PR.
