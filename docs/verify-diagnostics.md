# Verify & Test Diagnostics

Failure-mode reference for interpreting `poof verify` and `poof task test-results` output. Use this when:

- `test-results` shows failures for files you don't recognise.
- `verify` exits 1 with `"no fresh test results were produced"`.
- Result counts or `--history` output don't match what you expect.

All citations below are prose `file:line` references to the Poof platform repo and the `poof-cli` repo on the author's machine. They are not Markdown links because this doc ships in a published skill.

## 1. Diagnostic table

### Symptom: `test-results` shows failures for files I don't recognise

The server collapses execution rows by `(source, fileName, testName)` (platform `src/app/api/project/[projectId]/test-results/route.ts`, dedupe loop after the bounded raw fetch) and the CLI further collapses by `(source, fileName)` (CLI `internal/cli/doctor.go`, `collapseResultsToLatest`). Rows for since-deleted files survive both collapses — pass *and* fail rows alike, since collapse is presence-blind. The harmful "ghost failure" signal happens specifically when the surviving row is `failed` or `error` (passed deleted-file rows are usually invisible noise).

Three result classes need different cross-checks before being classified as ghosts. Important constraints for the first two:

- The `test-results` response shape (`id, source, fileName, testName, status, counts, lastError, duration, startedAt`) does **not** include `taskId` or any tool invocation ID. So there is no definitive row-to-tool join key.
- `poof project messages` exposes stored conversation content, but the MCP tool-card extractor only preserves generic fields (`toolName`, `mcpServerName`, `mcpToolName`, `description`, and a small set of input metadata like `fileName`/`command`/`linesAffected` derived from `file_path` or `path`). It does **not** reliably preserve `run_ui_test`'s `testName`, nor `call_backend_api`'s full method+path combination. Assistant narrative in stored messages may mention a test or endpoint by name, but no guaranteed join-key field is exposed.
- Net: messages provide supporting narrative context that some inline UI or backend-API check ran. They do not provide a structured way to associate that activity with a specific result row.

**Inline UI tests from `run_ui_test`.** When the originating tool invocation omits a `fileName`, the row is inherently non-file-backed and cannot be a ghost. But the normalized `test-results` response falls back to `testName` for the `fileName` field when the original was absent, so a row with `source: "ui"` alone does **not** prove inline-vs-file-backed origin. Narrative in `poof project messages -p <id> --limit 100 --json` may indicate that `run_ui_test` was used recently with similar test names, which raises the prior probability the row is inline — but this is not a definitive provenance association. If provenance is unconfirmed: file presence on a `lifecycle-actions/ui-test-*.json` cross-check can rule in a file backing; file **absence is inconclusive** (the row may be a deleted-file ghost or a legitimate inline result). Keep such rows unresolved unless stronger context (e.g. the user states what they ran) lands.

**Backend-API synthetic rows from `call_backend_api`.** Written with `fileType: 'test'` and a `test-backend-api-<method>-<path>.json` filename. The platform records normal lifecycle test rows with the same `fileType: 'test'` for any filename starting with `test-`, and the response surface exposes normalized result fields, not tool provenance. So the `test-backend-api-` prefix is a **clue, not proof**: a user could in principle create a real `lifecycle-actions/test-backend-api-something.json` file. Narrative in `poof project messages` may indicate that backend-API verification ran recently against the implied endpoint, which raises the prior probability the row is synthetic — but again this is not a definitive association. If provenance is unconfirmed: file presence cross-check can rule in a file backing; file **absence is inconclusive**. Keep such rows unresolved unless stronger context lands.

**Formal-verification rows.** File-backed at `formal-verifications.json` or `formal-verifications/*.json`. **Can** be ghosts if those files were deleted, but a `lifecycle-actions/*` cross-check won't detect them; check `formal-verifications.json` and `formal-verifications/*.json` separately.

Note on which CLI command exposes tool messages: `poof project messages -p <id> --limit 100 --json` returns conversation content; `poof chat active -p <id>` returns only activity state (active/idle), not tool messages, so it is useful for "is something still running" but cannot be used to confirm provenance.

Cross-check with `poof files get -p <id> --path "lifecycle-actions/*" --list` (requires a credit purchase — see the `poof files get` row in [api-reference.md](api-reference.md)). **Interpret the response carefully:**

- **File retrieval succeeded but the CLI reports `no files matched --path "lifecycle-actions/*"`**: the snapshot the files endpoint returned (and the CLI then locally glob-filtered) contains no lifecycle-action files. This is strong evidence that test-results rows referencing `lifecycle-actions/*.json` are ghosts relative to *that snapshot*. Important caveat on snapshot choice: the files endpoint selects the first recent task carrying a `codeDeltaUrl` regardless of task status, while `run_all_lifecycle_tests` selects the latest task with `codeDeltaUrl` and `status: 'completed'`. A newer failed, cancelled, or in-progress task carrying a code delta will leave the two views looking at different snapshots — they don't automatically converge in any "settled" state, only when the most recent code-delta-bearing task happens to also be the most recent completed one.
- **Any other error** — file retrieval itself failed (authorization or credit 403, `NO_CODE_REVISION` when the recent-task window exposes no code delta, or transient request failures) — means the cross-check did **not** complete. Do not infer file presence (or absence) from a failure response. For free-tier projects where the credit error is unavoidable, rely on your own knowledge of recent file edits or task messages around the incident.

