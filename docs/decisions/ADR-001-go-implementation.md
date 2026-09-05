# ADR-001: Implement agentgate in Go with mvdan.cc/sh

- Status: accepted
- Date: 2026-07-05
- Deciders: design agent (Fable), pending human ratification

## Context

agentgate is a local CLI security tool that must (a) parse real-world bash command
strings into an AST reliably, (b) ship as a single dependency-free binary users can drop
into `$PATH` and reference from agent hook configs, (c) start fast enough to run on every
tool call (< 100 ms budget, §14 of DESIGN.md).

## Decision

Implement in **Go ≥ 1.22**, module `github.com/Saber5656/agentgate`, with this fixed
dependency set: `spf13/cobra` (CLI), `mvdan.cc/sh/v3` (`syntax` package only — parsing,
never interpretation), `bmatcuk/doublestar/v4` (globs), `gopkg.in/yaml.v3` (strict
policy decode), `gofrs/flock` (audit lock), `rogpeppe/go-internal/testscript` (E2E,
test-only). Adding any other dependency requires a new ADR.

## Alternatives considered

| Option | Why rejected |
|---|---|
| Rust | Best-in-class binaries, but no shell parser with `mvdan.cc/sh`'s bash coverage and maturity; `conch-parser`/`yash-syntax` are less battle-tested. Slower delivery for a weaker implementation agent. |
| TypeScript/Node | Requires a runtime on the user machine and adds ~100 ms+ startup; unacceptable for a per-tool-call hook. Packaging as a single binary (pkg/bun) is fragile. |
| Python | Same runtime/startup objections; `shlex` cannot parse compound commands, `bashlex` is unmaintained. |

`mvdan.cc/sh` is the decisive factor: static analysis quality of the firewall is bounded
by parser fidelity, and it is the only maintained, widely deployed (shfmt) bash parser
in any candidate language.

## Consequences

- Single static binary per platform (darwin/linux × arm64/amd64) via goreleaser.
- Shell **execution** in `exec --shell` still delegates to `/bin/sh -c`; agentgate never
  interprets shell itself (parse-only), which keeps the trust boundary clear.
- Windows is deferred (path semantics, no flock; DESIGN §16).
