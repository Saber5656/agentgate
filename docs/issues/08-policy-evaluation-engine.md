# Issue 08: Policy evaluation engine

## Title

Implement layered first-match engine with most-restrictive aggregation

## Summary

Implement `internal/engine`: assemble policy layers, evaluate every command unit
against the combined rule order, apply `on_opaque` and defaults, aggregate to a final
decision, and produce the full `Decision` object with trace data — DESIGN §9 minus the
§9.4 overlays (issue 09).

## Context

This is where allow/ask/deny is decided. Semantics are fixed by ADR-002 and DESIGN §9;
this issue turns them into code with exhaustive tests. The `Decision` object feeds four
consumers (human output, JSON, audit, hook) — get the shape right here.

## Scope

Package `internal/engine`. Public API (exact):

```go
type Action string // "allow"|"ask"|"deny"; ordering helper: Severity(Action) int {allow:1, ask:2, deny:3}

type Input struct {
    Raw  string   // original string (shell mode) or space-joined argv (vector mode)
    Mode string   // "shell" | "vector"
    Cwd  string   // absolute evaluation cwd
}
type RuleRef struct { ID, Layer string; Action Action; Reason string }
type UnitResult struct {
    Argv []string; Origin string; Opaque bool
    Action Action
    DecidedBy string   // "rule"|"default"|"on_opaque"|"hazard"
    Rule *RuleRef      // nil unless DecidedBy=="rule" (or "hazard" kept from prior rule)
    Hazards []string   // hazard kinds attached to this unit (issue 09 fills more)
    Trace []TraceStep  // every rule tried: {RuleID, Layer, Matched bool, WhyNot string}
}
type TraceStep struct { RuleID, Layer string; Matched bool; WhyNot string }
type PolicyMeta struct { Path, SHA256 string } // zero value = layer absent
type Decision struct {
    Final Action
    Units []UnitResult
    Hazards []shellparse.Hazard
    Policy struct{ Global, Project PolicyMeta }
    Duration time.Duration
}

type Engine struct { /* holds compiled layers */ }
func New(global, project *policy.File) (*Engine, error) // error: any validation Finding present
func (e *Engine) Evaluate(in Input, an shellparse.Analysis) Decision
```

## Detailed Requirements

1. **Combined rule order** (DESIGN §9.1): global-enforced (file order) → project (file
   order) → global non-enforced (file order). Precompute this slice in `New` with each
   rule's `RuleRef` and compiled matchers.
2. **Per-unit evaluation** (DESIGN §9.2):
   - `unit.Opaque` → `Action = onOpaque()`, `DecidedBy = "on_opaque"`, empty trace.
   - Else walk the combined order; per rule call `match.Program/Args/AnyArg/Cwd/Raw`
     (raw matched against `Input.Raw`); all present fields AND. First match → action,
     `DecidedBy: "rule"`, RuleRef, and stop. Record every tried rule as a TraceStep
     with a short `WhyNot` (first failing field name, e.g. `"args"`).
   - No match → layered default (project.Default → global.Default → deny),
     `DecidedBy: "default"`.
3. `onOpaque()` resolution: project.OnOpaque → global.OnOpaque → deny.
4. **Aggregation** (DESIGN §9.3): `Final = max severity across units`. Zero units →
   `Final = "allow"` with `Units == nil`; do NOT synthesize a hazard for this — the
   renderer (issue 11) prints the `no-exec` note by checking `len(Units) == 0`.
5. Trace size guard: cap `Trace` at 64 steps per unit (paranoia for huge policies;
   note in doc comment).
6. `Evaluate` must be pure (no I/O, no clock beyond Duration measurement) —
   deterministic given inputs.
7. **Tests** (the §9 semantics table, table-driven; construct `policy.File` literals
   in-code):
   - First-match within layer: two global rules matching same cmd → earlier wins.
   - Layering: project allow overrides later global deny (non-enforced) for same cmd.
   - Enforced: global `enforced` deny beats project allow for `sudo`.
   - Default resolution: project default set → used; only global → used; neither →
     deny.
   - on_opaque: same three-way resolution; opaque unit bypasses rules entirely.
   - Aggregation: units [allow, ask] → ask; [ask, deny] → deny; [] → allow.
   - AND-composition: rule `{program: git, args: [push, "**"], any_arg: "--force*"}`
     matches `git push --force-with-lease origin` and not `git push origin`.
   - Raw escape hatch: `raw: "(?i)curl .*\\|\\s*sh"` matches the raw string even
     though units see curl and sh separately.
   - Trace: denied `git push` under starter-like rules yields trace steps with correct
     `WhyNot` for at least program/args misses.
   - sudo pair from issue 07: deny-sudo rule matches wrapper unit → final deny even if
     wrapped `rm` would be allowed.

## Acceptance Criteria

- [ ] All semantics tests above pass; package coverage ≥ 90%.
- [ ] `New` rejects files that carry findings (defense in depth vs issue-04 contract).
- [ ] Benchmark `BenchmarkEvaluate200Rules` exists (guards DESIGN §14; no CI threshold
      yet — issue 19 adds the smoke assertion).

## Validation

```sh
go test ./internal/engine/ -v -cover
go test ./internal/engine/ -bench Evaluate -benchtime 100x
```

## Dependencies

Issues 04 (policy types), 05 (matchers), 07 (units incl. wrapper outputs).

## Non-goals

Protected-path/redirect overlays (09); rendering (11/13); audit (10).

## Design References

DESIGN §9.1–§9.3, §9.5; §7 (match composition); ADR-002.
