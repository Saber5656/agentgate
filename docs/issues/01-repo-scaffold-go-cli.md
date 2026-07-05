# Issue 01: Repo scaffold — Go CLI skeleton

## Title

Scaffold Go module, cobra CLI skeleton, Makefile, lint config

## Summary

Create the Go module `github.com/Saber5656/agentgate`, the package layout from DESIGN
§5, a cobra root command with global flags, a working `version` subcommand, stub
subcommands for everything else, a Makefile, and golangci-lint configuration.

## Context

Greenfield repository (only README.md and docs/ exist). Every later issue assumes this
package skeleton, the exit-code helper, and the build tooling exist.

## Scope

- `go.mod` (Go ≥ 1.22), `go.sum` with only: `github.com/spf13/cobra`.
  (Other deps from DESIGN §5 are added by the issues that use them — do NOT pre-add.)
- `cmd/agentgate/main.go`: calls `internal/cli.Execute()`, `os.Exit` with its return.
- `internal/cli/root.go`: root command `agentgate`, global persistent flags
  `--home DIR`, `--cwd DIR`, `-q/--quiet` (DESIGN §11); help text one-liner:
  "agentgate — a local tool-execution firewall for AI agents".
- `internal/cli/exitcodes.go`: constants `ExitOK=0, ExitAsk=10, ExitDeny=20,
  ExitAskDeclined=21, ExitUsage=64, ExitConfig=65, ExitInternal=70` (DESIGN §11.1) and
  type `ExitError{Code int; Err error}`; `Execute()` maps errors to codes (default 70,
  cobra usage errors → 64).
- Stub subcommands (registered, print "not implemented" to stderr, exit 70):
  `check`, `exec`, `hook claude-code`, `log show|tail|verify`, `init`,
  `policy validate|test`. Each stub lives in its final file (`check.go`, `exec.go`,
  `hook.go`, `log.go`, `initcmd.go`, `policy.go`) so later issues edit, not create.
- `internal/version/version.go`: `var Version = "dev"`, `Commit = "none"`,
  `Date = "unknown"`; `Human() string` returning
  `agentgate VERSION (COMMIT, DATE, goX.Y, GOOS/GOARCH)`.
- `internal/cli/version.go`: `agentgate version` prints `version.Human()`, exit 0.
- Empty (doc.go-only) packages so the tree compiles: `internal/config`,
  `internal/policy`, `internal/shellparse`, `internal/match`, `internal/engine`,
  `internal/audit`, `internal/adapters/claudecode`. Each doc.go has a one-sentence
  package comment from the DESIGN §4 component table.
- `Makefile` targets: `build` (ldflags inject Version/Commit/Date into
  `internal/version`, output `bin/agentgate`), `test` (`go test ./...`),
  `race` (`go test -race ./...`), `lint` (`golangci-lint run`), `fmt` (`gofumpt -w .`),
  `clean`.
- `.golangci.yml`: enable `govet`, `staticcheck`, `errcheck`, `gofumpt`, `revive`;
  timeout 3m.
- `.gitignore`: `bin/`, coverage files.

## Detailed Requirements

1. `Execute()` signature: `func Execute() int` — builds the root command, runs it,
   translates `*ExitError` / cobra errors to the exit code, prints errors to stderr
   prefixed `agentgate: `.
2. Global flags are readable by subcommands via a small `internal/cli` context struct
   (e.g. `type app struct{ home, cwd string; quiet bool }`), NOT via package globals.
3. `--cwd` default: `os.Getwd()`. `--home` default: empty string (resolution happens in
   issue 03; stubs must not resolve it).
4. Unknown subcommand/flag → cobra error → exit 64.
5. `go vet ./...` and `golangci-lint run` pass with zero findings.

## Acceptance Criteria

- [ ] `make build` produces `bin/agentgate`; `bin/agentgate version` prints the
      DESIGN §11.9 format with injected ldflags values.
- [ ] `bin/agentgate` (no args) prints help, exit 0; `bin/agentgate nope` → exit 64.
- [ ] Every stub subcommand exists and exits 70 with "not implemented" on stderr.
- [ ] All packages from DESIGN §5 layout exist and compile.
- [ ] `make test lint` pass locally.

## Validation

```sh
make build && bin/agentgate version && bin/agentgate --help
bin/agentgate check --shell 'ls'; test $? -eq 70
bin/agentgate definitely-not-a-command; test $? -eq 64
make test && make lint
```

## Dependencies

None (first issue).

## Non-goals

Any real command behavior; config resolution; CI (issue 02); release builds (issue 21).

## Design References

DESIGN §4 (components), §5 (layout/toolchain), §11 (CLI surface, exit codes); ADR-001.
