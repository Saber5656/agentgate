# Issue 17: `agentgate log verify`

## Title

Implement `agentgate log verify` chain verification

## Summary

Implement strict hash-chain verification over all (or selected) audit segments,
reporting the first break precisely. Reuses `VerifyChain` internals from issue 10;
this issue productizes it into the CLI with exact reporting. DESIGN §11.5, §12.3.

## Context

Tamper evidence (ADR-003) is only real if verification is easy and its failure output
is actionable.

## Scope

`internal/cli/log.go` (`verify` subcommand) + extending `internal/audit` verification
to return structured results instead of a bare error:

```go
type VerifyResult struct {
    SegmentsChecked int; RecordsChecked int64
    OK bool
    // on failure:
    BreakFile string; BreakLine int; BreakSeq int64
    Kind string // "hash-mismatch"|"prev-mismatch"|"seq-gap"|"decode-error"|"cross-file-link"
    Detail string
}
func Verify(logsDir string, files []string) (VerifyResult, error)
```

## Detailed Requirements

1. Default (`log verify` / `--all`): all segments ascending; explicit FILE args:
   verify those segments' internal chains AND their mutual links if consecutive
   (non-consecutive selections skip cross-file link checks with a stderr note).
2. Checks per record, in order: strict decode (issue-10 `ReadSegment` semantics) →
   `seq` == previous seq + 1 (or 1 at segment start) → `prev` == previous record's
   `hash` (or cross-file/genesis rule, DESIGN §12.3) → recomputed `HashRecord` ==
   stored `hash`.
3. First failure stops verification; output:
   `chain broken at FILE:LINE (seq N): KIND — detail` and exit 20. Clean:
   `ok: S segments, R records verified` exit 0. Unreadable dir/file → exit 65.
   Empty logs dir → `ok: 0 segments, 0 records verified`, exit 0.
4. Schema forward-compat: a record whose `schema` != `agentgate.audit/v1` → 
   `decode-error` (v1 verifier refuses unknown schemas; ADR-003 consequence).
5. **Tests**: clean multi-segment pass; each `Kind` triggered by a targeted fixture
   mutation (flip content byte → hash-mismatch; edit a `prev` → prev-mismatch; delete
   a middle line → seq-gap; garbage line → decode-error; edit first record of segment
   2's `prev` → cross-file-link); exit codes 0/20/65; explicit-files mode.

## Acceptance Criteria

- [ ] All five failure kinds detected with correct FILE:LINE:seq attribution.
- [ ] Exit codes exactly 0/20/65 per DESIGN §11.5.
- [ ] Verification of 10k-record fixture completes < 1s (sanity perf test, logged not
      gated).

## Validation

```sh
bin/agentgate check --shell 'ls' >/dev/null   # ensure at least one record
bin/agentgate log verify; test $? -eq 0
SEG=$(ls "$AGENTGATE_HOME"/logs/audit-*.jsonl | tail -1)
sed -i.bak 's/"final":"/"final":"X/' "$SEG"    # corrupt a record body
bin/agentgate log verify; test $? -eq 20
mv "$SEG.bak" "$SEG" && bin/agentgate log verify; test $? -eq 0
go test ./internal/audit/ ./internal/cli/ -run Verify -v
```

## Dependencies

Issue 10.

## Non-goals

Repairing broken chains; signing; continuous background verification.

## Design References

DESIGN §12.3, §11.5; ADR-003 (tamper-evident claim + schema consequence).
