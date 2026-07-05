# agentgate — v1 Issue Plan

Derived from `docs/DESIGN.md` (canonical). GitHub Issues are generated from
`docs/issues/*.md`; if they diverge, these files win.

## v1 completion statement

When every issue listed below (01–21) is completed and its Validation section passes,
agentgate v1 as specified in `docs/DESIGN.md` §1–§15 is complete: a documented,
CI-tested, releasable single binary providing `init`, `check`, `exec`, `hook
claude-code`, `log show|tail|verify`, `policy validate|test`, and `version`, with
layered fail-closed policy evaluation, shell static analysis with wrapper/nesting/
substitution handling, protected-path overlays, and a hash-chained audit log — with no
v1 behavior specified only in prose outside this plan. Remaining work beyond these
issues is limited to newly discovered implementation unknowns (tracked in §Known
unknowns) and v2 items.

## Issue list (recommended execution order)

| # | File | Title (GitHub) | Size |
|---|---|---|---|
| 01 | `issues/01-repo-scaffold-go-cli.md` | Scaffold Go module, cobra CLI skeleton, Makefile, lint config | S |
| 02 | `issues/02-ci-build-test-lint.md` | Add GitHub Actions CI: build, test (race), lint on macOS+Linux | S |
| 03 | `issues/03-config-home-and-policy-discovery.md` | Implement AGENTGATE_HOME resolution and policy file discovery | S |
| 04 | `issues/04-policy-schema-and-validate.md` | Implement policy YAML schema, strict loader, and `policy validate` | M |
| 05 | `issues/05-glob-matching-primitives.md` | Implement match primitives: program, positional args, any_arg, cwd, raw | S |
| 06 | `issues/06-shell-parsing-command-units.md` | Implement shell AST walk extracting command units and hazards | L |
| 07 | `issues/07-wrapper-unwrap-and-nested-shells.md` | Implement wrapper unwrapping table and nested-shell recursion | M |
| 08 | `issues/08-policy-evaluation-engine.md` | Implement layered first-match engine with most-restrictive aggregation | M |
| 09 | `issues/09-redirects-and-protected-paths.md` | Implement redirect extraction and protected-path escalation overlays | M |
| 10 | `issues/10-audit-log-writer-hash-chain.md` | Implement audit JSONL writer with flock and SHA-256 hash chain | M |
| 11 | `issues/11-check-command.md` | Implement `agentgate check` with human/JSON output and exit codes | M |
| 12 | `issues/12-exec-command.md` | Implement `agentgate exec` with ask prompt, dry-run, exit passthrough | M |
| 13 | `issues/13-explain-trace-output.md` | Implement `check --explain` rule-trace rendering | S |
| 14 | `issues/14-init-and-starter-policy.md` | Implement `agentgate init` and the embedded starter policy | M |
| 15 | `issues/15-claude-code-hook-adapter.md` | Implement `agentgate hook claude-code` PreToolUse adapter | M |
| 16 | `issues/16-log-query-commands.md` | Implement `agentgate log show` and `log tail` | S |
| 17 | `issues/17-log-verify-command.md` | Implement `agentgate log verify` chain verification | S |
| 18 | `issues/18-policy-test-command.md` | Implement `agentgate policy test` for embedded policy tests | S |
| 19 | `issues/19-e2e-and-bypass-corpus.md` | Add bypass corpus and testscript end-to-end suite | L |
| 20 | `issues/20-user-docs-and-examples.md` | Write README, usage docs, policy reference, examples | M |
| 21 | `issues/21-release-workflow.md` | Add goreleaser config and manual release workflow | S |

## Dependency table

| Issue | Depends on | Parallel with |
|---|---|---|
| 01 | — | — |
| 02 | 01 | 03, 05, 06 |
| 03 | 01 | 02, 05, 06 |
| 04 | 03 | 05, 06 |
| 05 | 01 | 03, 04, 06 |
| 06 | 01 | 03, 04, 05 |
| 07 | 06 | 04, 05 |
| 08 | 04, 05, 07 | — |
| 09 | 08 | 10 |
| 10 | 03, 08 | 09 |
| 11 | 09, 10 | — |
| 12 | 11 | 13, 14, 15, 16, 17, 18 |
| 13 | 11 | 12, 14–18 |
| 14 | 04 (soft: 18 to run embedded tests) | 12, 13, 15–18 |
| 15 | 09, 10 | 12, 13, 14, 16, 17, 18 |
| 16 | 10 | 12–15, 17, 18 |
| 17 | 10 | 12–16, 18 |
| 18 | 08 | 12–17 |
| 19 | 11, 12, 14, 15, 16, 17, 18 | 20 |
| 20 | 11–18 | 19 |
| 21 | 02, 19 | — |

