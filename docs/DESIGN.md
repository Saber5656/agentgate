# agentgate — v1 Design (Canonical)

Status: canonical design for v1. All issues in `docs/ISSUE_PLAN.md` derive from this document.
When this document and any GitHub Issue disagree, this document wins; update it first.

Section numbers (§N) are stable reference anchors used by `docs/issues/*.md`.

---

## §1 Product overview

agentgate is a **local tool-execution firewall for AI agents**. It sits between an agent
(Claude Code, Codex CLI, or any harness that can call an external command) and the user's
machine, and decides — per shell command — whether the command is **allowed**, must be
**asked** (escalated to the human), or is **denied**, based on a user-owned policy file.

Three pillars (from the repository README):

| Pillar | v1 realization |
|---|---|
| Allowlist | Layered YAML policy (global + project) with allow / ask / deny rules, first-match, fail-closed defaults |
| Dry-run | `agentgate check` / `agentgate exec --dry-run` / `agentgate check --explain` evaluate without executing and show the full match trace |
| Audit log | Append-only JSONL with per-record SHA-256 hash chain, `agentgate log` query and `agentgate log verify` |

agentgate is a **decision engine, not a sandbox**. It statically analyzes commands and
decides; it does not contain, trace, or isolate the process at runtime (see §3 Guarantees).

### v1 goal statement

A single Go binary `agentgate` that:

1. evaluates shell command strings against a layered YAML policy (`check`),
2. wraps execution with that evaluation (`exec`),
3. plugs into Claude Code as a `PreToolUse` hook for the `Bash` tool (`hook claude-code`),
4. records every evaluation and execution in a tamper-evident audit log (`log`),
5. ships with an installer for a curated starter policy (`init`) and policy tooling
   (`policy validate`, `policy test`).

---

## §2 Users, integration surfaces, and product shape

Primary user: a developer running AI coding agents locally who wants (a) hard limits on
what agents can execute, (b) a human-in-the-loop escalation path, (c) a reviewable record.

v1 integration surfaces, in order of importance:

1. **Claude Code PreToolUse hook** (`agentgate hook claude-code`) — automatic interception
   of every `Bash` tool call. This is the flagship integration.
2. **Generic JSON check API** (`agentgate check --json`, stdin input mode) — any harness
   that can run a subprocess and parse JSON can integrate.
3. **Exec wrapper** (`agentgate exec -- CMD…` / `agentgate exec --shell 'STRING'`) — for
   harnesses that route command execution through a configurable wrapper, and for humans
   testing policies.

Not an integration surface in v1: PATH shims, MCP proxying, LD_PRELOAD/dtrace tricks,
Codex-specific adapters (see §16 deferred).

---

## §3 Threat model and guarantees

### What agentgate defends against (v1)

- An agent issuing a destructive or out-of-policy command **in good faith or by error**
  (wrong cwd, overly broad `rm`, `git push --force`, package publishing, `sudo`).
- An agent **trivially obfuscating** a command with shell syntax: pipelines, `&&`/`;`
  chains, subshells, command substitution, `sh -c '…'` nesting, `env`/`nohup`/`timeout`
  wrappers, writing to protected paths via redirects.
- Loss of accountability: every decision is written to a hash-chained audit log.

### What agentgate does NOT defend against (documented honestly)

- A **deliberately adversarial** agent with an allow-listed interpreter: if policy allows
  `python`, `node`, `make`, or any compiler/test runner, arbitrary code runs. Policy
  authoring guidance must say this loudly.
- Runtime divergence: agentgate decides **statically before execution**. It cannot see
  what an allowed binary actually does (TOCTOU, symlinks, `$PATH` swaps).
- An attacker with the same OS user privileges: they can edit policy or rewrite the audit
  log wholesale. The hash chain makes tampering **evident** (verify fails / chain breaks),
  not impossible. See ADR-003.
- Claude Code tools other than `Bash` (e.g. `Write`, `Edit`) in v1 — the hook passes them
  through with no opinion (§13). v2 candidate.

### Fail-closed principles (ADR-002)

1. Anything the analyzer cannot statically resolve is **opaque** and gets the configured
   `on_opaque` action (built-in default: `deny`).
2. The decision for a multi-command input is the **most restrictive** decision across all
   analyzed command units (`deny` > `ask` > `allow`).
