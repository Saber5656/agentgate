# Issue 09: Redirects and protected paths

## Title

Implement redirect extraction and protected-path escalation overlays

## Summary

Two halves: (a) in `internal/shellparse`, extract write-capable redirects per unit
(DESIGN §8.6); (b) in `internal/engine`, apply the built-in escalate-only overlays for
protected paths and dynamic redirects (DESIGN §9.4).

## Context

`echo key > ~/.ssh/authorized_keys` must be deny regardless of any allow rule for
`echo`. Overlays run after rule matching and can only raise severity (ADR-002 #4).

## Scope

- `internal/shellparse`: fill `CommandUnit.Redirects` (type exists from issue 06).
- `internal/engine`: overlay step inside `Evaluate`, after per-unit rule decision,
  before aggregation; plus protected-paths assembly in `New`.

## Detailed Requirements

1. **Extraction** (shellparse): for each `CallExpr`'s redirects (and stmt-level
   redirects attached to compound statements — apply to every unit inside that
   statement): ops `>`, `>>`, `<>`, `>|`, `&>`, `&>>` are write-capable. Static target
   word → `Redirect{Op, Target: <expanded>, Dynamic: false}` where expansion = tilde
   expansion + `filepath.Clean`; relative targets stay relative (resolved against cwd
   in the engine, which knows the evaluation cwd). Non-static target →
   `Redirect{Dynamic: true}` + hazard `dynamic-redirect`. Read redirects ignored.
   fd-duplication forms (`>&2`, `2>&1`) are NOT path writes — ignore.
2. **Protected paths assembly** (engine `New`): effective list = built-in
   (`$AGENTGATE_HOME/**` — pass the resolved home into `New`; extend its signature to
   `New(global, project *policy.File, home string)`) ∪ global.ProtectedPaths ∪
   project.ProtectedPaths. All patterns pre-validated/`~`-expanded by issue 04.
3. **Overlay A — static write redirect** (DESIGN §9.4 row 1): for each unit redirect
   with `Dynamic == false`: resolve target against `Input.Cwd` if relative; if it
   matches ANY protected glob (doublestar on the cleaned absolute path) → unit
   `Action = deny`, `DecidedBy = "hazard"`, append hazard kind
   `protected-path-write` to the unit and the Decision.
4. **Overlay B — write-capable program with protected path argument** (§9.4 row 2):
   programs `cp mv rm tee dd chmod chown ln truncate rsync install` (basename match on
   static argv[0]); for each static, path-looking argument (starts with `/`, `./`,
   `../`, or `~` before expansion), resolve as in Overlay A; protected match → same
   deny escalation.
5. **Overlay C — dynamic redirect target** (§9.4 row 3): unit
   `Action = max(Action, onOpaque())`; `DecidedBy = "hazard"` only if it changed.
6. Escalate-only invariant: overlays never lower a unit's action; implement as
   `unit.Action = maxSeverity(unit.Action, overlayAction)` and assert in tests.
7. New hazard kind constant: `protected-path-write` (completes the DESIGN §8.7 set).
8. **Tests**:
   - shellparse: `echo x > /tmp/a` (static), `echo x >> b.log` (relative kept),
     `echo x > "$F"` (dynamic + hazard), `cmd 2>&1` (no redirect), `&> out` handling,
     heredoc `<<EOF` ignored, stmt-level redirect on a block applies to inner units.
   - engine overlays (protected: `~/.ssh/**`, `~/.agentgate/**`; allow-all policy to
     prove escalation): `echo k > ~/.ssh/authorized_keys` → deny;
     `tee ~/.ssh/config` → deny (Overlay B); `cp x ~/.ssh/` → deny;
     `rm ./relative` under cwd `~/.ssh` → deny (relative resolution);
     `echo hi > /tmp/ok` → allow; `echo x > $F` with on_opaque ask → ask;
     deny stays deny (already-deny unit unchanged by overlays);
     `cat ~/.ssh/id_rsa` → NOT escalated (read; document as known v1 gap in the test
     comment — exfiltration via read is policy's job via rules, not overlays).
9. Wire the full pipeline: after this issue `Evaluate(in, AnalyzeShell(raw))` yields
   final decisions matching DESIGN end-to-end for non-audit paths.

## Acceptance Criteria

- [ ] All tests above pass; combined shellparse+engine coverage stays ≥ 90%.
- [ ] `~/.agentgate/**` protection works with a custom `--home` (self-protection
      follows the resolved home, not the literal `~/.agentgate`).
- [ ] Escalate-only invariant has an explicit test (allow→deny OK, deny never→ask).

## Validation

```sh
go test ./internal/shellparse/ ./internal/engine/ -run 'Redirect|Protected|Overlay' -v
```

## Dependencies

Issue 08 (engine core).

## Non-goals

Read-access protection (v2 candidate via rules); `protected_paths` removal syntax;
symlink resolution of targets (documented TOCTOU limitation, DESIGN §3).

## Design References

DESIGN §8.6, §8.7, §9.4; ADR-002 #4.
