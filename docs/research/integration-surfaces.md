# Research note: agent integration surfaces for a local tool firewall

- Date: 2026-07-05 (knowledge-based comparative analysis; re-verify marked items at
  implementation time — this note materially motivated DESIGN §2/§13 and the v2 list)

## Question

Where can a local policy engine intercept an AI agent's tool executions, and which
surface should v1 target?

## Comparison

| Surface | Coverage | Reliability | Effort | v1? |
|---|---|---|---|---|
| Claude Code `PreToolUse` hook (`matcher: Bash`) | Every Bash tool call, receives full command string + cwd + session id; can return allow/ask/deny with reason | Documented, versioned harness feature; JSON I/O; `ask` integrates with the native permission prompt | Small adapter over the generic engine | **Yes — flagship** |
| Generic subprocess JSON API (`check --json`) | Any harness that lets users wrap tool execution | Fully under our control | Free (it is the engine's native interface) | **Yes** |
| Exec wrapper (`agentgate exec -- …`) | Harnesses with a configurable command wrapper; humans testing policy | Fully under our control | Small | **Yes** |
| Codex CLI | Codex has built-in sandbox + approval modes; as of knowledge cutoff there is **no stable external pre-exec hook** equivalent to Claude Code's (**re-verify**: Codex releases move fast) | Unknown | Unknown | v2 research task |
| PATH shims (fake `rm`, `git`, …) | Only shimmed binaries; trivially bypassed via absolute paths; pollutes non-agent shells | Fragile | Medium | No |
| MCP proxy (wrap MCP servers, filter `tools/call`) | MCP tools only — does not see the harness's native Bash tool | Protocol is stable; proxy is a real product surface | Medium-large | v2 |
| OS-level (EndpointSecurity/eBPF/seccomp) | Everything, including runtime behavior | High assurance but high privilege, platform-specific, heavy | Large | Out of scope (agentgate is a decision engine, not a sandbox — DESIGN §1) |

## Claude Code hook contract (as of knowledge cutoff — re-verify at impl time, DESIGN §13.3)

- Input on stdin (PreToolUse): `session_id`, `transcript_path`, `cwd`,
  `hook_event_name: "PreToolUse"`, `tool_name` (e.g. `"Bash"`), `tool_input`
  (`{command, description, …}` for Bash).
- Preferred output: exit 0 + stdout JSON
  `{"hookSpecificOutput": {"hookEventName": "PreToolUse", "permissionDecision":
  "allow"|"deny"|"ask", "permissionDecisionReason": "…"}}`.
- Legacy output (`decision: approve|block`) exists but is deprecated — do not emit it.
- Exit code 2 = blocking error with stderr fed back to Claude; we deliberately do not
  use exit codes for decisions (JSON is explicit and forward-compatible).
- Hooks have a configurable timeout (default 60s); we self-impose 1.5s (DESIGN §13.2).

## Conclusion (adopted in DESIGN)

v1 targets the generic JSON engine + Claude Code hook + exec wrapper. Codex adapter and
MCP proxy are the two highest-value v2 surfaces; both reuse the engine unchanged.