3. Internal errors never produce a silent `allow`. In hook mode, engine/config errors map
   to `ask` with the error as the reason (human sees and decides); in `check`/`exec` they
   map to a non-zero error exit, never to execution.

---

## §4 Architecture

```
                 ┌────────────────────────────────────────────────┐
                 │                agentgate binary                │
 stdin JSON ───▶ │ adapters/claudecode ─┐                         │
 argv/flags ───▶ │ cli (cobra)          ├─▶ engine ─▶ decision ───┼─▶ stdout (human/JSON)
                 │                      │     │                   │
                 │        policy loader ┘     │                   │
                 │  (global + project YAML)   ▼                   │
                 │                      shellparse                │
                 │                (mvdan.cc/sh AST → units)       │
                 │                            │                   │
                 │                            ▼                   │
                 │                        audit writer ───────────┼─▶ ~/.agentgate/logs/*.jsonl
                 └────────────────────────────────────────────────┘
```

Component responsibilities:

| Component | Package | Responsibility |
|---|---|---|
| CLI | `internal/cli` | cobra commands, flag parsing, output rendering, exit codes |
| Config | `internal/config` | home dir resolution, policy file discovery, layer assembly |
| Policy | `internal/policy` | YAML schema, strict decode, validation, layer model |
| Shell analysis | `internal/shellparse` | raw string → `[]CommandUnit` + hazards (mvdan AST walk, wrapper unwrap, nested shells, redirects) |
| Matching | `internal/match` | glob/regex primitives for program/args/cwd/raw |
| Engine | `internal/engine` | rule evaluation, layering, aggregation, decision+trace |
| Audit | `internal/audit` | JSONL writer, hash chain, rotation, query, verify |
| Claude Code adapter | `internal/adapters/claudecode` | PreToolUse JSON in → permissionDecision JSON out |

Data flow for every entry point is identical: *input → shellparse → engine (policy) →
decision → (render | execute) → audit append*.

---

## §5 Repository layout and toolchain (decision: Go — ADR-001)

- Language: **Go ≥ 1.22**. Module path: `github.com/Saber5656/agentgate`.
- Dependencies (pinned in `go.mod`; no others without a new ADR):
  - `github.com/spf13/cobra` — CLI framework
  - `mvdan.cc/sh/v3` — POSIX/bash parser (`syntax` package only; not `interp`)
  - `github.com/bmatcuk/doublestar/v4` — glob matching
  - `gopkg.in/yaml.v3` — policy parsing (strict mode)
  - `github.com/gofrs/flock` — audit log file locking
  - `github.com/rogpeppe/go-internal/testscript` — CLI end-to-end tests (test-only)
- Layout:

```
cmd/agentgate/main.go        # thin main; calls internal/cli.Execute()
internal/cli/                # one file per subcommand: root.go check.go exec.go hook.go
                             #   log.go policy.go init.go version.go
internal/config/
internal/policy/
internal/shellparse/
internal/match/
internal/engine/
internal/audit/
internal/adapters/claudecode/
internal/version/            # Version/Commit/Date vars set via -ldflags
examples/                    # example policies (user docs)
testdata/corpus/             # adversarial bypass corpus (§15)
docs/                        # this design, issue plan, ADRs
.github/workflows/           # ci.yml, release.yml
Makefile                     # build, test, lint, fmt targets
.golangci.yml
```

- Build: `make build` → `bin/agentgate` with `-ldflags "-X …/internal/version.Version=…"`.
- Lint: `golangci-lint` with `govet`, `staticcheck`, `errcheck`, `gofumpt` enabled.
- Support targets: macOS (arm64/amd64), Linux (arm64/amd64). Windows is out of scope (§16).

---

## §6 Runtime file & directory layout

All state lives under a single home directory, resolved in this order:

1. `$AGENTGATE_HOME` if set (absolute path required; relative → config error)
2. `~/.agentgate`

```
$AGENTGATE_HOME/
  policy.yaml          # global policy (layer: global)
  logs/
    audit-YYYYMMDD.jsonl   # UTC-dated audit segments (§12)
    audit.lock             # flock file for writers
```

Project policy discovery: starting at the evaluation cwd, walk up parent directories
until filesystem root; the first directory containing `.agentgate/policy.yaml` provides
the **project** layer. Search stops at the first hit (no multi-project stacking). The
walk does **not** cross into other users' home dirs — it simply stops at `/`.

