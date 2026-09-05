# Issue 02: CI — build, test, lint

## Title

Add GitHub Actions CI: build, test with race detector, lint on macOS and Linux

## Summary

Add `.github/workflows/ci.yml` running build, `go test -race`, and golangci-lint on
`ubuntu-latest` and `macos-latest` for every push and pull request targeting `main`.

## Context

All later issues rely on CI as the merge gate (ISSUE_PLAN validation strategy #2).
The repository's `main` is protected; changes arrive via PR.

## Scope

- One workflow file `.github/workflows/ci.yml`, workflow name `ci`.
- Triggers: `pull_request` (all branches), `push` to `main`.
- Job matrix: `os: [ubuntu-latest, macos-latest]`.
- Steps: checkout → setup-go (version from `go.mod`, module cache on) →
  `make build` → `make race` → golangci-lint official action (pinned version).
- Concurrency group cancels superseded runs per ref.
- Permissions block: `contents: read` only (least privilege).

## Detailed Requirements

1. Pin all actions by major version at minimum (`actions/checkout@v4`,
   `actions/setup-go@v5`, `golangci/golangci-lint-action@v6` or current majors).
2. `setup-go` uses `go-version-file: go.mod`.
3. Lint runs only on ubuntu (matrix exclude for macOS) to save minutes; build+race run
   on both OSes.
4. Total runtime target < 5 minutes with warm cache.
5. No secrets used or required.

## Acceptance Criteria

- [ ] CI runs on a test PR and is green on both OSes.
- [ ] A deliberately failing test on a branch turns CI red (verified once, then
      reverted before merge).
- [ ] Workflow has `permissions: contents: read` and a concurrency group.

## Validation

Open a draft PR with a trivial change; confirm both matrix legs pass. Push a commit
adding `func TestFail(t *testing.T){t.Fatal("x")}`; confirm red; drop the commit.

## Dependencies

Issue 01 (Makefile, module).

## Non-goals

Release/publish workflows (issue 21); coverage upload; scheduled runs; Windows.

## Design References

DESIGN §5 (toolchain), §15 (testing strategy); ISSUE_PLAN validation strategy.
