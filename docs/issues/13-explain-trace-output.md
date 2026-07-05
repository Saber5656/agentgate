# Issue 13: `check --explain` trace rendering

## Title

Implement `check --explain` rule-trace rendering

## Summary

Render the full evaluation trace (`engine.UnitResult.Trace` + layer/default/hazard
provenance) as readable text under `agentgate check --explain`. Pure rendering — the
trace data already exists (issue 08).

## Context

Policy debugging is the dominant iteration loop for users ("why was this denied?" /
"why didn't my rule match?"). DESIGN §11.2 promises: layer order, each rule tried with
matched/why-not, hazards, defaults consulted.

## Scope

`internal/cli/render.go` (extend) + enabling the already-registered `--explain` flag in
`check.go` (issue 11 left it stubbed). `--explain --json` → exit 64 (trace is not part
of the v1 JSON contract).

## Detailed Requirements

1. Output layout per unit (stdout), after the normal unit line:

```
  unit 2/3  rm -rf /   [origin: cmdsubst]
    layer order: global-enforced → project → global
    1. deny-sudo        (global-enforced)  no match: program
    2. git-readonly     (project)          no match: program
    3. rm-anywhere      (global)           MATCH → deny  "destructive"
    hazards: —
    decided by: rule rm-anywhere (deny)
```

2. Show at most the first 20 non-matching steps, then `… N more rules skipped`
   (engine already caps trace at 64; renderer caps display at 20).
3. For `decided_by: default` / `on_opaque`: print which layer's value applied and the
   resolution chain (`default: ask (from project policy /path/to/.agentgate/policy.yaml)`).
4. For hazard escalations (issue 09): print the hazard kind, the matched protected
   pattern and resolved target path.
5. Footer: policy file paths + sha256 (first 12 hex) per layer, evaluation duration.
6. `WhyNot` field names must match the schema field names (`program`, `args`,
   `any_arg`, `cwd`, `raw`).
7. Golden-file tests: 4 scenarios (rule match, default fallthrough, opaque unit,
   protected-path escalation) under `internal/cli/testdata/explain/*.golden` with an
   `-update` flag convention.

## Acceptance Criteria

- [ ] All four golden scenarios render exactly as committed goldens.
- [ ] `--explain` adds zero behavior change to decisions/exit codes (assert same code
      with and without the flag in one test).
- [ ] `--explain --json` → 64 with a clear stderr message.

## Validation

```sh
bin/agentgate check --explain --shell 'echo x > ~/.ssh/k'
go test ./internal/cli/ -run Explain -v
```

## Dependencies

Issue 11.

## Non-goals

Machine-readable trace (v2: `decision/v2` may add it); localization; color output.

## Design References

DESIGN §11.2 (--explain), §9.5 (trace data), §10 (JSON contract exclusion).