Both files are optional. No global and no project policy → engine still runs with
built-in defaults (`default: deny`, `on_opaque: deny`, empty rule set) — fail-closed
out of the box; `agentgate init` installs a friendlier curated global policy (§11.6).

Permissions: `init` creates `$AGENTGATE_HOME` with `0700` and `policy.yaml` with `0600`.
Audit segments are created `0600`.

---

## §7 Policy schema reference (YAML)

Top-level document (strict decoding — unknown keys are errors):

```yaml
version: 1            # required; int; only 1 accepted
default: ask          # optional; allow|ask|deny; see §9 layering for resolution
on_opaque: ask        # optional; allow|ask|deny; action for opaque units (§8.5)
protected_paths:      # optional; list of glob patterns (union across layers, §9)
  - "~/.ssh/**"
rules:                # optional; ordered list; order = priority within a layer
  - id: git-readonly          # required; unique within file; ^[a-z0-9][a-z0-9-]{0,63}$
    action: allow             # required; allow|ask|deny
    reason: "read-only git"   # optional; free text shown in decisions/logs
    enforced: true            # optional; bool; ONLY valid in the global policy file
    match:                    # required; at least one field; all present fields AND-ed
      program: "git"          # glob; see §7.1
      args: ["status", "**"]  # positional arg globs; see §7.2
      any_arg: "--force*"     # glob that must match ≥1 argument
      cwd: "~/dev/**"         # glob on the absolute evaluation cwd
      raw: "(?i)curl .*\\| *sh"  # RE2 regex on the raw input string (escape hatch)
tests:                # optional; consumed by `agentgate policy test` (§11.8)
  - name: "git status allowed"
    cmd: "git status"
    expect: allow             # allow|ask|deny
    rule: git-readonly        # optional; expected deciding rule id
```

### §7.1 `program` matching

- The pattern is a doublestar glob, matched case-sensitively.
- If the pattern contains `/`: match against the unit's program string as written
  (after quote removal), path-cleaned (`path.Clean`).
- Otherwise: match against `filepath.Base(program)`.
- No `$PATH` resolution, no symlink resolution in v1 (documented limitation, §3).

### §7.2 `args` matching (positional)

- `args` is a list of doublestar globs matched against the unit's arguments
  (`argv[1:]`), position by position.
- The literal pattern `"**"` is special **only as the final list element**: it matches
  zero or more remaining arguments. Anywhere else, `**` in a pattern is just a glob.
- Without a trailing `"**"`, the argument count must equal `len(args)` exactly.
- A *dynamic* argument (contains unresolved expansion, §8.4) fails every glob except a
  trailing `"**"` rest-matcher (which matches by count, not content).

### §7.3 Semantic validation rules (enforced by `policy validate`, §11.7)

- `version` present and `== 1`.
- Rule `id` unique within the file; matches the regex above.
- `action`, `default`, `on_opaque` ∈ {allow, ask, deny}.
- Every rule has `match` with ≥ 1 field; every glob compiles under doublestar; every
  `raw` compiles under Go `regexp` (RE2).
- `enforced: true` in a **project** policy file → validation error E_ENFORCED_IN_PROJECT.
- `tests[].expect` valid action; `tests[].rule` (if set) references an existing rule id
  in the same file.
- `~/` prefix in `cwd`, `protected_paths` patterns is expanded to the user home at load
  time. No environment variable expansion anywhere in v1.

---

## §8 Command analysis pipeline (`internal/shellparse`)

Input: a raw command string (shell mode) or an argv vector (vector mode, from
`exec -- CMD…`). Output: `Analysis{Units []CommandUnit, Hazards []Hazard}`.

```go
type CommandUnit struct {
    Argv    []string  // static argv; dynamic words → sentinel (§8.4)
    Dynamic []bool    // per-argv element: true if value unresolved
    Opaque  bool      // program position unresolvable, or unit is an opaque construct
    Origin  string    // "top" | "cmdsubst" | "procsubst" | "nested-shell" | "wrapper"
    Redirects []Redirect // write redirects with static/dynamic target (§8.6)
}
type Hazard struct { Kind string; Detail string } // see table §8.7
```

### §8.1 Parsing

Parse with `mvdan.cc/sh/v3/syntax`, `syntax.LangBash` variant. A parse **error** makes
the whole input one opaque unit (`Hazard{Kind: "parse-error"}`) — fail-closed.

