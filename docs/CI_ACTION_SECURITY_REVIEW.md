# CI, GitHub Action and Publication Security Review

> Review date: 2026-10-11. Scope: `action.yml`, `.github/workflows/smoke.yml`, `publish-pages.yml`, source adapters and published-artifact verifier. This is a bounded review, not a third-party security certification.

## Confirmed safeguards

- **Workflow permission boundary:** smoke jobs use `contents: read`. Publication build uses `contents: read` and never writes production. The distinct publish job uses `contents: write`, requires a successful build and runs **only on `refs/heads/main`**.
- **Publish integrity:** build outputs one source snapshot, checks metadata, recomputed snapshot identity and file digests, validates SVG XML/safety, uploads a verified artifact, then the publish job downloads and re-verifies it before a **fast-forward-only** push to `activity-assets`. Review-branch GraphQL canaries have the publish job skipped.
- **Secret boundary:** the composite Action takes the GitHub token through an Actions input/environment variable rather than a command-line argument. The local Git source does not need this token. Public payloads use the `daily-aggregates-only-v1` policy, with restricted/private breakdown fields prohibited by the output verifier.
- **Source isolation:** `source-adapters.js` dispatches GitHub GraphQL and offline Git without coupling either to rendering. Git commands use `spawnSync` with an argument array (no shell), reject ref names starting with a dash, and have 20-second/16 MiB subprocess bounds.
- **Network failure policy:** GitHub requests time out after 15 seconds per attempt by default. Authentication, authorization, malformed JSON and GraphQL response errors do not retry; 429, 5xx, transport and timeouts retry within a bounded attempt budget. `GitHubSourceError` provides stable categories and status/attempt counters without tokens.
- **Compatibility checks:** smoke CI exercises the published composite Action in offline/local mode, CLI HTML/SVG outputs, 30/90/365 SVG integrity, the inline HTML controls, and a forward-only rollback. Real GraphQL canary builds cover the external API separately from local fixtures.

## Evidence

- [PR #7](https://github.com/dturovskiy/deus-commit-chart/pull/7) source refactor merged as `ab272c1d50e72e7bce01b3609100399fbdb7936e`.
- [Feature smoke 38094293267](https://github.com/dturovskiy/deus-commit-chart/actions/runs/38094293267): **49 Node tests** and **4 Python tests** passed, composite Action and rollback passed.
- [Feature GraphQL 38094296015](https://github.com/dturovskiy/deus-commit-chart/actions/runs/38094296015): real GitHub calendar, HTML and SVG build/verification passed, publish skipped.
- [Production publish 38094426663](https://github.com/dturovskiy/deus-commit-chart/actions/runs/38094426663) and [main smoke 38094426657](https://github.com/dturovskiy/deus-commit-chart/actions/runs/38094426657): **SUCCESS**.
- [Post-deploy monitor 38094479108](https://github.com/dturovskiy/deus-commit-chart/actions/runs/38094479108): published-asset integrity and freshness verified **SUCCESS**.
- Published manifest `workflowRun=38094426663`, `sourceCommit=ab272c1d50e72e7bce01b3609100399fbdb7936e`, `snapshotId=5e236808b8452d099416b06c3ab71e391457610690be0118f7dfe1b909dc61f6`, generated at `2026-10-10T23:16:12.605Z`. HTML and SVG were inspected directly via DEUS MCP; numbers 30d=228, 90d=789, 365d=3074 at snapshot time.

## Residual risks and accepted limitations

- **P2 supply-chain hygiene (OPEN):** the upstream GitHub Actions `checkout`, `setup-node`, `upload-artifact`, and `download-artifact` are referenced by their rolling major version tags (`@v4`). Pin them to verified immutable upstream commit SHAs and document how they are updated. No unverified hashes are recommended here.
- **P2 browser realism (OPEN):** inline script behavior is tested in a controlled JS VM, not across all real browsers/devices. The design is unchanged; no claim of universal visual parity.
- **P2 evidence retention (OPEN):** build artifacts have a finite 90-day retention policy. The fast-forward `activity-assets` branch retains source history, but externally cached older public assets and historical private aggregate metadata cannot be silently erased.
- **Separate public-visibility trade-off:** GitHub-sourced daily contribution totals can include token-visible private contributions; the private/restricted breakdown itself is not requested or published.
- **Observability:** GitHub scheduled workflow and README/image-cache delays are not bounded by the repository's two-hour publication schedule.
