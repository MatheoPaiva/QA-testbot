# GitHub App controller setup and operations

This guide covers the optional controller that runs from a separate bot repository. For hosted product setup, use the [Patchsentry website](https://patchsentry.vercel.app). For a credential-free local demonstration, start with the [README](../README.md#quick-start).

The controller is an integration path, not a prerequisite for the demo. Configure it only for repositories, environments and external deliveries you are authorized to use. Running the demo or local model dashboard does not activate it.

## Execution flow

The separate GitHub App workflow below is an optional integration path. **The public repository’s QA controller was verified disabled on November 10, 2026.** Check its current Actions state before operating an installation; the shipped configuration also has activation disabled. The cron expression in its source does not mean it is running. The no-key demo and the website's separately configured hosted flow do not enable this controller. An authorized operator must configure and explicitly enable it for an installation.

```mermaid
flowchart LR
    PR[Target PR: SHA, patches, description, specs, tests] --> Plan[Deterministic baseline plan]
    Plan --> AI[Optional AI specialists: inspect and select trusted suites]
    AI --> State[Durable state and execution request]
    State --> Prepare[Separate prepare job: download exact source]
    Prepare --> Test[Secret-free executor: network-isolated container]
    Test --> Results[Structured results and evidence artifact]
    Results --> Complete[workflow_run completion controller]
    Complete --> Final[AI final review: results, code, optional screenshots]
    Complete --> Slack[Prompt test completion]
    Final --> SlackFinal[Final findings and targeted questions]
    Complete --> Drive[Drive current summary and run history]
    Decision[Authorized GitHub decision comment] --> State
    Poll[Optional 10-minute reconciliation] --> State
```

For this optional controller path, the target application needs a supported suite or runnable preview and installation of the configured GitHub App. Bot source, controller workflows and Slack/Drive credentials stay outside the product repository. **When the controller is configured and enabled**, its ten-minute reconciliation discovers new PR revisions/comments and `workflow_run` handles execution completion. Optional Slack delivery precedes Drive I/O; a later message can supply an upload link. None of those external deliveries occurs in the reviewer demo.

The privileged controller checks out reviewed bot code. A separate prepare job downloads an exact commit archive as data. The executor uses a fresh GitHub-hosted runner and runs PR code in containers with no network, integration credentials, Docker socket, writable source mount, or privileged capabilities. Bot-owned preview tests use a separate bounded browser process with Chromium sandboxing, fresh contexts, and an explicit preview-origin policy; no target scripts are launched by that worker. The privileged collector reads bounded JSON and optional validated screenshot bytes from an artifact; it never extracts or executes that artifact's code. See [execution details](execution.md).

## Planning and supported tests

`src/planner.js` combines changed paths, added/removed patch content, PR descriptions, configured specification files, and the path/content of existing tests. Each check has a stable ID, reason, expected result, execution method, and supporting files/requirements. The fingerprint includes the exact SHA and inspected inputs, so a new commit with identical filenames is reassessed.

`qa-rules.json` supplies feature journeys and scenario expectations. `qa-config.json` supplies the **trusted runner allowlist**. PR content cannot specify commands or credentials. An essential baseline always runs, with additional suites selected using `match`, `patchMatch`, and `requirementMatch` regular expressions. The shipped feature rules reference example runner IDs such as `auth-tests` and `api-tests`: configure these for your application. A selected but unconfigured scenario is explicitly **blocked**, never assumed covered by the baseline.

Supported runner types:

| Type | Intended target | Required output |
| --- | --- | --- |
| `node-test` | Node built-in tests, explicit files or relative globs | TAP produced by `node --test --test-reporter=tap` |
| `npm-script` | A trusted named npm script running the application's suite | Complete TAP, `format: "tap"` |
| `npm-script` with `format: "vitest-json"` | Existing Vitest suite, optionally in a configured relative `cwd` | Complete JSON with consistent assertion counts |
| `go-test` | Trusted relative Go package paths | Completed `go test -json` package and test events |
| `web-preview` | Bot-owned UI suite against an exact-revision disposable preview | Browser assertions, discovery, accessibility/layout findings, screenshots and optional visual differences |
| `native-app` | Declared Appium journeys on a separately enrolled disposable device or VM | Artifact-bound results and device evidence; see [native QA](native-qa.md) |
| `junit-report` | Import of tests already executed by trusted CI | Bounded JUnit XML with a locally enrolled identity/digest attestation; see [CI imports](releases.md#cross-platform-ci-reports) |
| `workspace-node-test` | Explicitly enabled, digest-bound bot-owned Node tests | Structured Node test results from the offline container; see [authored tests](releases.md#website-jobs-and-authored-unit-tests) |

An example additional runner:

```json
{
  "id": "auth-tests",
  "type": "node-test",
  "files": ["test/auth/*.test.js"],
  "match": ["auth", "session"],
  "patchMatch": ["[+-].*permission"],
  "timeoutMs": 60000,
  "expected": "Valid sessions succeed; expired sessions and forbidden roles are rejected.",
  "requirement": "specs/access-policy.md"
}
```

The default image supports dependency-free Node tests. Projects with dependencies need a reviewed image containing those dependencies at `/opt/qa/node_modules`; Go suites need their toolchain and cached modules in the image. Configure `execution.image`, preferably with an immutable digest. No install hook runs against PR code with privileged credentials. Opt-in `execution.scratchWorktree` supports framework build files and ESM resolution in a disposable container copy while preserving the read-only source mount. Offline source-test containers do not support service containers, production data, or external network calls; preview browser checks use the separately enrolled network policy. Locally attested JUnit imports and explicitly enrolled authored Node tests are described in [build/release QA](releases.md). Details are in [test execution](execution.md) and [autonomous QA](autonomous-qa.md).

Patch analysis identifies validation, authentication, authorization, error handling, and dynamic execution observations. These are heuristics, not a security audit or proof of correctness. Missing expectations, explicit `TBD`/`open question` text, and findings requiring interpretation produce targeted human checks. Default CSS changes select automated browser coverage rather than imposing routine manual visual sign-off. Contradictions can be made machine-detectable with `QA-EXPECT key: value` lines in descriptions/specs: different values for the same key create an owner question. The deterministic planner does not interpret arbitrary prose contradictions; optional AI reviewers can propose additional questions with explicit uncertainty. Test association is heuristic and gaps are included in reports. Generated source-code tests remain inert candidates. The autonomous path can execute validated declarative journeys and retain them after two verified passes, preserving operator-owned assertions. The bot never executes model-supplied commands or silently applies suggested product fixes.

For a check needing separately authorized access or external side effects, add a rule scenario with `method: "human"`, `kind: "access"`, a specific `question`, `reason`, and `expected` result. The request asks the configured owner to provide an authorized test result and evidence; the bot never grants itself external access or runs that action through a comment.

Configured specs are read at the PR SHA. Large/truncated file lists, trees, inaccessible specs, and incomplete inspection become visible limitations and required blocked checks. At most 40 existing test files are inspected to bound API usage; tune this implementation for large repositories rather than silently asserting complete coverage.

## Configure integrations

Configuration is reviewed in the **bot repository**:

| File | Purpose |
| --- | --- |
| `qa-config.json` | AI models and limits, scope, runners, environment, owners, retry bounds, Drive destination, state branch, trusted workflow ref |
| `qa-rules.json` | Feature rules, regression expectations, human scenarios |
| `people.json` | Verified GitHub-to-Slack mapping only; not a review roster |

Set `scope.prNumbers` to specific PR numbers, or use `PR_NUMBER` to select one PR without needing a label. Otherwise the default scope is PRs labeled `needs-qa`; set `scope.milestone` to select a release instead. Previously tracked PRs remain tracked after the completion label replaces `needs-qa`. `qa-complete` is based on results and decisions. The SHA-bound `qa-bot` commit status is the authoritative merge check; labels are a reconciled convenience and cannot be atomically updated with a concurrent force-push.

Protect the bot default branch, the `qa-state` branch, and the **`qa-control` GitHub environment**. Restrict the environment's deployment branches to reviewed bot refs and allow the controller to write state. Do not add a per-run manual environment approval if automatic unattended testing is intended. Use a private bot repository when target code, reports, or evidence are private; state and Actions artifacts contain target information. Set `executionRef` to the trusted bot branch (default `main`).

Create and install a GitHub App on the target repository with:

- Metadata, Contents, and Pull requests **read**.
- Issues **read/write** for QA comments and labels (Pull requests read/write if required by your organization for PR comments).
- Commit statuses **read/write** for the exact-SHA `qa-bot` status.

The bot repository's `GITHUB_TOKEN` needs Contents write for state/request persistence and Actions write for dispatch and artifact reads. App credentials are separate from that token. No target Contents write, workflow write, commit push, or merge permission is needed. Keep `targetAccess: "report-status"` to post comments, labels, and statuses; `"read-only"` disables those metadata writes too. The controller rejects using the same repository for the product and bot state.

Configure secrets/variables in the protected `qa-control` environment or bot repository:

| Name | Kind | Use |
| --- | --- | --- |
| `OPENROUTER_API_KEY` | Secret | Optional hosted AI calls; only in the privileged controller |
| `LOCAL_MODEL_API_KEY` | Secret | Optional authentication to the chosen local model server |
| `QA_CONTROLLER_RUNNER` | Variable | Optional dedicated controller runner label for a local model server; default `ubuntu-latest` |
| `QA_APP_ID`, `QA_APP_PRIVATE_KEY` | Secrets | Target GitHub App |
| `TARGET_OWNER`, `TARGET_REPO` | Variables | Target repository |
| `SLACK_WEBHOOK_URL` | Secret | Incoming webhook for the QA channel |
| `SLACK_BOT_TOKEN` | Secret | Optional bot transport; requires the channel variable |
| `SLACK_CHANNEL_ID` | Variable | QA channel for the optional bot transport |
| `GOOGLE_SERVICE_ACCOUNT_JSON` | Secret | Google service account for a Workspace Shared Drive |
| `GOOGLE_DRIVE_ACCESS_TOKEN` | Secret | Alternative OAuth access token; expires and must be refreshed |
| `DRIVE_FOLDER_ID` | Variable | Report root; overrides `drive.folderId` |
| `DRIVE_SHARED_DRIVE_ID` | Variable | Optional Shared Drive ID; overrides `drive.sharedDriveId` |
| `QA_ENABLED` | Variable | Default `false`; explicit activation gate |
| `DRY_RUN` | Variable | Defaults to `true`; `false` is required for external writes |

Local dashboard selections update `qa-config.json`. Commit that reviewed configuration to the bot repository to use it in Actions, and configure the OpenRouter secret in the protected environment. AI enablement and production activation are separate; both remain off in the shipped configuration. The selected backend receives source excerpts and opted-in screenshots. OpenRouter forwards them to its selected provider; local mode sends only to the configured loopback model endpoint and has no OpenRouter fallback. See [privacy and cost controls](ai.md).

For local `npm run sync`, use `APP_ID`/`APP_PRIVATE_KEY` (or `QA_APP_ID`/`QA_APP_PRIVATE_KEY`), `TARGET_OWNER`, `TARGET_REPO`, `BOT_REPOSITORY=owner/bot`, and a bot-scoped `GITHUB_TOKEN`. Set `DRY_RUN=true` first. Dry-run may read GitHub to construct plans, but it previews writes and does not persist dedupe state, send Slack messages, upload Drive files, change GitHub state, or dispatch workflows. `npm run demo` is the fully offline alternative.

### Google Drive

Enable the Drive API in your Google Cloud project. Configure one existing report root folder and grant only the required writer access to the bot identity. The adapter creates:

```text
configured-root/
  owner_repo/
    PR-123/
      current.md
      run-<logical-id>-attempt-<n>-actions-<run>-<attempt>-<revision>.md
```

Reports contain the PR identity, tested revision/environment, plan, results, actual test counts, static observations, bounded evidence, skipped/untested checks, questions, decisions, outcome, and limitations. Individual logical runs and Actions attempts have distinct history files. `current.md` is updated only by the current assessment. No ACL or public-link API is called: sharing is inherited from the configured destination.

For **Workspace Shared Drives**, add the service account as a member with permission to create folders and files, and provide both root folder ID and Shared Drive ID. The adapter uses `supportsAllDrives`, `includeItemsFromAllDrives`, and the configured Drive corpus. Service accounts do not have personal My Drive storage quota; use a Shared Drive for this mode.

For a **shared My Drive folder**, use user OAuth and explicitly grant the app access to the selected folder (for example, using Google Picker). The adapter accepts an OAuth access token or, programmatically, a `getAccessToken` callback. The callback can integrate your refresh-token store; the CLI's fixed access-token mode does not refresh automatically. Both modes use `drive.file` access. Service-account JWT exchange and refresh are built in; domain-wide impersonation is not implemented.

Uploads use managed identity properties and checkpoint reserved file IDs before creation, allowing retries after an interrupted response without duplicating files. If Drive fails, retained markdown, upload IDs, and bounded retry state remain in `qa-state`. Test status is unchanged. Refer to Google's [service-account authentication](https://developers.google.com/identity/protocols/oauth2/service-account), [file authorization](https://developers.google.com/workspace/drive/api/guides/api-specific-auth), and [Shared Drive support](https://developers.google.com/workspace/drive/api/guides/enable-shareddrives).

### Slack and human decisions

Create an incoming webhook bound to the QA channel, or use a bot token with `chat:write` and a channel ID. The existing webhook capability is retained. Incoming webhooks append messages; they do not return a message timestamp for later updates. This implementation posts a follow-up when a report becomes available. Bot transport returns a timestamp and supplies a stable client message ID, but updates and interactive buttons are not implemented. See [Slack's webhook documentation](https://api.slack.com/messaging/webhooks).

Only independently verified Slack member IDs are used:

```json
[
  { "github": "feature-owner", "slackId": "U0123456789", "verified": true }
]
```

Add those GitHub logins to `featureOwners` by the exact rule area label, or to `decisionReviewers`. Set `qaGroupId` to a verified Slack user-group ID such as `S0123456789`. The bot selects the owner/reviewer, includes the author for implementation questions, and falls back to the author and/or QA group when appropriate. It never guesses identity from display names or contacts people simply because they appear in the directory. Slack identity mapping does not itself authorize a decision.

Each request includes the check, revision, question, reason, and response instructions. An authorized person responds on the PR using a **new single-line GitHub comment**:

```text
/qa-tested check:requirements-clarification revision:<full-40-character-SHA> result:pass reason:The intended threshold uses post-discount subtotal; verified this revision against the accepted specification.
```

Use `result:fail` when the check is unsatisfactory. The GitHub account must be a configured feature owner or decision reviewer. The PR author may resolve an `implementation` question, but cannot approve a requirement, visual judgment, or security interpretation solely by being the author. Each accepted decision records responder, explanation, timestamp, comment ID, check ID, and SHA. Acknowledgments, ordinary reviews, missing reasons, stale SHAs, pre-assessment comments, and unauthorized comments do not resolve checks. Editing an already processed comment does not revise a decision; post a new comment. A decision cannot override a failed or unsupported automated check.

Only affected human checks wait. Independent automation continues. When automated checks pass and only a mandatory human decision remains, Slack says exactly: **“Automated checks passed; human QA review pending.”** When no required decision remains and checks pass, QA completes automatically. For an enabled GitHub-controller installation, responses are reconciled on the next configured scan.

## State, reruns, and recovery

`qa-state/state.json` holds current PR pointers, run and attempt history, decisions, requests, notification receipts, report IDs, retained report text, and an integration outbox. Immutable `requests/<key>.json` files are persisted before dispatch. GitHub Contents updates use SHA compare-and-swap, a 20-minute durable lease, and a workflow concurrency group. A conflict aborts before side effects; the next reconciliation retries. The controller job's ten-minute timeout is shorter than the lease. Do not run controllers that bypass this lease against the same state.

Normal repeated events reuse the assessment. A new SHA or changed planning input creates a fresh assessment and invalidates old decisions without deleting history. Late artifacts can update historical reports but cannot replace the current assessment. Every artifact must match repository, PR, SHA, environment, logical run, and Actions attempt. Missing reports, zero tests, malformed TAP, canceled jobs, unsupported tests, and timeouts never report successful QA. Supplied summaries cannot override results.

For a configured installation whose controller has been explicitly enabled, use **Run workflow → QA controller** with `rerun_token` set to a unique value for an intentional rerun; optionally specify `pr_number` to scope that rerun. Reusing the token deduplicates delivery. GitHub's native Actions rerun is also supported: each Actions attempt has a separate result artifact and historical report. Rerunning after modifying trusted runner configuration creates a new assessment. New metadata/requirement assessments conservatively require new human decisions, even at the same SHA.

External delivery attempts are persisted before calls. Exponential retry delays are bounded at one hour, with five attempts by default. Set manual `retry_integrations=true` after correcting an exhausted integration. This resets exhausted delivery attempts without inventing passing test results. A controller restart rediscovers dispatched workflow IDs by request key, reads completed artifacts, and retries pending uploads/messages. A lost HTTP response from a Slack webhook or workflow dispatch can still cause a rare duplicate external delivery; durable receipts prevent ordinary duplicates, and duplicate workflow results do not overwrite the selected run. Neither API supplies a general transaction with the bot's state store.

The result collector accepts up to 16 MiB compressed artifact data, 2 MiB JSON, and four validated screenshots totaling at most 6 MiB, never executes artifact content, and uses attempt-specific names. Logs expire after 30 days and source bundles after one day; report text and bounded failure evidence persist in state and Drive. State history is retained without automatic deletion; for large installations, plan state archival/storage migration before approaching GitHub's file-size limits.

For local `FileStore` use, writes are atomic and protected by `state.lock`. If a process crashes, stop all local workers before removing a leftover lock and restarting. Corrupt state fails closed; it is never silently replaced. Restore the state branch from its Git history if needed. Do not remove state to resolve an integration outage: doing so loses dedupe and decision history.