### §8.2 Unit extraction — walk rules

Extract a `CommandUnit` for every `*syntax.CallExpr` reached by walking:

- statement lists (`;`, newline), binary commands (`&&`, `||`, `|`, `|&`),
- subshells `(…)`, blocks `{ …; }`,
- `if`/`elif`/`else`, `for`, `while`/`until`, `case` — **all** branches and bodies
  (static analysis takes the union of everything that could run),
- background statements (`… &`), coprocesses,
- command substitutions `$(…)` and backticks — inner statements are extracted as
  first-class units with `Origin: "cmdsubst"` (they DO execute),
- process substitutions `<(…)` / `>(…)` — same, `Origin: "procsubst"`.

Constructs that make the **enclosing scope** opaque (one opaque unit, matching hazard):

| Construct | Hazard.Kind |
|---|---|
| function declaration (`f() { … }`) | `funcdecl` |
| `alias` builtin | `alias` |
| `eval` with any non-literal argument | `eval-dynamic` |
| `source` / `.` | `source` |
| `trap` with command string | `trap` |

`eval` with a single **literal** string argument: parse the string recursively (depth
limit 4; beyond → opaque, hazard `depth-limit`).

### §8.3 Assignments

- Pure assignment statements (`FOO=bar`) with **literal** values and no command: no unit
  (nothing executes), no hazard.
- Assignment values containing `$(…)`: the substitution is extracted as a unit (§8.2).
- Prefix assignments (`FOO=bar cmd …`): analyze `cmd …` normally; the assignment itself
  is ignored for matching (v1 does not model env), hazard `env-prefix` recorded.

### §8.4 Word resolution: static vs dynamic

A word is **static** if it is composed only of literals, single-quoted strings, and
double-quoted strings containing only literals. Its value is the quote-removed text.

A word containing parameter expansion (`$X`, `${X}`), arithmetic (`$((…))`), command or
process substitution, globs are fine (globs stay literal text), tilde is expanded to the
current user home. Non-static words:

- in **program position** → `Opaque: true` for the unit (program unknowable),
  hazard `dynamic-program`;
- in **argument position** → that argv element is the sentinel `"\x00dyn"` with
  `Dynamic[i] = true`; the unit stays matchable by `program`/`cwd` (§7.2 defines how
  arg patterns treat it); hazard `dynamic-arg`.

### §8.5 Wrapper unwrapping and nested shells

Before matching, each unit is unwrapped through a built-in table (applied repeatedly,
max 4 layers, then opaque with hazard `depth-limit`):

| Program | Handling |
|---|---|
| `env` | skip leading `-i`/`-u NAME`/`NAME=val` operands; rest is the real unit. `env` alone → plain unit |
| `command`, `exec` | skip flags (`-p`, `-v`, `-V`, `-l`, `-a name`); rest is the real unit |
| `nohup`, `nice` (with optional `-n N`), `time`, `stdbuf` (skip `-i/-o/-e VAL`), `timeout` (skip flags + DURATION operand) | rest is the real unit |
| `sudo`, `doas` | unit for the wrapper itself **and** unit for the wrapped command (both evaluated; deny-sudo rules match the wrapper unit) |
| `sh`, `bash`, `zsh`, `dash`, `ksh` with `-c STRING` (STRING literal) | parse STRING recursively as bash; non-literal STRING → opaque, hazard `nested-shell-dynamic` |
| `sh`/`bash`/… with a script *file* operand | opaque, hazard `shell-script-file` |
| `xargs` | opaque, hazard `xargs` (constructed argv unknowable) |
| `find` with `-exec`/`-execdir`/`-ok`/`-okdir`/`-delete` | keep the `find` unit AND emit an extra unit for the `-exec` command template (`{}` → dynamic sentinel); `-delete` → hazard `find-delete` |

Unwrapped units keep `Origin: "wrapper"` ancestry in the trace (§10).

### §8.6 Redirect extraction

For every unit, collect write-capable redirects: `>`, `>>`, `<>`, `>|`, `&>`, `&>>`.
Static target word → cleaned absolute path (relative targets joined to the evaluation
cwd; `~` expanded). Dynamic target → `Redirect{Dynamic: true}`, hazard `dynamic-redirect`.
Read redirects (`<`, `<<`, `<<<`) are ignored in v1.