### Symptom: `verify exits 1 with "no fresh test results were produced"`

`poof verify` isolates fresh results by comparing the result-page IDs returned before and after the verify prompt — each snapshot is a **single page of 100** (per the CLI's `internal/cli/verify.go` baseline + after-fetch logic). It then applies the CLI's per-file collapse to the fresh set (so an earlier failed row that the AI fixed and re-ran within the same file doesn't double-count), and computes `freshSummary` on that collapsed view. The gate fails iff the collapsed-fresh view is empty OR any row in it is `failed`/`error`. Results outside the page-of-100 in either snapshot are invisible to the diff. The exit status is a **strict gate signal, not authoritative proof of current suite state** — it conflates "no fresh artifacts visible in the bounded diff" with "tests broke," and the per-file collapse means within-same-file failures earlier in the run don't fail the gate if the latest fresh row for that file passes.

Cross-check with `poof files get --list` (do lifecycle test files exist?) and `poof task list -p <id> --json` to confirm a relevant turn ran or completed around the incident. **Note:** `poof task list` exposes task metadata and `codeDeltaUrl` per task — it can show that a turn happened, but does **not** prove the turn produced test artifacts or list lifecycle files. `poof task test-results` shows the recorded result rows (subject to the ghost-row and bounded-page caveats above); the API doesn't expose a `taskId` on each row, so it cannot prove which turn produced a given row either.

### Symptom: `test-results shows 0 total but the AI just claimed tests passed`

Already covered in [testing.md](testing.md) — see the `summary.total = 0` note near the bottom. Don't duplicate it here; that section already directs you at `poof project messages -p <id> --limit 100 --json` and the related tool-call inspection.

## 2. What `--history` actually does

The server-side response is already deduped to one row per `(source, fileName, testName)`. The CLI's `--history` flag only bypasses the CLI's **additional** `(source, fileName)` collapse, exposing multiple `testName` variants within the same file. It does **not** show raw run history — the platform doesn't expose that.

## 3. The "0 fresh = fail" gate

The CLI's `verify` declares pass only when `freshSummary.Total > 0 && freshSummary.Failed == 0 && freshSummary.Errors == 0`, where `freshSummary` is computed over the bounded-diff fresh set **after** applying the CLI's per-file collapse (`source|fileName`). The bounded-diff comes from comparing single pages of 100 before and after the verify run. So the gate has three layers — page-of-100 fetch, before/after ID diff, per-file collapse on the diff result — and the pass/fail decision is made on that summary. `poof iterate` uses the same three-layer pipeline for its `freshResults` / `freshSummary` ("no tests ran during this turn" fires when that collapsed-fresh view is empty); it additionally applies the per-file collapse to the returned after-page to produce its `results` / `summary` fields. Both should be read as "no fresh results were visible in the collapsed bounded diff," not "results were definitely not created."

Cross-check before treating either signal as a regression.

## 4. Quick reference commands

- `poof files get -p <id> --path "lifecycle-actions/*" --list` — lifecycle-action files in the snapshot selected by `files get` (requires credit purchase). Note: this snapshot may differ from what `run_all_lifecycle_tests` sees — see the snapshot caveat in section 1.
- `poof task test-results -p <id> --json | jq '.results | map({source, fileName})'` — what test-results currently shows.
- `poof task test-results -p <id> --history --json` — bypass the CLI's per-file collapse to see same-file `testName` variants.
- `poof project messages -p <id> --limit 100 --json` — recent conversation content, useful as supporting narrative context when reasoning about provenance.

## 5. CLI-side wording mismatch (known)

The installed `poof` CLI's `task test-results --history` flag-help text, command long help, and text-mode footer still describe `--history` as showing "raw server history" — the same inaccurate wording this skill is now correcting. A separate CLI documentation-only PR is needed to fix the CLI-side wording. Until that ships, trust this skill's wording over `poof task test-results --help` output.

## 6. What this doc does NOT claim

Defensive listing so future agents don't mistake hypothesis for fact:

- It does **not** claim `run_all_lifecycle_tests` is "delta-based discovery." The platform tool uses S3-snapshot-based discovery rather than current-turn-delta discovery. Its snapshot selection (latest *completed* task with `codeDeltaUrl`) is **not identical** to the files endpoint's (first recent task with `codeDeltaUrl`, regardless of status) — see the snapshot-selection caveat in section 1 above.
- It does **not** recommend specific workarounds for forcing test re-discovery (e.g. "no-op-touch a lifecycle file"). The real mechanism behind observed `freshSummary.Total == 0` incidents likely involves checkpoint timing between in-progress and completed tasks — not something we can claim with certainty from outside the platform.
- The platform-source citations above were verified against a local checkout of the Poof platform repo on the original author's machine; the deployed revision may differ. Update this doc if the platform behaviour materially changes.
