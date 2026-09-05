# Issue 21: Release workflow

## Title

Add goreleaser config and manual release workflow

## Summary

Add `.goreleaser.yaml` and a tag-triggered GitHub Actions release workflow producing
signed-checksum binary archives for darwin/linux × arm64/amd64. Releasing remains a
deliberate human action (merge ≠ release).

## Context

DESIGN §5 fixes the target platforms; ADR-001 promises single static binaries. The
repository owner triggers releases by pushing a tag manually — no auto-release on
merge.

## Scope

- `.goreleaser.yaml`, `.github/workflows/release.yml`, `make snapshot` target.

## Detailed Requirements

1. goreleaser config:
   - builds: `cmd/agentgate`, `CGO_ENABLED=0`, targets darwin/linux × amd64/arm64,
     `-trimpath`, ldflags injecting `internal/version.{Version,Commit,Date}` from
     goreleaser template vars;
   - archives: tar.gz, name template `agentgate_VERSION_OS_ARCH`, containing binary +
     LICENSE + README.md;
   - checksum file `checksums.txt` (sha256);
   - snapshot config for local testing; changelog from conventional-ish commit
     titles, grouped (feat/fix/other).
2. Workflow `release.yml`: trigger `push: tags: ["v*"]`; permissions
   `contents: write` only; steps: checkout (fetch-depth 0) → setup-go → run tests
   (`make race`) → goreleaser-action (pinned major) with `GITHUB_TOKEN`. A tag on a
   commit where tests fail publishes nothing.
3. `make snapshot`: `goreleaser release --snapshot --clean` for local artifact
   inspection; document in USAGE or CONTRIBUTING section of README.
4. No code signing/notarization in v1 (document macOS Gatekeeper implication in the
   release notes template: users may need `xattr -d com.apple.quarantine`).
5. Version discipline: tags `vMAJOR.MINOR.PATCH`; first release `v0.1.0` (not 1.0 —
   schema contracts marked `/v1` are internal schema ids, not product semver;
   note this distinction in the workflow file comment).

## Acceptance Criteria

- [ ] `make snapshot` locally yields 4 archives + checksums with correct
      `agentgate version` output inside each.
- [ ] Release workflow green on a `v0.1.0-rc.1` pre-release tag in a test run
      (pre-release flag honored by goreleaser).
- [ ] Workflow has least-privilege permissions and pinned actions.
- [ ] No release is produced without the human pushing a tag (verified by workflow
      trigger config review).

## Validation

```sh
make snapshot && ls dist/ && tar -xOf dist/agentgate_*_darwin_arm64.tar.gz agentgate | file -
# maintainer-only, after human approval: git tag v0.1.0-rc.1 && git push origin v0.1.0-rc.1
```

## Dependencies

Issues 02 (CI green baseline), 19 (E2E suite is the release gate).

## Non-goals

Homebrew tap, package managers (v2); auto-release on merge; code signing; SBOM.

## Design References

DESIGN §5 (platforms), §11.9 (version output); ADR-001 (static binaries); user rule:
merge and release are separate gates.
