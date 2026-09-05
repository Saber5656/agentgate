# ADR-002: Fail-closed evaluation semantics

- Status: accepted
- Date: 2026-07-05
- Deciders: design agent (Fable), pending human ratification

## Context

A tool firewall that guesses optimistically is worse than none: it creates false
confidence. Shell commands routinely contain constructs whose effect cannot be
statically determined (command substitution in program position, `eval` on variables,
`sh -c "$X"`, `xargs`). The engine needs one coherent rule for every "cannot know" case,
and one coherent rule for multi-command inputs.

## Decision

1. **Opaque → configured action, default deny.** Any command unit whose effective
   program cannot be statically resolved is *opaque* and receives the `on_opaque`
   action (project → global → built-in `deny`). The starter policy sets `ask` for
   usability; headless/built-in default stays `deny`.
2. **Most-restrictive aggregation.** The final decision for an input is the maximum
   severity (`deny > ask > allow`) across all extracted command units, including units
   from command substitutions, process substitutions, nested `sh -c` strings, and
   wrapper unwrapping. Rationale: `a && b` executes both; allowing the pair requires
   allowing each.
3. **Command substitutions execute — treat them as first-class units.** `echo $(rm -rf ~)`
   contains a real `rm` execution; it is evaluated exactly like a top-level `rm`.
4. **Escalate-only hazard overlays.** Built-in protections (protected-path writes,
   dynamic redirects) may raise a unit's decision, never lower it.
5. **Errors never allow.** Parse errors → opaque. Policy load errors → exit 65 in CLI
   modes; in hook mode → `permissionDecision: "ask"` with the error as reason. `ask`
   (not `deny`) on hook errors is deliberate: it is still fail-closed (nothing runs
   without a human), but a broken config degrades to "Claude Code asks the human", not
   "agent is bricked and the user rips the hook out".
6. **Layering: global-enforced > project > global.** Project policies may specialize or
   relax the global baseline, but rules the user marks `enforced: true` in the global
   file cannot be overridden by any repository's local policy (a repo an agent can write
   to must not be able to grant itself `sudo`).

## Alternatives considered

- *Best-effort allow on unknown constructs*: rejected — silently defeats the product's
  core promise.
- *Hard `deny` on hook errors*: rejected for recoverability (see 5).
- *Most-restrictive across layers instead of ordered first-match*: rejected — makes
  per-repo allowlisting of globally-asked commands impossible, which is the primary
  project-policy use case. `enforced` covers the security-critical subset.

## Consequences

- Some common idioms (`VAR=$(git rev-parse HEAD)`) surface as `ask` under the starter
  policy until users add rules; `check --explain` and `policy test` exist to make that
  iteration cheap.
- The hazard/opaque taxonomy (DESIGN §8.7) is a closed, testable set; the bypass corpus
  (DESIGN §15) pins the behavior.