### §8.7 Hazard kinds (closed set for v1)

`parse-error`, `dynamic-program`, `dynamic-arg`, `env-prefix`, `funcdecl`, `alias`,
`eval-dynamic`, `source`, `trap`, `nested-shell-dynamic`, `shell-script-file`, `xargs`,
`find-delete`, `dynamic-redirect`, `depth-limit`, `protected-path-write` (§9.4).

---

## §9 Evaluation semantics (`internal/engine`) — ADR-002

### §9.1 Layer assembly

Two layers may exist: **global** (`$AGENTGATE_HOME/policy.yaml`) and **project**
(nearest `.agentgate/policy.yaml`, §6). Effective evaluation order per unit:

1. global rules with `enforced: true`, in file order
2. project rules, in file order
3. global rules without `enforced`, in file order

**First match wins** within this combined order. If nothing matches:

4. `default` of the project file if set, else `default` of the global file if set,
   else built-in `deny`.

`on_opaque` resolves the same way (project → global → built-in `deny`).
`protected_paths` is the **union** of built-in defaults (§9.4) + global + project.

### §9.2 Per-unit decision

```
if unit.Opaque            → on_opaque
else                      → first-matching-rule action, else layered default
then                      → escalate with hazard overlays (§9.4)
```

Rule matching per §7: all present `match` fields must match. Sub-units produced by
substitution/wrapper expansion are evaluated **identically and independently**.

### §9.3 Aggregation

Severity order: `deny(3) > ask(2) > allow(1)`. The input's final decision is the
**maximum severity across all units**. Zero units (empty input, comments only,
pure literal assignments) → `allow` with note `no-exec`.

### §9.4 Built-in hazard overlays

These act after rule matching, and can only **escalate**, never downgrade:

| Condition | Effect |
|---|---|
| Write redirect (static target) matching any `protected_paths` glob | unit decision := `deny`, hazard `protected-path-write` |
| Write-capable program (`cp`,`mv`,`rm`,`tee`,`dd`,`chmod`,`chown`,`ln`,`truncate`,`rsync`,`install`) with a **path-looking** static argument (starts with `/`, `./`, `../`, or `~`) that, resolved against cwd, matches `protected_paths` | unit decision := `deny`, hazard `protected-path-write` |
| Dynamic redirect target | unit decision := max(unit, `on_opaque`) |

Built-in default `protected_paths` (always present, cannot be removed in v1):
`~/.agentgate/**` (self-protection). All other protections come from policy files
(the starter policy ships a curated list, §11.6).

### §9.5 Decision object

Every evaluation produces one `Decision`:

```go
type Decision struct {
    Final    Action        // allow|ask|deny
    Units    []UnitResult  // per-unit: argv, origin, action, matched rule (id, layer, action, reason), opaque, hazards
    Hazards  []Hazard      // input-level union
    Policy   PolicyMeta    // per layer: path, sha256 of file bytes; empty if absent
    Duration time.Duration
}
```

This object is the single source for: human rendering, `--json` output (§10), audit
records (§12), and the hook adapter (§13).

---

## §10 Decision JSON (stable output contract)

`agentgate check --json` (and `exec --dry-run --json`) emit exactly one JSON object on
stdout, schema id `agentgate.decision/v1`:

```json
{
  "schema": "agentgate.decision/v1",
  "final": "deny",
  "input": {"raw": "git push && rm -rf /", "mode": "shell", "cwd": "/Users/u/dev/x"},
  "units": [
    {"argv": ["git", "push"], "origin": "top", "action": "ask",
     "rule": {"id": "git-write", "layer": "project", "action": "ask", "reason": "pushes need a human"},
     "opaque": false, "hazards": []},
    {"argv": ["rm", "-rf", "/"], "origin": "top", "action": "deny",
     "rule": {"id": "deny-rm-outside", "layer": "global", "action": "deny", "reason": ""},
     "opaque": false, "hazards": []}
  ],
  "hazards": [],
  "policy": {
    "global": {"path": "/Users/u/.agentgate/policy.yaml", "sha256": "ab12…"},
    "project": {"path": "/Users/u/dev/x/.agentgate/policy.yaml", "sha256": "cd34…"}
  },
  "duration_ms": 3
}
```

