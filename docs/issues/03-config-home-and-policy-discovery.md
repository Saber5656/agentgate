# Issue 03: Config — home resolution and policy discovery

## Title

Implement AGENTGATE_HOME resolution and global/project policy file discovery

## Summary

Implement `internal/config`: resolve the agentgate home directory and locate the global
and project policy files per DESIGN §6, returning raw bytes + metadata (no YAML parsing
here — that is issue 04).

## Context

Every command needs the same deterministic answer to "which policy files apply here".
DESIGN §6 fixes the layout: `$AGENTGATE_HOME` (default `~/.agentgate`) for global state;
nearest ancestor `.agentgate/policy.yaml` for the project layer.

## Scope

Package `internal/config` only. Public API (exact):

```go
type Paths struct {
    Home       string // resolved home dir (may not exist yet)
    GlobalFile string // Home + "/policy.yaml"
    LogsDir    string // Home + "/logs"
}
func ResolvePaths(homeFlag string) (Paths, error)

type Source struct {
    Layer  string // "global" | "project"
    Path   string // absolute file path
    Bytes  []byte
    SHA256 string // lowercase hex of Bytes
}
// Discover returns global and/or project sources; either may be nil (absent file).
func Discover(p Paths, cwd string) (global *Source, project *Source, err error)
```

## Detailed Requirements

1. `ResolvePaths` precedence: `homeFlag` (from `--home`) > `$AGENTGATE_HOME` >
   `~/.agentgate` (via `os.UserHomeDir`). A non-absolute flag/env value → error
   ("home must be an absolute path"; CLI maps it to exit 65).
2. `Discover`:
   - Global: read `p.GlobalFile`; `os.IsNotExist` → `global = nil` (not an error);
     any other read error → error.
   - Project: from `cwd` (must be absolute; error otherwise), walk upward: check
     `<dir>/.agentgate/policy.yaml`; first existing file wins; stop after filesystem
     root. Symlinked dirs are not resolved (walk on the lexical path).
   - If the project file found IS the global file (same cleaned path), return it only
     as global (no duplicate layer).
3. Compute `SHA256` on the exact bytes read (used later in decisions/audit, DESIGN
   §9.5/§12.1).
4. No caching, no goroutines, no YAML. Pure functions over the FS.
5. Unit tests use `t.TempDir()`; cover: flag/env/default precedence, relative home
   error, missing global, project found at cwd, project found 3 levels up, no project,
   project == global dedup, unreadable file (permission 000) → error.

## Acceptance Criteria

- [ ] API exactly as scoped (later issues compile against it unchanged).
- [ ] All behaviors in Detailed Requirements covered by table-driven tests.
- [ ] No new third-party dependencies.

## Validation

```sh
go test ./internal/config/ -run . -v
make lint
```

## Dependencies

Issue 01.

## Non-goals

YAML parsing/validation (04); creating directories (14 `init`); watching files.

## Design References

DESIGN §6 (runtime layout), §9.1 (layer assembly inputs), §11 (global flags).
