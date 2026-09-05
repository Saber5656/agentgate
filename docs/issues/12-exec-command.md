# Issue 12: `agentgate exec`

## Title

Implement `agentgate exec` with ask prompt, dry-run, and exit-code passthrough

## Summary

Implement `agentgate exec`: evaluate like `check`, then execute on allow, prompt on ask
(TTY only), refuse on deny — with `--dry-run`, `--yes`, signal forwarding, child exit
passthrough, and full audit records including execution results. DESIGN §11.3.

## Context

`exec` is the wrapper surface for harnesses without hooks and the human policy-testing
tool. Fail-closed rule: `ask` without a TTY never executes (ADR-002), unless the caller
explicitly pre-approved with `--yes`.

## Scope

`internal/cli/exec.go` (replace stub), reusing `evaluate()` and renderers from issue 11.

## Detailed Requirements

1. Input modes: `--shell STRING` or `-- CMD ARGS…` (same parsing as check; stdin mode
   NOT supported here — exit 64 if neither flag given; stdin belongs to the child).
2. Decision handling:
   - **allow** → run.
   - **ask** → `--yes` → run (audit `ask_resolved: "approved"`); else if
     `term.IsTerminal` on stdin AND stderr → print unit table + reasons to stderr,
     prompt `Execute? [y/N] ` on stderr, read one line from stdin: `y`/`Y`/`yes` → run
     (`ask_resolved: "approved"`), anything else → exit 21 (`ask_resolved:
     "declined"`). Non-TTY without `--yes` → print reason, exit 10
     (`ask_resolved: "no-tty"`).
   - **deny** → print units+reasons to stderr, exit 20. `--yes` MUST NOT override deny
     (test this).
3. Execution:
   - vector mode: `exec.Command(argv[0], argv[1:]...)`; `Cmd.Path` resolution via
     standard `exec.LookPath` semantics; NOT through a shell.
   - shell mode: `exec.Command("/bin/sh", "-c", raw)` (documented parse-vs-exec gap,
     DESIGN §11.3).
   - Working dir: the `--cwd` flag (default process cwd); env: inherited unmodified;
     stdio: inherited (`os.Stdin/out/err` directly, no pipes).
   - Signals: relay SIGINT and SIGTERM to the child's process group
     (`Setpgid: true`, relay to `-pgid`); on child exit return its code
     (`ExitError.ExitCode()`); signal-death → 128+signal (Go reports via
     `ProcessState`; map with `WaitStatus.Signaled()`).
4. `--dry-run`: identical evaluation and rendering (including what WOULD run), never
   executes, audit `source: "exec-dry"`, exit codes as `check` (0/10/20).
5. Audit: `source: "exec"`, `Exec: &ExecRecord{Ran, ExitCode, DurationMS,
   AskResolved}`; written AFTER child exit when run (records the outcome); on deny/
   declined/no-tty written immediately with `Ran: false`.
6. `--json`: allowed only with `--dry-run` (else 64) — exec's stdout belongs to the
   child.
7. **Tests** (in-process where possible; child = `/bin/echo`, `sh -c 'exit 7'`, `sleep`):
   - allow vector → child runs, stdout passes through, exit 0;
   - allow shell `exit 7` → agentgate exits 7;
   - deny → 20, child never spawned (assert via a sentinel file the child would
     create);
   - ask + `--yes` → runs; deny + `--yes` → still 20;
   - ask + non-TTY → 10, not run, audit `no-tty`;
   - `--dry-run` on an allow → 0, nothing executed;
   - audit record contains exit code and duration for a run child;
   - SIGINT forwarding: start `sleep 5` (allowed), send SIGINT to agentgate, child
     dies, exit code 130 (mark test `-short`-skipped on CI flakiness if needed —
     document).
   - TTY prompt path: use `github.com/creack/pty`? NO — new dependency forbidden
     without ADR; instead factor the prompt as
     `func askApproval(in io.Reader, out io.Writer, isTTY bool) bool` and unit-test it
     directly; the TTY detection stays thin and untested-at-unit-level (E2E covers it,
     issue 19).

## Acceptance Criteria

- [ ] All decision × TTY × flag combinations above behave and are tested.
- [ ] Child exit codes pass through verbatim; 21 reserved for declined ask.
- [ ] Deny is never executable regardless of flags.
- [ ] Audit records include execution results exactly per DESIGN §12.1.

## Validation

```sh
bin/agentgate exec -- echo hello                # runs (starter allows echo)
bin/agentgate exec --shell 'exit 7'; test $? -ne 0   # decision-dependent; with --yes on ask: 7
bin/agentgate exec -- sudo ls; test $? -eq 20
bin/agentgate exec --dry-run --json -- rm -rf / | jq .final
go test ./internal/cli/ -run Exec -v
```

## Dependencies

Issue 11 (evaluate/render plumbing, audit integration).

## Non-goals

PTY allocation for the child; env scrubbing; timeouts for the child; Windows signals.

## Design References

DESIGN §11.3, §11.1 (codes 10/20/21/passthrough), §12.1 (ExecRecord); ADR-002 (ask
fail-closed).
