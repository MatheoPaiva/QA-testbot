# Patchsentry

**Automated QA for pull requests, CI builds, and complete product releases.**

[![Bot verification](https://github.com/Vaidehi2510/QA-testbot/actions/workflows/check.yml/badge.svg)](https://github.com/Vaidehi2510/QA-testbot/actions/workflows/check.yml)
[![Node.js 24](https://img.shields.io/badge/Node.js-24-5FA04E?logo=nodedotjs&logoColor=white)](package.json)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

Patchsentry combines test execution, browser checks, optional AI investigation, and revision-bound evidence to help individuals and teams assess changes before shipping. **This repository is the QA engine.** Product source stays read-only; tests, reports, and suggested fixes live in the bot workspace.

[Website](https://patchsentry.vercel.app) · [Sample dashboard](https://patchsentry.vercel.app/demo) · [Reviewer guide](docs/reviewer-quickstart.md) · [Contributing](CONTRIBUTING.md)

## Quick start

Reproduce a real assertion failure without an account, API key, Docker, or browser installation. Use **Node.js 24** and npm; Git and access to the public npm registry are needed for the initial setup.

```sh
git clone https://github.com/Vaidehi2510/QA-testbot.git
cd QA-testbot
npm ci --ignore-scripts
npm run demo
```

The demo tests temporary copies of a synthetic shipping application:

| Scenario | Expected result |
| --- | --- |
| Healthy application | 4 tests pass. |
| Introduced defect: `amount >= 100` becomes `amount > 100` | 3 tests pass, 1 fails: an order of exactly $100 incorrectly incurs $5 shipping. |
| New revision | Previous results and decisions cannot approve the new code. |

Look for `expected: 0`, `actual: 5`, and the final `Demo checks passed` message. The demo exits successfully **because it caught the intended defect**; the defective application’s QA outcome remains failed.

The assertions execute for real. GitHub, Slack, Drive, and the sample owner response are mocked; no external writes or model calls occur. Temporary fixture state is cleaned up. The [reviewer guide](docs/reviewer-quickstart.md) explains the output and distinguishes real execution from simulation. The website’s sample dashboard uses illustrative data.

## What you can test

| Capability | What the engine does | Setup and scope |
| --- | --- | --- |
| Pull requests | Builds a check plan from committed changes, requirements, and existing tests; invalidates evidence when inputs change. | [Trusted runner configuration](docs/github-controller.md#planning-and-supported-tests) |
| Unit and integration tests | Executes configured Node, npm/TAP, Vitest, and Go suites; supports explicitly enabled bot-owned Node tests and attested JUnit imports. | [Execution](docs/execution.md) · [Authored tests and CI imports](docs/releases.md) |
| Web UI and APIs | Checks configured journeys, API responses, accessibility, keyboard interactions, responsive layouts, resources, and visual changes. | [Product QA](docs/product-qa.md) · [Autonomous QA](docs/autonomous-qa.md) |
| AI investigation | Uses bounded specialist tool loops for planning, code/security/UI observations, failure diagnosis, and suggested fixes or test candidates. | [OpenRouter and local models](docs/ai.md) |
| Builds and releases | Runs configured release suites and checks evidence against the source revision, build/version, platform, and artifact digest before deployment. | [Release QA and gates](docs/releases.md) |
| Native applications | Runs declared Appium journeys on separately enrolled disposable devices or VMs. | [Native QA](docs/native-qa.md); protocol fixtures do not validate real devices. |
| Reports and team updates | Records results, failure evidence, limitations, and targeted questions; optionally publishes GitHub metadata, Slack updates, and Drive reports. | [Controller integrations](docs/github-controller.md#configure-integrations) · [Team operations](docs/team-operations.md) |

Browser execution supports Chromium, Firefox, and WebKit. Visual comparisons need approved baselines, and meaningful journey/API checks need product-specific expectations. A full-release run covers the **configured release suite**; it does not imply exhaustive product coverage.

## How an assessment works

1. **Identify the target.** Bind the assessment to a committed revision and, for builds/releases, the artifact and version being shipped.
2. **Plan and execute.** Select trusted checks, run them in the enrolled environment, and collect structured outcomes and bounded evidence.
3. **Investigate failures.** Optionally use AI specialists to inspect code and results, propose fixes, and surface questions that need an authorized person.
4. **Report the decision.** Show what passed, failed, was skipped, or remains untested. Required failures, missing/stale evidence, and unresolved mandatory decisions prevent approval.

AI summaries and ordinary GitHub approvals cannot override failed required checks. Patchsentry suggests code changes; it does not edit, push, merge, or deploy connected product code.

## Three repositories, one product

| Repository | Responsibility |
| --- | --- |
| [patchsentry-web](https://github.com/Vaidehi2510/patchsentry-web) | Website, accounts, projects, model setup, hosted queue, reports, and release policies. Start here to use the product. |
| **[QA-testbot](https://github.com/Vaidehi2510/QA-testbot)** | This engine: planning, test execution, browser/native adapters, local model dashboard, and reproducible demos. |
| [Patchsentry-Ledger-Java](https://github.com/Vaidehi2510/Patchsentry-Ledger-Java) | Optional Java assessment archive. Deploy and connect your own service through the website’s project-scoped adapter to use remote archival. |

The website can dispatch supported hosted web QA without a customer-managed runner. Private-network, local-model, and native environments use separately enrolled infrastructure. The Ledger service is optional and is not deployed by this demo. See the [product architecture](https://patchsentry.vercel.app/architecture).

## Choose an AI backend

AI is optional and disabled in the shipped configuration. To explore the specialist workflow with scripted model replies and real fixture assertions:

```sh
npm run demo:ai
```

To choose models for live reviews, start the **local engine dashboard**:

```sh
npm run dashboard
```

Open **http://127.0.0.1:8787**. This dashboard configures the engine on your machine; hosted account/project setup belongs on the [website](https://patchsentry.vercel.app).

| Backend | What you provide |
| --- | --- |
| **OpenRouter** | Browse its public model catalog without a key. For inference, provide `OPENROUTER_API_KEY` through your terminal’s secret manager/environment, select eligible models, enable AI, and save configuration. |
| **Local server** | Start a compatible server such as Ollama or LM Studio, select **Local server**, and configure its literal loopback `/v1` address and model capability profile. Use `LOCAL_MODEL_API_KEY` only if that server requires authentication. |

Local examples are `http://127.0.0.1:11434/v1` and `http://127.0.0.1:1234/v1`. Profiles must declare context limits and tool support; screenshot review also requires image support. The bot does not download models or start the server. Local mode has no OpenRouter fallback.

Keys stay in the server/CLI environment, outside configuration and browser code. Selected source excerpts and opted-in screenshots go to the configured backend. Review [privacy, capability, and budget controls](docs/ai.md) before enabling live calls.

With models configured and any required credential available, review **committed** product changes:

```sh
npm run review -- --repo /path/to/product --base main --head HEAD
```

This saves a local report; it does not execute product tests by itself. Configure and supply `--web-suite /path/to/trusted-suite.json` for browser execution, or attach supported revision-bound results. `--dry-run` previews without model calls or saved state. See [review inputs](docs/ai.md#review-a-local-product) and [dashboard setup](docs/dashboard.md).

## Connect a product safely

For the optional GitHub App controller, maintain reviewed configuration in a **separate bot repository**:

| File | Purpose |
| --- | --- |
| [qa-config.json](qa-config.json) | Scope, trusted runners, AI models/limits, owners, execution settings, and integration destinations. |
| [qa-rules.json](qa-rules.json) | Feature expectations, regression scenarios, and specific human questions. |
| [people.json](people.json) | Verified GitHub-to-Slack identities; this is not an authorization roster. |

Product contents are read-only. A separately authorized GitHub App can post PR comments, labels, and exact-SHA statuses with `targetAccess: "report-status"`; `"read-only"` also disables those metadata writes. Keep private target information in a private bot workspace because state, reports, and artifacts contain target details.

Untrusted Node/npm/Go tests run in offline, resource-limited Docker containers without integration credentials or a writable source mount. Preview browsers use a separate sandboxed worker and enrolled origin/network policy. Native checks require their own isolated device environment. Trusted dependencies and test images must be prepared before execution.

The shipped configuration has `enabled: false` and `ai.enabled: false`. The public controller was verified disabled on November 10, 2026; its cron expression alone does not indicate activation. Follow the [controller setup and operations guide](docs/github-controller.md) for permissions, secrets, Slack/Drive delivery, activation, reruns, and recovery.

## Verification and current limits

After installing dependencies, run the local suite:

```sh
npm test
```

The suite also needs Git, Python 3, and `zip`/`unzip`. Node.js 24 matches CI; the package minimum is 22.17. Browser and Docker integrations require their explicit flags, installed browsers, and prepared images; an ordinary local run can skip them. Use the [full verification recipe](docs/reviewer-quickstart.md#4-reproduce-broader-engine-verification-separately) or [CI workflow](.github/workflows/check.yml) to run those checks.

Controlled verification includes real browser-engine fixtures, offline Go/Vitest containers, revision invalidation, artifact identity, failure classification, and source read-only checks. AI/provider and native-protocol fixtures exercise interfaces; they do not establish live model accuracy or physical-device interoperability.

Customer-specific provider access, real-device execution, workload defect recall, production capacity, and independent security review still need their own evidence. Passing tests does not establish a code-coverage percentage, staffing savings, or replacement of every QA responsibility. The [enterprise verification record](https://github.com/Vaidehi2510/patchsentry-web/blob/main/docs/enterprise-verification.md) tracks measured results and remaining dependencies.

## Documentation and contribution

- **First run:** [Reviewer quick-start](docs/reviewer-quickstart.md).
- **Test a product:** [Read-only product QA](docs/product-qa.md), [execution isolation](docs/execution.md), [autonomous browser QA](docs/autonomous-qa.md), and [release gates](docs/releases.md).
- **Models and teams:** [AI configuration](docs/ai.md), [local dashboard](docs/dashboard.md), [team operations](docs/team-operations.md), and [native environments](docs/native-qa.md).
- **Operate the controller:** [GitHub, Slack, Drive, state, and recovery](docs/github-controller.md).
- **Contribute:** [Contribution guide](CONTRIBUTING.md), [Code of Conduct](CODE_OF_CONDUCT.md), and [open issues](https://github.com/Vaidehi2510/QA-testbot/issues).
- **Report vulnerabilities privately:** [Security policy](SECURITY.md).

Reproducibility reports, failing synthetic regressions, accessibility feedback, and documentation improvements are welcome. You can contribute without an AI key or a company repository.

## License

[MIT](LICENSE) · Copyright (c) 2026 Vaidehi Pandya. Dependencies retain their own licenses.
