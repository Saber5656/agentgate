# Issue 18: `agentgate policy test`

## Title

Implement `agentgate policy test` for embedded policy test cases

## Summary

Execute the `tests:` block of policy files against the real engine with TAP-like
output, making policy iteration test-driven. DESIGN §11.8.

## Context

Allowlist maintenance is the product's recurring user cost; embedded tests turn every
policy edit into a checked change. The starter policy (issue 14) ships tests that
double as an engine regression suite.

## Scope

`internal/cli/policy.go` (`test` subcommand). No new packages.

## Detailed Requirements

1. `agentgate policy test [FILE…]`:
   - no args: discovered global and/or project files (issue 03), each with tests;
     files without a `tests:` block are reported `# skipped: FILE (no tests)`;
   - explicit FILE args: those files.
2. **Layer context** (DESIGN §11.8): for each file under test, build the engine as
   that file's layer would see it in real evaluation:
   - global file → engine with global = file, project = nil;
   - project file → engine with project = file AND the discovered global (if any) —
     because that is what will actually happen at runtime. Print the composition
     (`# project file X tested with global Y`).
3. Per test case: evaluate `cmd` (shell mode, cwd = the policy file's directory for
   project files, process cwd for global). Pass criteria: `Final == expect` AND (if
   `rule` set) some unit's deciding rule id == `rule`.
4. Output (TAP-inspired, stable):
   `ok N - NAME` / `not ok N - NAME (expected EXPECT, got GOT; decided by X)`,
   summary line `# FILE: P passed, F failed`. Exit: all pass → 0; any fail → 1;
   load/validation findings in any target file → 65 (findings printed first).
5. `--json` flag: array of `{file, name, pass, expect, got, rule_expected,
   rule_actual}` — CI-friendly.
6. **Tests**: fixture policy with passing+failing cases (assert exit 1 and exact
   lines); rule-id mismatch counts as failure even when action matches; project-layer
   composition honored (global enforced deny makes a project test expecting allow
   fail — this exact scenario as a test); no-tests file skip; `--json` golden.

## Acceptance Criteria

- [ ] Starter policy's embedded tests all pass via
      `agentgate policy test $AGENTGATE_HOME/policy.yaml` after `init` (this closes
      the soft dependency from issue 14 — add this command to CI in this issue).
- [ ] Exit codes 0/1/65 as specified.
- [ ] Layer composition rule implemented and tested.

## Validation

```sh
bin/agentgate --home "$AG" init
bin/agentgate --home "$AG" policy test; test $? -eq 0
go test ./internal/cli/ -run PolicyTest -v
```

## Dependencies

Issue 08 (engine; 09 recommended so overlay-dependent starter tests pass).

## Non-goals

Watch mode; coverage of which rules are untested (v2 idea); mutation testing.

## Design References

DESIGN §11.8, §7 (tests schema), §9.1 (layer semantics being replicated).