- `rule` is `null` when the layered default decided. Then `"decided_by": "default"` is
  present on the unit; likewise `"decided_by": "on_opaque"` / `"hazard"` / `"rule"`.
- Field additions are allowed within v1; renames/removals require bumping the schema id.

---

## §11 CLI reference

Global flags: `--home DIR` (override `$AGENTGATE_HOME`), `--cwd DIR` (evaluation cwd,
default: process cwd), `-q/--quiet`.

### §11.1 Exit codes (whole binary, stable contract)

| Code | Meaning |
|---|---|
| 0 | success / decision = allow |
| 10 | decision = ask (check mode; or exec-mode prompt declined: 21) |
| 20 | decision = deny |
| 21 | exec: ask was prompted and the human declined |
| 64 | usage error (bad flags/args) |
| 65 | config/policy error (unreadable, invalid schema, semantic errors) |
| 70 | internal error |
| else | exec: allowed child's exit code, passed through verbatim |

### §11.2 `agentgate check [--shell STRING | -- CMD ARGS…] [--json] [--explain]`

Evaluate, never execute. Input modes: `--shell STRING` (shell parse), `-- CMD ARGS…`
(vector mode: single unit, no shell parsing), or **stdin** (no args → read one raw
command line… stdin mode reads the entire stdin as one shell string). Human output: one
line per unit (`DECISION  rule-id  argv-preview`) + final line `final: DECISION`.
`--explain` adds the full trace: layer order, each rule tried → matched/why-not (first
20 misses), hazards, defaults consulted. Writes an audit record with `source: "check"`.

### §11.3 `agentgate exec [--shell STRING | -- CMD ARGS…] [--dry-run] [--json] [--yes]`

Evaluate, then:

- `allow` → execute; **vector mode** runs `exec.Command(argv[0], argv[1:]…)` directly;
  **shell mode** runs `/bin/sh -c 'STRING'` (documented: evaluation parsed bash; execution
  uses sh) with inherited stdio/env/cwd; child exit code passed through; SIGINT/SIGTERM
  forwarded to the child.
- `ask` → if stdout+stdin are a TTY: print the unit table + reasons, prompt
  `Execute? [y/N]`; `y` → execute (audit `ask_resolved: "approved"`), else exit 21.
  Non-TTY: **do not execute**, exit 10 (fail-closed), unless `--yes` was given (which
  auto-approves `ask` — never `deny`).
- `deny` → print reasons, exit 20.
- `--dry-run` → identical output/decision path, never executes, audit `source: "exec-dry"`.

Every path writes an audit record (`source: "exec"`) including execution result when run.

### §11.4 `agentgate hook claude-code` — see §13.

### §11.5 `agentgate log …`

- `agentgate log show [--since DUR|RFC3339] [--decision allow|ask|deny] [--source S] [--limit N] [--json]`
  — newest-last table (ts, source, decision, rule, argv preview); `--json` = raw JSONL lines.
- `agentgate log tail [-f]` — stream new records (poll, 500ms).
- `agentgate log verify [--all|FILE…]` — recompute the hash chain (§12.3); exit 0 clean,
  20 broken (report first broken seq + file), 65 unreadable.

### §11.6 `agentgate init [--force]`

Create `$AGENTGATE_HOME` (0700), `logs/` and install the **starter global policy**
(0600) if absent. With existing `policy.yaml`: no overwrite unless `--force` (then the
old file is renamed `policy.yaml.bak-<unixts>` first). Prints the Claude Code settings
snippet (§13.4) at the end. Idempotent: re-running without `--force` changes nothing and
exits 0. The starter policy content is a versioned asset embedded via `go:embed`; it is
specified normatively in `docs/issues/14-init-and-starter-policy.md` and mirrored in
`examples/starter-policy.yaml`; its shape: `default: ask`, `on_opaque: ask`, curated
protected paths (`~/.ssh/**`, `~/.aws/**`, `~/.gnupg/**`, `~/.claude/**`, `~/.codex/**`,
`**/.env`, `**/.env.*`, `**/*.pem`, `**/id_rsa*`, `**/id_ed25519*`), enforced denies
(`sudo`, `doas`, bare `sh|bash|zsh` with empty args — the pipe-to-shell approximation),
read-only allows (`ls cat head tail wc stat file pwd echo which rg grep fd`), git
read-only allows, `git push --force*` deny, and embedded `tests:` for all of the above.

