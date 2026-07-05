# Issue 05: Matching primitives

## Title

Implement match primitives: program, positional args, any_arg, cwd, raw

## Summary

Implement `internal/match`: the five matcher functions the engine composes, with the
exact semantics of DESIGN §7.1–§7.2, including dynamic-argument sentinels.

## Context

Matching must be boringly predictable — users reason about their policies with these
rules. All glob behavior comes from doublestar; this package only encodes agentgate's
composition semantics.

## Scope

Package `internal/match` only. Public API (exact):

```go
const DynamicSentinel = "\x00dyn" // mirrors shellparse (issue 06); duplicated const, single meaning

func Program(pattern, program string) bool
func Args(patterns []string, args []string, dynamic []bool) bool
func AnyArg(pattern string, args []string, dynamic []bool) bool
func Cwd(pattern, absCwd string) bool
func Raw(re *regexp.Regexp, raw string) bool
```

## Detailed Requirements

1. `Program` (DESIGN §7.1): if `pattern` contains `/` → match against
   `path.Clean(program)`; else match against `filepath.Base(program)`. doublestar
   `Match`, case-sensitive. Empty pattern → true (field absent).
2. `Args` (DESIGN §7.2): positional; the literal final element `"**"` consumes zero or
   more remaining args (content-independent — matches even dynamic args). Without
   trailing `"**"`: length must be equal. A dynamic arg (`dynamic[i]==true`) fails every
   pattern except the trailing rest-`"**"`. `patterns == nil` → true.
3. `AnyArg`: true if ≥ 1 non-dynamic arg matches the glob. `pattern == ""` → true.
4. `Cwd`: doublestar match of the pattern (already `~`-expanded by issue 04) against
   the cleaned absolute cwd. Also match when the pattern equals the cwd exactly
   (doublestar handles literals). Empty pattern → true.
5. `Raw`: `re == nil` → true; else `re.MatchString(raw)` where `raw` is the original
   input string (shell mode) or the space-joined argv (vector mode) — the caller
   provides it; this function is trivial but exists for symmetry/testing.
6. Errors: patterns are pre-validated (issue 04); on `doublestar.Match` error return
   false (and this can only happen with an invalid pattern — add a debug assertion in
   tests, not a panic in prod).
7. Table-driven tests — minimum cases (each row: pattern(s), input, want):
   - Program: `git`/`git`→T; `git`/`/usr/bin/git`→T; `g?t`/`git`→T; `git`/`gitk`→F;
     `/usr/bin/*`/`/usr/bin/git`→T; `/usr/bin/*`/`git`→F; `bin/git`(cleaned) cases.
   - Args exact: `["status"]` vs `["status"]`→T, vs `["status","-s"]`→F, vs `[]`→F.
   - Args nil-vs-empty (CRITICAL — the starter `deny-bare-shell` rule depends on it):
     `nil` vs anything→T; empty non-nil `[]` vs `[]`→T; `[]` vs `["x"]`→F. yaml.v3
     preserves the distinction (absent field → nil slice; `args: []` → empty slice) —
     add a decode assertion for both forms in issue-04 fixtures too (note there).
   - Args rest: `["add","**"]` vs `["add"]`→T, vs `["add","a","b"]`→T, vs `["rm"]`→F.
   - `"**"` mid-list is a normal glob: `["**","x"]` vs `["anything","x"]`→T (single
     segment glob semantics), vs `["a/b","x"]` — document doublestar behavior in test.
   - Dynamic: `["status"]` vs `[dyn]`→F; `["add","**"]` vs `["add",dyn]`→T;
     AnyArg `--force*` vs `[dyn]`→F.
   - Cwd: `~/dev/**` (pre-expanded) vs `/Users/u/dev/x/y`→T; vs `/Users/u/other`→F;
     exact dir equality.

## Acceptance Criteria

- [ ] API exactly as scoped; no engine imports (pure leaf package).
- [ ] All listed test rows present and passing; coverage ≥ 95% for the package.
- [ ] Dependency added: doublestar only (shared with issue 04).

## Validation

```sh
go test ./internal/match/ -v -cover
```

## Dependencies

Issue 01 (skeleton). Parallel with 03/04/06.

## Non-goals

Rule composition/first-match (08); parsing argv out of strings (06); PATH resolution
(explicit v1 non-goal, DESIGN §7.1).

## Design References

DESIGN §7.1 (program), §7.2 (args/dynamic), §8.4 (sentinel), §9.2 (how the engine
composes these).
