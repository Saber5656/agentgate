# ADR-003: Audit log as JSONL with SHA-256 hash chain (tamper-evident, not tamper-proof)

- Status: accepted
- Date: 2026-07-05
- Deciders: design agent (Fable), pending human ratification

## Context

The audit log is one of the three product pillars. It must be appendable from many
concurrent short-lived processes (every Claude Code Bash call spawns a hook), readable
with standard tools, and trustworthy enough to review after an incident.

## Decision

- **Format**: JSONL, one record per evaluation, fixed field order via Go struct marshal
  (DESIGN §12.1). Human tools (`jq`, `grep`) work directly.
- **Segmentation**: one file per UTC day, `logs/audit-YYYYMMDD.jsonl`; `seq` restarts
  per segment.
- **Integrity**: each record carries `prev` (previous record's `hash`; segment-first
  records link to the previous segment's last hash, genesis = 64 zeros) and
  `hash = sha256(prev + "\n" + record-serialized-with-empty-hash)`. Canonical form is
  defined as "the byte output of `encoding/json` on the fixed Go struct", avoiding a
  bespoke canonical-JSON spec. `agentgate log verify` re-derives the full chain.
- **Concurrency**: exclusive `flock` on `logs/audit.lock` around read-last → append →
  fsync. Single-host, single-user scope; no distributed coordination.
- **Honest claim**: the chain is **tamper-evident** — any edit, deletion, or reorder of
  past records breaks verification unless the attacker rewrites the entire suffix of
  the chain. An attacker with the user's OS privileges *can* rewrite everything;
  agentgate does not claim otherwise anywhere in docs or output.

## Alternatives considered

| Option | Why rejected (for v1) |
|---|---|
| SQLite store | Better querying, but adds a C dependency or cgo-free driver risk, complicates `jq`-style review, and locking across many short-lived processes is what flock already solves. |
| Per-record signatures (age/minisign keys) | Key management burden on users; still defeated by same-user attackers who can read the key. Deferred to v2 ("signed/forward-secure logs"). |
| Syslog/os_log | Not portable across macOS/Linux uniformly, poor structured querying, no integrity story. |
| No integrity mechanism | Cheap, but "audit log" without tamper evidence undermines the firewall positioning; chain cost is ~1 sha256 per record. |

## Consequences

- `log verify` is O(total records); acceptable at local scale (≤ ~10^5/day).
- The fixed-struct canonicalization means **any struct field change is a schema event**:
  additions append at the end and bump nothing (verify re-serializes what it parsed —
  unknown fields are rejected to keep determinism), renames/removals bump
  `agentgate.audit/v1` → `/v2`. Verifier keeps one decoder per schema id.
- Rotation/retention beyond daily segmentation (size caps, pruning) is deferred; a
  `log prune` command is a v2 candidate.