### §11.7 `agentgate policy validate [FILE…]`

No args: validate the discovered global + project files. Structural (strict YAML decode)
+ semantic checks per §7.3. Output: one line per finding `FILE:LINE: CODE message`;
exit 0/65.

### §11.8 `agentgate policy test [FILE…]`

Run the `tests:` block of each policy file **against the full layered engine as seen
from that file's layer context** (global file: global-only; project file: project+global).
Output: TAP-like `ok 1 - name` / `not ok 2 - name (expected deny, got ask; rule=…)`;
exit 0 all pass, 1 any fail, 65 config error.

### §11.9 `agentgate version`

Prints `agentgate VERSION (COMMIT, DATE, goVERSION, GOOS/GOARCH)`.

---

## §12 Audit log specification — ADR-003

### §12.1 Records

One JSON object per line (JSONL), UTF-8, no pretty-printing. Schema id
`agentgate.audit/v1`. Field order is fixed by the Go struct (deterministic marshal):

```json
{"schema":"agentgate.audit/v1","seq":42,"ts":"2026-07-05T12:00:00.123456Z",
 "source":"exec","session":"9f0c…","agent":"claude-code",
 "cwd":"/Users/u/dev/x","raw":"git push","mode":"shell",
 "final":"ask","units":[{"argv":["git","push"],"action":"ask","rule_id":"git-write","layer":"project","opaque":false}],
 "hazards":[],"policy_sha":{"global":"ab12…","project":"cd34…"},
 "exec":{"ran":true,"exit_code":0,"duration_ms":812,"ask_resolved":"approved"},
 "prev":"<64 hex>","hash":"<64 hex>"}
```

- `source` ∈ {`check`, `exec`, `exec-dry`, `hook.claude-code`, `policy-test`(not logged),
  … } — closed set per issue specs. `policy test`/`validate`/`log` do **not** write audit
  records; `check`, `exec`, and the hook always do.
- `session`: UUIDv4 generated per process; hook mode uses Claude Code's `session_id`.
- `exec` is `null` for non-exec sources.

### §12.2 Segments, ordering, locking

- Directory `$AGENTGATE_HOME/logs/`, file per UTC day: `audit-YYYYMMDD.jsonl`.
- Writers take an exclusive `flock` on `logs/audit.lock` around
  *read-last-line → build record → append+fsync → release*. Concurrent hook processes
  are the normal case; the lock serializes them. `seq` restarts at 1 per segment file.
- Append is a single `write(2)` of the full line + `\n`, then `fsync`.

### §12.3 Hash chain

- `prev` = `hash` of the previous record in the same segment; the **first record** of a
  segment carries `prev` = `hash` of the **last record of the newest older segment**
  (lexicographic filename order), or 64 zeros if none exists.
- `hash` = lowercase hex SHA-256 of: `prev` + `"\n"` + the exact serialized record bytes
  **with the `hash` field set to the empty string**. Writers serialize once with
  `hash:""`, compute, then substitute; verifiers re-serialize the parsed record with
  `hash:""` using the same struct and compare. This makes canonicalization = "the Go
  struct's `encoding/json` output", which is deterministic for a fixed struct.
- `log verify` walks segments oldest→newest, re-deriving every hash and the cross-file
  link. Guarantee: **tamper-evident**, not tamper-proof (§3, ADR-003).

---

## §13 Claude Code hook adapter

### §13.1 Contract

`agentgate hook claude-code` reads **one** PreToolUse JSON event from stdin. Relevant
input fields (others ignored): `session_id`, `cwd`, `hook_event_name`, `tool_name`,
`tool_input.command`.

### §13.2 Behavior

| Case | stdout | exit |
|---|---|---|
| `tool_name != "Bash"` | `{}` (no opinion) | 0 |
| `Bash`, decision allow/ask/deny | `{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"allow|ask|deny","permissionDecisionReason":"<final>: rule <id> — <reason>; hazards: …"}}` | 0 |
| policy/config unreadable or invalid | permissionDecision `ask`, reason = the error | 0 |
| internal panic/error | permissionDecision `ask`, reason = the error | 0 |

Never exit non-zero for decisions (a non-zero hook exit has harness-defined semantics we
do not rely on); never output `allow` on any error path (fail-closed, ADR-002). Total
budget: < 2s; the adapter sets a 1500ms internal timeout on evaluation, timeout → `ask`.

