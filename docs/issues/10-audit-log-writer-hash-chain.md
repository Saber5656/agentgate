# Issue 10: Audit log writer with hash chain

## Title

Implement audit JSONL writer with flock serialization and SHA-256 hash chain

## Summary

Implement `internal/audit`: the append-only JSONL writer with per-UTC-day segments,
exclusive flock, fsync, and the record hash chain — DESIGN §12 / ADR-003. Reading and
verification CLIs are issues 16/17; this issue delivers the writer plus the shared
record codec both will reuse.

## Context

Every `check`/`exec`/hook invocation appends one record; concurrent hook processes are
the normal case. Correctness of the chain (canonical serialization) is the subtle part:
the canonical form is "encoding/json output of the fixed Go struct with hash set to
empty string" (ADR-003).

## Scope

Package `internal/audit`. Public API (exact):

```go
type UnitRecord struct {
    Argv []string `json:"argv"`; Action string `json:"action"`
    RuleID string `json:"rule_id,omitempty"`; Layer string `json:"layer,omitempty"`
    Opaque bool `json:"opaque"`
}
type ExecRecord struct {
    Ran bool `json:"ran"`; ExitCode int `json:"exit_code"`
    DurationMS int64 `json:"duration_ms"`; AskResolved string `json:"ask_resolved,omitempty"`
}
type Record struct {
    Schema string `json:"schema"` // "agentgate.audit/v1"
    Seq    int64  `json:"seq"`
    TS     string `json:"ts"`     // RFC3339Nano UTC
    Source string `json:"source"` // "check"|"exec"|"exec-dry"|"hook.claude-code"
    Session string `json:"session"`
    Agent  string `json:"agent"`  // "claude-code"|"cli"
    Cwd    string `json:"cwd"`
    Raw    string `json:"raw"`
    Mode   string `json:"mode"`   // "shell"|"vector"
    Final  string `json:"final"`
    Units  []UnitRecord `json:"units"`
    Hazards []string `json:"hazards"`
    PolicySHA struct{ Global string `json:"global,omitempty"`; Project string `json:"project,omitempty"` } `json:"policy_sha"`
    Exec   *ExecRecord `json:"exec"` // null for non-exec sources
    Prev   string `json:"prev"`
    Hash   string `json:"hash"`
}
func FromDecision(d engine.Decision, in engine.Input, source, session, agent string, ex *ExecRecord) Record
func HashRecord(r Record) (string, error)   // r.Hash forced to ""; sha256(prev+"\n"+json)
type Writer struct{ /* logsDir */ }
func NewWriter(logsDir string) *Writer
func (w *Writer) Append(r Record) (Record, error) // fills Seq/Prev/Hash; creates dirs/files 0700/0600
func LastHash(logsDir string) (hash string, err error) // helper shared with verify (17)
func ReadSegment(path string) ([]Record, error)        // strict decode, shared with 16/17
```

## Detailed Requirements

1. **Canonical bytes**: `HashRecord` marshals with `encoding/json.Marshal` on `Record`
   with `Hash: ""`; digest input = `r.Prev` + `"\n"` + those bytes. Struct field order
   above is frozen — a code comment must say "field order is the canonical form; see
   ADR-003; changing it is a schema break".
2. **Append algorithm** (all under `flock` on `logsDir/audit.lock`, blocking acquire
   with 5s timeout → error after):
   1. today's segment path `audit-YYYYMMDD.jsonl` (UTC now);
   2. `Prev`: last line of today's segment if non-empty; else last line of the newest
      older `audit-*.jsonl` (lexicographic max below today); else 64 zeros;
   3. `Seq`: last seq in today's segment + 1, else 1;
   4. fill `TS` (now UTC RFC3339Nano), compute `Hash`;
   5. single `write()` of line+`\n` on an `O_APPEND` fd, then `fsync`, then unlock.
3. **Strictness**: `ReadSegment` uses `json.Decoder` with `DisallowUnknownFields`
   (ADR-003: unknown fields would break re-serialization determinism); a bad line →
   error carrying line number.
4. `LastHash` never errors on a missing dir (returns 64 zeros) — first-ever write
   works on a fresh machine.
5. Writer failures must be non-fatal to callers by contract: callers (11/12/15) print
   a warning to stderr and continue (DESIGN §14) — document this on `Append`.
6. Dependency added: `github.com/gofrs/flock`.
7. **Tests**:
   - round-trip: Append 3 records → ReadSegment → recompute chain → all hashes match.
   - genesis: fresh dir → first record `Prev == strings.Repeat("0", 64)`, `Seq == 1`.
   - cross-segment link: write a fake `audit-20200101.jsonl` (via Writer with an
     injected clock — make `now func() time.Time` a Writer field for tests), then
     append "today" → first new record's Prev == old segment's last hash; Seq resets
     to 1.
   - concurrency: 8 goroutines × 25 Appends each (same Writer dir, separate Writer
     instances to simulate processes) → 200 records, seq strictly 1..200, chain valid.
   - determinism guard: a test that marshals a fixed Record and asserts the exact
     byte string (golden) — fails loudly if anyone reorders struct fields.
   - tamper: flip one byte in a middle line → recompute mismatch detected (the check
     itself lives here as a helper `func VerifyChain(logsDir string) error` — the CLI
     around it is issue 17).

## Acceptance Criteria

- [ ] All tests above pass, including `-race` on the concurrency test.
- [ ] Files/dirs created with 0600/0700.
- [ ] `VerifyChain` helper exists and is used by the tamper test (issue 17 reuses it).
- [ ] Golden-bytes determinism test present.

## Validation

```sh
go test ./internal/audit/ -v -race -cover
```

## Dependencies

Issues 03 (logs dir path), 08 (Decision shape for FromDecision).

## Non-goals

`log show/tail` (16); `log verify` CLI (17); retention/pruning (v2); signing (v2).

## Design References

DESIGN §12 (all), §14 (non-fatal writer failures); ADR-003.