## Implementation waves

| Wave | Issues | Theme | Exit criterion |
|---|---|---|---|
| 0 | 01, 02 | Foundation | `make build test lint` green in CI on macOS+Linux |
| 1 | 03, 04, 05, 06 | Config, schema, primitives (parallelizable) | policies load+validate; globs match; units extract |
| 2 | 07, 08, 09 | Analysis + engine | engine returns correct decisions for §8/§9 test tables |
| 3 | 10 | Audit core | chained records written under concurrency, verified in tests |
| 4 | 11, 12, 13, 14 | CLI commands | full local workflow: init → check/exec (+explain, dry-run) |
| 5 | 15, 16, 17, 18 | Adapter + log UX + policy tooling | hook round-trip; log show/verify; policy test |
| 6 | 19, 20, 21 | Quality + ship | corpus green, E2E green, docs complete, release dry-run OK |

## Coverage: DESIGN.md sections → issues

| DESIGN section | Covered by |
|---|---|
| §1 Product overview | all; user-facing framing in 20 |
| §2 Integration surfaces | 11, 12, 15, 20 |
| §3 Threat model & guarantees | 19 (corpus enforces), 20 (documented honestly) |
| §4 Architecture | 01 (package skeleton), enforced by every implementation issue |
| §5 Repo layout & toolchain | 01, 02, 21 |
| §6 Runtime file & dir layout | 03, 10, 14 |
| §7 Policy schema reference | 04 (schema/validate), 05 (matching semantics) |
| §8 Command analysis pipeline | 06 (§8.1–§8.4), 07 (§8.5), 09 (§8.6), hazards table 06+07+09 |
| §9 Evaluation semantics | 08 (§9.1–§9.3, §9.5), 09 (§9.4) |
| §10 Decision JSON | 11 |
| §11.1 Exit codes | 01 (root plumbing), 11, 12 |
| §11.2 check | 11, 13 (--explain) |
| §11.3 exec | 12 |
| §11.5 log CLI | 16, 17 |
| §11.6 init | 14 |
| §11.7 policy validate | 04 |
| §11.8 policy test | 18 |
| §11.9 version | 01 |
| §12 Audit log spec | 10, 16, 17 |
| §13 Claude Code hook | 15 |
| §14 Performance targets | 19 (perf smoke test) |
| §15 Testing strategy | 02 (CI), 19 (corpus+E2E), per-issue unit tests |
| §16 Scope / deferred / unknowns | this plan (below) |

Verified: every DESIGN section §1–§16 maps to ≥ 1 issue; every issue maps back to a
DESIGN section (no orphan work).

## Validation strategy (whole product)

1. **Per-issue**: each issue's Validation section (commands + expected output) must pass
   before the issue closes; unit tests land with the issue, not after.
2. **Continuous**: CI (issue 02) runs `go build`, `go vet`, `golangci-lint`,
   `go test -race ./...` on macOS + Linux for every PR.
3. **Adversarial**: the bypass corpus (issue 19) asserts minimum decision severity for
   known evasion patterns against the starter policy; it only grows, never shrinks.
4. **End-to-end**: testscript scenarios (issue 19) drive the real binary through
   init → check/exec/hook/log → verify, including tamper detection and concurrent hook
   writes.
5. **Dogfooding gate (manual, pre-release)**: install the hook in a real Claude Code
   session per §13.4; run a scripted set of allowed/asked/denied commands; confirm
   `ask` UX and audit records. Known unknown #3 is checked here.
6. **Performance smoke** (issue 19): `check` p95 < 50 ms with a 200-rule policy;
   hook < 100 ms p99 (measured over 100 runs in CI, threshold asserted loosely at 2×
   to absorb CI noise).

## Deferred to v2 (do not implement in v1)

Codex CLI adapter; hook coverage for Write/Edit/WebFetch tools; MCP proxy mode;
`log suggest` (rule mining); policy includes/rule packs; env-var matching; signed or
forward-secure audit logs; `log prune`/retention; Windows support. See DESIGN §16.

## Known unknowns (may create new issues during implementation)

1. Claude Code hook JSON schema drift — issue 15 requires re-verification against
   current docs and records the verified version.
2. `mvdan.cc/sh` edge cases (coproc, extglob, zsh-isms in agent output) — corpus grows
   as found; may spawn parser-handling issues.
3. `ask` rendering acceptability across Claude Code surfaces (CLI/IDE/web) — checked at
   the dogfooding gate; may spawn a UX issue on the reason string.
4. GNU vs BSD `timeout`/`env` flag differences for the wrapper table (issue 07 notes).
5. Walk-up policy discovery cost on network filesystems — measure in 19; may need a
   depth cap flag.
