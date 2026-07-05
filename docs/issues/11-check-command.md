# Issue 11: `agentgate check`

## Title

Implement `agentgate check` with human/JSON output and exit codes

## Summary

Wire the full pipeline behind `agentgate check`: input modes, config discovery, engine
evaluation, human and `--json` rendering (decision JSON contract, DESIGN §10), audit
record, exit codes. This issue also builds the shared `internal/cli` evaluation plumbing
that `exec` (12), `--explain` (13), and the hook (15) reuse.

## Context

`check` is the reference consumer of the engine and the product's dry-run heart
(DESIGN §1 pillar 2). Its `--json` output is a stable machine contract for generic
harness integration (DESIGN §2 surface 2).

## Scope

- `internal/cli/check.go` (replace stub), plus `internal/cli/run.go`: shared helper
  `func (a *app) evaluate(mode, shellStr string, argv []string) (engine.Decision,
  engine.Input, *engine.Engine, error)` doing: ResolvePaths → Discover → policy.Load
  (findings → ExitConfig with findings printed as in `policy validate`) → engine.New →
  shellparse.Analyze{Shell,Vector} → Evaluate.
- `internal/cli/render.go`: human renderer + JSON renderer for `engine.Decision`.

## Detailed Requirements

1. **Input modes** (DESIGN §11.2), mutually exclusive, else exit 64:
   - `--shell STRING` → shell mode;
   - `-- CMD ARGS…` (cobra `ArgsLenAtDash`) → vector mode;
   - neither, and stdin is not a TTY → read ALL of stdin as one shell string;
   - neither, stdin is a TTY → usage error (64).
2. **Human output** (stdout): one line per unit:
   `<ACTION>  <decided-by:rule-id|default|on_opaque|hazard>  <argv joined, max 80 cols>`
   then `final: <ACTION>`. Zero units → `final: allow (no-exec)`. `-q` prints only the
   final line. Reasons (rule `reason`, hazard kinds) print indented under their unit.
3. **JSON output** (`--json`): exactly the DESIGN §10 document. Field mapping from
   `engine.Decision` is mechanical; `duration_ms` from `Decision.Duration`. One JSON
   object, one trailing newline, nothing else on stdout (errors → stderr).
4. **Exit codes**: allow → 0, ask → 10, deny → 20 (DESIGN §11.1); config findings →
   65; internal errors → 70.
5. **Audit**: append a Record with `source: "check"`, `agent: "cli"`, `session` =
   UUIDv4 per process (small helper in `internal/cli`; `crypto/rand`-based, no new
   dependency), `Exec: nil`. Audit failure → stderr warning `agentgate: audit: <err>`,
   decision/exit unchanged.
6. `--explain` flag: accepted but delegated to issue 13 (flag registered now, renders
   the normal output + `explain: not implemented` on stderr until 13 lands — keeps 13
   a pure-rendering issue).
7. **Tests**: CLI-level tests via `cobra` execute (in-process, temp `--home`):
   - allow/ask/deny × human/JSON snapshot tests with a fixture policy;
   - stdin mode (`echo 'sudo ls' | agentgate check`) → deny, exit 20;
   - vector mode `check -- git status` → single unit, no shell parse;
   - both `--shell` and `--` → 64;
   - broken policy file → 65, findings on stderr;
   - JSON output round-trips through `json.Unmarshal` and matches the schema id;
   - audit record appears in the temp home with source "check".

## Acceptance Criteria

- [ ] All DESIGN §10 fields present and correct in `--json` (golden test).
- [ ] Exit codes match §11.1 for allow/ask/deny/config/usage.
- [ ] Audit record written for every evaluation, including denies.
- [ ] `evaluate()` helper is the single path (no duplicate discovery logic in 12/15
      later — enforced by review, noted in code comment).

## Validation

```sh
make build
bin/agentgate init --force  # once issue 14 lands; until then use a fixture --home
bin/agentgate check --shell 'ls -la'; echo $?          # expect 0 or 10 per policy
bin/agentgate check --shell 'sudo rm -rf /'; test $? -eq 20
bin/agentgate check --json --shell 'git status' | jq .schema  # "agentgate.decision/v1"
go test ./internal/cli/ -run Check -v
```

## Dependencies

Issues 09 (complete engine), 10 (audit writer).

## Non-goals

`--explain` rendering (13); exec (12); hook (15); shell completion.

## Design References

DESIGN §10 (JSON contract), §11.1 (exit codes), §11.2 (check), §2 (generic surface).