Evaluation cwd = the event's `cwd`. Audit record: `source: "hook.claude-code"`,
`agent: "claude-code"`, `session` = `session_id`.

### §13.3 Schema-drift isolation

All Claude Code JSON shapes live only in `internal/adapters/claudecode`. The
implementer must verify the current hook I/O schema against the official hooks
documentation at implementation time and record the verified doc URL + date in the
package doc comment (known unknown, §16).

### §13.4 Documented installation snippet

```json
{"hooks": {"PreToolUse": [{"matcher": "Bash",
  "hooks": [{"type": "command", "command": "agentgate hook claude-code", "timeout": 10}]}]}}
```

(User settings `~/.claude/settings.json` or project `.claude/settings.json`.)

---

## §14 Performance and reliability targets

| Metric | Target |
|---|---|
| `check` end-to-end (200-rule policy, cold) | < 50 ms p95 on Apple Silicon / modern x86 |
| hook adapter added latency | < 100 ms p99 |
| audit append (incl. flock+fsync) | < 10 ms p95 |
| binary size | < 15 MB |
| memory | < 50 MB RSS for any single invocation |

Reliability: no goroutine leaks (each invocation is one-shot); audit write failure in
`exec`/`hook` mode → the decision stands but a warning goes to stderr; audit write
failure never blocks or crashes the child process once started.

---

## §15 Testing strategy

1. **Unit tests** per package, table-driven; shellparse and engine aim ≥ 90% line
   coverage; every §8 walk rule and §9 semantic has at least one test.
2. **Bypass corpus** (`testdata/corpus/corpus.yaml`): adversarial inputs with the
   minimum acceptable decision under the reference policy (the starter policy):
   `sh -c 'rm -rf ~'`, `$(echo rm) -rf ~`, backticks, `eval`, `env -i rm …`,
   `command rm`, `exec rm`, `xargs rm`, `find / -exec rm {} \;`, `curl … | sh`,
   `bash <(curl …)`, `echo x > ~/.ssh/authorized_keys`, `tee ~/.aws/credentials`,
   `sudo anything`, `FOO=$(rm -rf ~) echo hi`, multiline scripts, quoting tricks
   (`r'm' -rf ~`), comments-only input. The corpus is a growing regression suite; the
   engine test harness asserts `actual ≥ expected` severity.
3. **CLI E2E** via testscript: temp `$AGENTGATE_HOME`, scripted init/check/exec/hook/log
   flows asserting stdout/stderr/exit codes, including hook JSON round-trips and
   `log verify` after simulated tampering.
4. **`policy test` dogfooding**: the starter policy's embedded tests run in CI.
5. **Race**: `go test -race ./…` in CI; a dedicated concurrent-hook audit test spawns
   ≥ 8 parallel writers and then verifies the chain.

---

## §16 Scope summary

### v1 in scope
Everything in §5–§13, delivered by the 21 issues in `docs/ISSUE_PLAN.md`.

### v1 non-goals
- Windows support; PATH-shim / LD_PRELOAD interception; runtime sandboxing or containment;
  network-level firewalling; GUI/daemon; policy remote sync; secrets scanning of command
  output; multi-user setups; i18n of CLI output (English only).

### v2 deferred (explicitly designed-around, not blocked)
- Codex CLI adapter (integration point unclear today — research task).
- Claude Code `Write`/`Edit`/`WebFetch` tool coverage in the hook (path/url policy).
- MCP proxy mode (wrap arbitrary MCP servers with the same policy engine).
- `agentgate log suggest` — mine the audit log and propose allowlist rules.
- Rule packs / includes (`include:` in policy), env-var matching, time-boxed rules.
- Signed/forward-secure audit logs; remote log shipping.
- Windows.

### Known unknowns (may spawn new issues during implementation)
1. Claude Code hook JSON schema drift across versions (§13.3) — verify at impl time.
2. `mvdan.cc/sh` behavior on rare bash constructs (coproc, extglob) — corpus will grow.
3. Whether `ask` in hook mode renders acceptably in all Claude Code surfaces (CLI vs
   IDE vs web) — needs a manual check on current versions.
4. Exact `timeout(1)` flag surface differences (GNU vs BSD) for the wrapper table.
5. Performance of walk-up project discovery on network filesystems.
