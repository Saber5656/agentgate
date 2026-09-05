# Issue 16: `agentgate log show` and `log tail`

## Title

Implement `agentgate log show` and `log tail`

## Summary

Implement the audit-log reading CLI: filterable `log show` and polling `log tail [-f]`,
over the `internal/audit` readers from issue 10. DESIGN §11.5.

## Context

The audit log is only as useful as its review UX. Filters cover the practical
questions: "what got denied today", "what did the hook decide this session".

## Scope

`internal/cli/log.go` (replace stubs for `show` and `tail`; `verify` remains stubbed
for issue 17). Small additions to `internal/audit` for multi-segment iteration:
`func Segments(logsDir string) ([]string, error)` (sorted ascending) and
`func Iterate(logsDir string, fn func(Record) bool) error`.

## Detailed Requirements

1. `log show [--since DUR|RFC3339] [--decision allow|ask|deny] [--source S]
   [--limit N] [--json]`:
   - `--since`: Go duration (`2h`, `30m`) relative to now, or RFC3339 timestamp;
   - filters AND-ed; `--limit N` keeps the newest N after filtering (default 100;
     0 = unlimited);
   - human output: aligned columns `TS(local, second precision)  SOURCE  FINAL
     RULE(first deciding unit's rule_id or "-")  RAW(truncated 60)`, oldest first;
   - `--json`: raw record lines (the original JSONL, not re-marshaled — re-marshaling
     would invite canonicalization drift), oldest first;
   - empty result → `no matching records`, exit 0; missing logs dir → same.
2. `log tail [-f]`: without `-f` = `show --limit 10`. With `-f`: print new records as
   appended (poll segment size every 500ms; handle day rollover by re-scanning for a
   newer segment on each poll); Ctrl-C exits 0. Output format = the human format.
3. Corrupt line handling (both commands): print `warning: PATH:LINE: <err>` to stderr,
   skip the line, continue, exit 0 (reading is forgiving; `verify` is the strict one —
   issue 17).
4. Decision/source values validated → else exit 64.
5. **Tests**: fixture logs dir with 2 segments (built via the issue-10 Writer with
   injected clock): since/decision/source/limit filters (table), json passthrough
   byte-equality, corrupt middle line skip+warn, empty dir, tail -f rollover (unit
   test the poll step function, not wall-clock).

## Acceptance Criteria

- [ ] All filters work individually and combined; defaults documented in `--help`.
- [ ] `--json` output lines byte-identical to file lines.
- [ ] Corrupt lines never crash reads.

## Validation

```sh
bin/agentgate check --shell 'ls' >/dev/null
bin/agentgate log show --limit 5
bin/agentgate log show --decision deny --since 24h --json | jq -r .final
go test ./internal/cli/ -run Log -v
```

## Dependencies

Issue 10.

## Non-goals

`log verify` (17); aggregation/stats; pruning; remote shipping (v2).

## Design References

DESIGN §11.5, §12.1 (record fields), §12.2 (segments).
