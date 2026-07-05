# Issue 20: User docs and examples

## Title

Write README, usage guide, policy reference, and example policies

## Summary

Replace the one-line README with real user documentation: quickstart, Claude Code
integration, policy authoring reference with copy-paste examples, honest security
guarantees, and an `examples/` directory. English as primary; keep the existing
Japanese one-liner as a subtitle.

## Context

v1 ships when a stranger can install, connect Claude Code, author a policy, and read
their audit log using only the repo docs. The honesty requirements of DESIGN §3 apply
to marketing copy too: agentgate is a decision engine, not a sandbox.

## Scope

- `README.md` (rewrite), `docs/USAGE.md` (new), `docs/POLICY.md` (new),
  `examples/*.yaml` (new; `starter-policy.yaml` already exists from issue 14).

## Detailed Requirements

1. **README.md** sections: what it is (3 sentences + the three pillars table);
   install (`go install github.com/Saber5656/agentgate/cmd/agentgate@latest` +
   release-binary instructions once issue 21 lands); 60-second quickstart
   (`init` → `check` → hook snippet → `log show`); "What agentgate is NOT"
   (verbatim-adapted from DESIGN §3 limits: static decisions, no sandboxing,
   interpreter allowlisting caveat, tamper-evident-not-proof); links to USAGE/POLICY/
   DESIGN; license note (add `LICENSE` — MIT — in this issue; repo owner is the
   copyright holder).
2. **docs/USAGE.md**: every command with synopsis, common flags, exit codes table
   (from DESIGN §11.1), worked examples with real output blocks; the Claude Code
   integration walkthrough incl. where settings live and how `ask` appears; audit log
   review recipes (`log show --decision deny --since 24h`, verify, jq one-liners);
   troubleshooting (`check --explain`, `policy validate`, common E_* findings).
3. **docs/POLICY.md**: the policy schema reference rendered for users — every field
   from DESIGN §7 with type/default/semantics; matching semantics §7.1–§7.2 with
   a "matches / does not match" example table per matcher; layering and `enforced`
   (§9.1) with a two-file worked example; `on_opaque` and hazards in user language
   ("what makes a command opaque"); protected paths; embedded tests + `policy test`
   workflow. Every YAML block in this doc must be validated by a doc-test (a Go test
   that extracts fenced `yaml` blocks from POLICY.md and runs `policy.Load` on them —
   include the extractor in `internal/policy/doctest_test.go`).
4. **examples/**: `strict-deny-default.yaml` (default deny, tight allowlist for CI
   agents), `project-node.yaml` (project layer for a Node repo: npm/pnpm scripts
   allowed, publish asks), `project-go.yaml` (go build/test allowed, `go install`
   asks). Each with `tests:` blocks; a Go test runs `policy validate`-level Load on
   every file in `examples/`.
5. Cross-check pass: every CLI flag/exit code mentioned in docs exists (manual
   checklist in PR description, reviewed against `--help` output).

## Acceptance Criteria

- [ ] README quickstart executes verbatim on a fresh machine (documented dry-run in
      the PR).
- [ ] All fenced YAML in POLICY.md and all `examples/*.yaml` load with zero findings
      (doc-tests green).
- [ ] "What agentgate is NOT" present and consistent with DESIGN §3.
- [ ] LICENSE (MIT) added.

## Validation

```sh
go test ./internal/policy/ -run Doc -v
go test ./... -run Examples -v
```

## Dependencies

Issues 11–18 (documents shipped behavior).

## Non-goals

Website/GitHub Pages; screencasts; Japanese translation of full docs (v2 if wanted);
CHANGELOG bootstrapping (issue 21 handles release notes).

## Design References

DESIGN §1–§3 (framing + honesty), §7 (policy reference source), §11 (CLI reference
source), §13.4 (snippet).
