# Issue 06: Shell parsing — command-unit extraction

## Title

Implement shell AST walk extracting command units, dynamic words, and hazards

## Summary

Implement the core of `internal/shellparse`: parse a raw bash string with
`mvdan.cc/sh/v3/syntax` and extract every `CommandUnit` that could execute, resolving
static words, marking dynamic words, extracting command/process substitutions as
first-class units, and emitting hazards — per DESIGN §8.1–§8.4.

## Context

This is the highest-risk component: the firewall's soundness is bounded by this walk.
DESIGN §8 enumerates the exact rules; ADR-002 fixes fail-closed behavior. Wrapper
unwrapping (§8.5) and redirects (§8.6) are follow-up issues — design the types now so
they extend cleanly.

## Scope

Package `internal/shellparse`. Public API (exact; fields for 07/09 included now):

```go
const DynamicSentinel = "\x00dyn"

type Redirect struct { Op string; Target string; Dynamic bool } // filled by issue 09
type CommandUnit struct {
    Argv      []string
    Dynamic   []bool
    Opaque    bool
    Origin    string // "top"|"cmdsubst"|"procsubst"|"nested-shell"|"wrapper"
    Redirects []Redirect
    Pos       string // "line:col" of the CallExpr, for traces
}
type Hazard struct { Kind, Detail string }
type Analysis struct { Units []CommandUnit; Hazards []Hazard }

func AnalyzeShell(raw string) Analysis        // never returns error: fail-closed inside
func AnalyzeVector(argv []string) Analysis    // trivial: one static unit, Origin "top"
```

## Detailed Requirements

1. **Parser**: `syntax.NewParser(syntax.Variant(syntax.LangBash))`. Parse error →
   `Analysis{Units: [one {Opaque: true, Origin: "top"}], Hazards: [{Kind:
   "parse-error", Detail: err.Error()}]}`.
2. **Walk** (DESIGN §8.2): emit a unit per `*syntax.CallExpr` with ≥ 1 word, reached
   through: stmt lists; `BinaryCmd` (`&&`,`||`,`|`,`|&`) both sides; `Subshell`;
   `Block`; `IfClause` (cond + then + else chains); `ForClause`, `WhileClause` (cond +
   body); `CaseClause` (all items); background stmts; `CmdSubst` (Origin
   `"cmdsubst"`); process substitutions `ProcSubst` (Origin `"procsubst"`). Use
   explicit recursion over node types (not blind `syntax.Walk`) so every handled node
   is deliberate; default branch for unhandled compound nodes: mark input-level hazard
   `{Kind: "parse-error", Detail: "unhandled node <type>"}` and add one opaque unit —
   fail-closed for future bash constructs (coproc etc. may be added later).
3. **Opaque-scope constructs** (DESIGN §8.2 table): `FuncDecl` → hazard `funcdecl` +
   one opaque unit (do not walk the body); `alias`/`source`/`.`/`trap`-with-command →
   corresponding hazard + opaque unit; `eval`:
   - all args static → join with spaces, re-parse recursively (depth counter; > 4 →
     opaque + `depth-limit`), units get Origin `"nested-shell"`;
   - any non-static arg → hazard `eval-dynamic` + opaque unit.
4. **Word resolution** (DESIGN §8.4): implement
   `func staticWord(w *syntax.Word) (val string, ok bool)` — Lit, SglQuoted,
   DblQuoted-of-lits concatenation; anything containing `ParamExp`, `CmdSubst`,
   `ArithmExp`, `ExtGlob`, `ProcSubst` → not static. Tilde: a leading `~` or `~/` in an
   otherwise-static word expands to `os.UserHomeDir()`; `~user` forms → not static.
   Program position non-static → unit `Opaque: true` + hazard `dynamic-program`
   (still emit the unit; its CmdSubsts were already extracted separately). Argument
   position non-static → `Argv[i] = DynamicSentinel`, `Dynamic[i] = true`, hazard
   `dynamic-arg` (input-level, deduplicated by Kind).
5. **Assignments** (DESIGN §8.3): `CallExpr` with only assignments → no unit; hazard
   none. Assignment values: walk them for `CmdSubst` (those become units). Prefix
   assignments before a command: hazard `env-prefix`, command analyzed normally.
6. **Comments / empty input**: zero units, zero hazards (engine renders `no-exec`,
   DESIGN §9.3).
7. `AnalyzeVector`: `Argv = argv`, all static, no hazards. Empty argv → zero units.
8. **Hazard kinds** emitted by this issue (closed set, DESIGN §8.7 subset):
   `parse-error`, `dynamic-program`, `dynamic-arg`, `env-prefix`, `funcdecl`, `alias`,
   `eval-dynamic`, `source`, `trap`, `depth-limit`. Kinds are string constants in
   `hazards.go` (one const block — issues 07/09 append theirs).
9. Add dependency `mvdan.cc/sh/v3`.
10. **Tests** (table-driven; input → expected units (argv/origin/opaque) + hazard
    kinds). Minimum rows:
    - `git status` → 1 unit.
    - `git add . && git commit -m 'x'; ls | wc -l` → 4 units.
    - `(cd /tmp && rm -rf y) || echo no` → 3 units.
    - `if test -f x; then rm x; else touch x; fi` → 3 units (test, rm, touch).
    - `for f in a b; do cat "$f"; done` → 1 unit `cat <dyn>`.
    - `echo $(rm -rf ~) done` → units: echo(with dyn arg) + rm(cmdsubst, argv
      `rm -rf /home/USER` or `~`-expanded — assert expansion), hazard `dynamic-arg`.
    - `` echo `whoami` `` → whoami unit (backticks = CmdSubst).
    - `diff <(sort a) <(sort b)` → 3 units, procsubst origins.
    - `$CC -o x x.c` → opaque unit + `dynamic-program`.
    - `FOO=bar make` → make unit + `env-prefix`; `FOO=$(date)` alone → date unit only.
    - `eval 'rm -rf /tmp/x'` → rm unit origin nested-shell; `eval "$X"` → opaque +
      `eval-dynamic`.
    - `f(){ rm -rf ~; }` → opaque + `funcdecl`.
    - `# comment` → zero units.
    - `r'm' -rf x` (quote splicing) → argv[0] == `rm` (static concat).
    - unparseable `if then fi (` → opaque + parse-error.

## Acceptance Criteria

- [ ] All listed test rows implemented and passing; package coverage ≥ 90%.
- [ ] Every §8.7 hazard kind this issue owns has ≥ 1 test.
- [ ] No use of `sh/v3/interp` or `sh/v3/expand` (parse-only; ADR-001).
- [ ] Unhandled-node fail-closed path proven with a synthetic AST test or a construct
      not in the walk list.

## Validation

```sh
go test ./internal/shellparse/ -v -cover
make lint
```

## Dependencies

Issue 01. Parallel with 03/04/05.

## Non-goals

Wrapper unwrapping and nested `sh -c` (07); redirect extraction (09); policy decisions
(08).

## Design References

DESIGN §8.1–§8.4, §8.7; ADR-002 (fail-closed rationale).
