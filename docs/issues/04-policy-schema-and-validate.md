# Issue 04: Policy schema, strict loader, and `policy validate`

## Title

Implement policy YAML schema, strict loader, semantic validation, and `policy validate`

## Summary

Implement `internal/policy`: Go types mirroring DESIGN §7, strict YAML decoding,
semantic validation with stable error codes, and wire the `agentgate policy validate`
subcommand.

## Context

The policy file is the product's primary user interface. DESIGN §7 is the normative
schema; validation errors must be precise (file:line, code) because users iterate on
policies constantly.

## Scope

Package `internal/policy` + `internal/cli/policy.go` (validate wiring only).

```go
type Action string // "allow" | "ask" | "deny"
type File struct {
    Version        int
    Default        Action   // "" if unset
    OnOpaque       Action   // "" if unset
    ProtectedPaths []string
    Rules          []Rule
    Tests          []TestCase
    // provenance
    Layer string // "global" | "project"
    Path  string
    SHA256 string
}
type Rule struct {
    ID string; Action Action; Reason string; Enforced bool; Match Match
}
type Match struct {
    Program string; Args []string; AnyArg string; Cwd string; Raw string
    // compiled forms filled by Load: rawRE *regexp.Regexp, etc.
}
type TestCase struct { Name, Cmd string; Expect Action; Rule string }

type Finding struct { Path string; Line int; Code string; Msg string }
func Load(src *config.Source) (*File, []Finding, error) // parse+validate one file
```

## Detailed Requirements

1. Decode with `yaml.v3` using `KnownFields(true)` (unknown keys → finding
   `E_UNKNOWN_FIELD` with the yaml.Node line). Collect ALL findings, do not stop at the
   first.
2. Semantic checks (each with a stable code; DESIGN §7.3):
   - `E_VERSION` — missing or ≠ 1.
   - `E_BAD_ACTION` — action/default/on_opaque not in {allow, ask, deny}.
   - `E_BAD_ID` — id missing or not matching `^[a-z0-9][a-z0-9-]{0,63}$`.
   - `E_DUP_ID` — duplicate id within the file.
   - `E_EMPTY_MATCH` — rule without `match` or with zero set fields.
   - `E_BAD_GLOB` — program/args/any_arg/cwd/protected_paths pattern fails
     `doublestar.ValidatePattern`.
   - `E_BAD_REGEX` — `raw` fails `regexp.Compile`.
   - `E_ENFORCED_IN_PROJECT` — `enforced: true` while `Layer == "project"`.
   - `E_BAD_TEST` — test expect invalid, or `rule` referencing a non-existent id.
3. `~` expansion: apply to `cwd` and `protected_paths` patterns at load time using the
   process home (DESIGN §7.3). Literal `~` elsewhere is untouched.
4. `Load` returns `error` only for I/O-level impossibilities; schema problems are
   findings. `File` is returned (best-effort) even with findings, but callers must
   treat any finding as fatal for evaluation (exit 65) — document on the function.
5. CLI `agentgate policy validate [FILE…]`:
   - No args: `config.ResolvePaths` + `Discover` with the app cwd; validate whichever
     of global/project exist (absent files are fine: print `no policy files found` and
     exit 0). Explicit FILE args: validate each as layer `"project"` unless the path
     equals the resolved global file.
   - Output one line per finding: `PATH:LINE: CODE message`, sorted by path then line.
   - Exit 0 when zero findings, else 65.
6. Add dependencies: `gopkg.in/yaml.v3`, `github.com/bmatcuk/doublestar/v4`.
7. Tests: golden policy files under `internal/policy/testdata/` — one fully valid
   (exercising every field incl. tests block), plus one file per error code above;
   assert exact codes and line numbers.

## Acceptance Criteria

- [ ] Valid example from DESIGN §7 loads with zero findings.
- [ ] Every `E_*` code above is produced by exactly its trigger and includes a correct
      line number.
- [ ] `agentgate policy validate` exit codes: 0 (clean), 65 (findings), 65 (unreadable
      explicit file), 0 (no files discovered).
- [ ] Strict mode proven: a misspelled key (`defualt:`) yields `E_UNKNOWN_FIELD`.

## Validation

```sh
go test ./internal/policy/ -v
bin/agentgate policy validate internal/policy/testdata/valid.yaml; test $? -eq 0
bin/agentgate policy validate internal/policy/testdata/bad-glob.yaml; test $? -eq 65
```

## Dependencies

Issue 03 (config.Source, discovery for the no-arg form).

## Non-goals

Rule matching/evaluation (05, 08); `policy test` execution (18); `~` env-var expansion.

## Design References

DESIGN §7 (schema, §7.3 validation), §11.7 (validate CLI), §6 (discovery).
