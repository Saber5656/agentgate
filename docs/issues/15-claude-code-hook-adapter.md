# Issue 15: Claude Code hook adapter

## Title

Implement `agentgate hook claude-code` PreToolUse adapter

## Summary

Implement `internal/adapters/claudecode` and wire `agentgate hook claude-code`: read
one PreToolUse JSON event from stdin, evaluate `Bash` commands through the engine,
emit the `hookSpecificOutput` permission-decision JSON, always exit 0, never allow on
error. DESIGN §13.

## Context

Flagship integration (DESIGN §2). The adapter is the only place Claude Code JSON shapes
may appear (schema-drift isolation, §13.3). Fail-closed mapping is fixed by ADR-002:
every error path degrades to `ask` with the error as the reason.

## Scope

- `internal/adapters/claudecode/adapter.go`: types + pure mapping functions.
- `internal/cli/hook.go` (replace stub): stdin/stdout plumbing, evaluation via the
  issue-11 `evaluate()` helper, audit, 1500ms timeout.

## Detailed Requirements

1. **Pre-implementation verification (mandatory)**: check the current official Claude
   Code hooks documentation for the PreToolUse input fields and output schema; record
   the doc URL, date, and verified Claude Code version in the package doc comment. If
   the schema differs from DESIGN §13, update DESIGN first (docs/ is canonical), then
   implement. (ISSUE_PLAN known unknown #1.)
2. Input struct (tolerant decode — unknown fields IGNORED here, unlike policy/audit):
   `session_id`, `cwd`, `hook_event_name`, `tool_name`, `tool_input` (raw JSON;
   for Bash decode `{command string}`).
3. Behavior table (DESIGN §13.2) — implement exactly:
   - `tool_name != "Bash"` → stdout `{}`, exit 0, no audit record.
   - `Bash` + evaluable → decision mapped verbatim (allow/ask/deny) to
     `permissionDecision`; `permissionDecisionReason` =
     `"<final>: rule <id> — <reason>"` for rule decisions,
     `"<final>: <decided_by>"` otherwise, plus `"; hazards: a,b"` when present; cap
     reason at 500 chars.
   - Empty/missing `tool_input.command` → treat as zero-unit evaluation → allow
     (`no-exec`).
   - Policy findings / config errors / panics (recover in the cobra RunE) / stdin not
     valid JSON / evaluation exceeding the 1500ms context timeout → permissionDecision
     `ask`, reason `"agentgate error: <detail>"`, exit 0.
4. Evaluation input: `Mode: "shell"`, `Raw: command`, `Cwd:` the event's `cwd` (fall
   back to process cwd if absent). The `--cwd` global flag is ignored in hook mode
   (event wins; document in help text).
5. Audit: `source: "hook.claude-code"`, `agent: "claude-code"`, `session` = event
   `session_id` (fallback: generated UUID). Audit failure → stderr warning, output
   JSON unaffected. No audit for non-Bash passthrough.
6. Output: single JSON object + newline on stdout, nothing else on stdout ever
   (warnings → stderr). Always exit 0 (DESIGN §13.2 — decisions are JSON, not exit
   codes).
7. **Tests** (feed stdin fixtures, capture stdout):
   - allow/ask/deny fixtures against a fixture policy → exact JSON golden per case;
   - non-Bash tool (`tool_name: "Write"`) → `{}`;
   - malformed JSON stdin → ask + error reason;
   - missing policy home → ask (config error path);
   - timeout path: inject a context already expired → ask;
   - audit record assertions (session id propagated, source correct);
   - reason truncation at 500 chars.
8. Latency guard: benchmark the full hook path (in-process) with the starter policy;
   assert < 50ms in a non-CI-gated test log line (hard perf gate lives in issue 19).

## Acceptance Criteria

- [ ] Behavior table implemented exactly; all golden tests pass.
- [ ] No error path can emit `permissionDecision: "allow"` (grep-level test: force
      every error branch in tests and assert output).
- [ ] Claude Code schema verification note (URL + date + version) present in package
      doc comment.
- [ ] Manual round-trip documented in the PR: `echo '<fixture>' | bin/agentgate hook
      claude-code` output pasted.

## Validation

```sh
printf '%s' '{"session_id":"s1","cwd":"/tmp","hook_event_name":"PreToolUse","tool_name":"Bash","tool_input":{"command":"sudo ls"}}' \
  | bin/agentgate hook claude-code | jq -e '.hookSpecificOutput.permissionDecision == "deny"'
printf '%s' '{"tool_name":"Write","hook_event_name":"PreToolUse"}' \
  | bin/agentgate hook claude-code   # {}
go test ./internal/adapters/... ./internal/cli/ -run Hook -v
```

## Dependencies

Issues 09 (engine complete), 10 (audit). Uses issue-11 plumbing.

## Non-goals

Non-Bash tools (Write/Edit — v2); PostToolUse events; auto-installing settings; legacy
`decision: approve|block` output format (deprecated — do not emit).

## Design References

DESIGN §13 (all), §2 (surface 1), §12.1 (hook source/agent); ADR-002 #5;
docs/research/integration-surfaces.md.
