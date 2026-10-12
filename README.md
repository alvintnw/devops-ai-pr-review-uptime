# DevOps Automation with n8n: AI Pull Request Reviewer + Uptime Monitor

Two DevOps automations in one n8n workflow, backed by one PostgreSQL database.

1. **AI pull request reviewer.** A GitHub webhook triggers a review of every pull request. The webhook signature is verified, secrets in the diff are found by regex before any AI is involved, an LLM reviews the rest, and a single advisory comment is posted on the PR and updated on every push.
2. **Uptime monitor.** Every 5 minutes your URLs are checked. Incidents are tracked in PostgreSQL, and Telegram is notified when something goes down and when it recovers, without alert spam or flapping.

---

## Table of Contents

- [Key Features](#key-features)
- [Design Principles](#design-principles)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Workflow Configuration](#workflow-configuration)
- [Usage](#usage)
- [Workflow Details](#workflow-details)
- [Database Schema](#database-schema)
- [Security](#security)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Limitations](#limitations)
- [Roadmap](#roadmap)
- [License](#license)

---

## Key Features

### Pull request reviewer

| Feature | Description |
|---|---|
| **Verified webhooks** | HMAC-SHA256 over the raw request bytes, compared in constant time. A missing secret or unreadable body is rejected (fail closed) and GitHub gets a `401`. |
| **Deterministic secret scanner** | AWS keys, GitHub and Slack tokens, Google and OpenAI-style keys, private key headers, and hard-coded credentials are found by regex in added lines, with the correct new-file line number. |
| **Redaction before the LLM** | Matched secrets are replaced with `[REDACTED]` in the text sent to the model. |
| **Focused AI review** | The model looks for bugs, security issues, error handling problems, performance regressions, and missing tests, and returns structured findings (severity, category, file, line, fix). |
| **Guardrails on AI output** | Findings must point at files that really are in the diff. Values are validated against fixed lists. Rules can raise the risk level but never lower it. |
| **One comment, updated in place** | A hidden marker lets the workflow edit its previous comment instead of posting a new one on every push. |
| **Smart filtering** | Drafts, bot authors, the `skip-ai-review` label, closed PRs, and non-PR events are skipped. Lock files, minified or generated files, vendored code, and binaries are excluded from the review. |
| **Cost control** | Each file diff is capped at 6,000 characters and the whole review at 30,000. Truncation is reported in the comment. |
| **Exactly one review per commit** | A unique constraint on repo, PR number, and commit SHA makes webhook redeliveries harmless. |
| **Telegram alerts** | High-risk PRs, blockers, and possible secrets are posted to your alert chat. |

### Uptime monitor

| Feature | Description |
|---|---|
| **HTTP checks with timing** | Status code, optional expected text in the body, per-monitor timeout, and latency, all checked in one Code node (10 requests in parallel). |
| **No flapping** | A monitor goes **down** after `failure_threshold` consecutive failures and **recovers** on the first success. A single blip never alerts. |
| **Atomic state changes** | One SQL statement updates every monitor, opens or closes incidents, and returns only real state changes. No repeat alerts while a service stays down. |
| **Incident history** | Every outage is stored with start, end, and error. A view calculates 30-day availability. |
| **Managed with SQL** | Add, disable, or tune monitors with `INSERT` and `UPDATE`. No workflow edits. |

### Shared

| Feature | Description |
|---|---|
| **Central error alerts** | An Error Trigger branch reports failed executions (workflow, node, error, link) to Telegram. |

---

## Design Principles

1. **Verify before you trust.** Nothing runs until the webhook signature is valid.
2. **Rules before AI, and rules win.** Secrets are detected deterministically. The AI cannot lower a risk level that rules raised, cannot invent files, and its text is escaped.
3. **Advisory, not blocking.** The reviewer comments. It never approves, requests changes, or merges.
4. **Idempotent by construction.** Unique constraints and `INSERT ... ON CONFLICT` make retries safe. Uptime state changes happen in a single statement.
5. **Fail loudly.** If the AI review fails, a human is alerted rather than the PR going silently unreviewed.

---

## Architecture

### Flow 1: Pull request review

```mermaid
flowchart LR
    W[GitHub Webhook] --> V[Verify Signature]
    V --> S{Signature Valid?}
    S -- no --> R401[Respond 401]
    S -- yes --> R200[Respond 200]
    R200 --> P[Parse PR Event]
    P --> SR{Should Review?}
    SR -- yes --> C[Claim Review<br/>one per commit]
    C --> D[Fetch PR Diff]
    D --> PD[Prepare Diff<br/>filter, scan, redact]
    PD --> H{Has Reviewable<br/>Changes?}
    H -- no --> SN[Skip Notice] --> A[Send PR Alert]
    H -- yes --> L[Review PR LLM]
    CM[Chat Model] -.-> L
    OP[Output Parser] -.-> L
    L --> B[Build Review Comment<br/>validate + markdown]
    B --> SP{Should Post?}
    SP -- no --> F[Review Failed Alert] --> A
    SP -- yes --> LC[List PR Comments]
    LC --> FE[Find Existing Comment]
    FE --> E{Comment Exists?}
    E -- yes --> U[Update Comment]
    E -- no --> CR[Create Comment]
    U --> M[Mark Review Done]
    CR --> M
    M --> NA{Needs Alert?}
    NA -- yes --> FA[Format PR Alert] --> A
```

### Flow 2: Uptime monitor

```mermaid
flowchart LR
    S[Uptime Schedule<br/>every 5 min] --> LM[Load Monitors]
    LM --> RC[Run Checks<br/>HTTP + timing]
    RC --> SV[Save Check Results<br/>one SQL statement]
    SV --> FU[Format Uptime Alert]
    FU --> SU[Send Uptime Alert]
```

If nothing changed state, **Save Check Results** returns no rows and the run ends silently.

### Error handling

```mermaid
flowchart LR
    ET[Error Trigger] --> FE[Format Error] --> SE[Send Error Alert]
```

---

## Tech Stack

| Component | Purpose |
|---|---|
| [n8n](https://n8n.io) | Workflow orchestration |
| GitHub REST API and webhooks | Pull request events, diffs, and comments |
| OpenAI `gpt-4o-mini` | Code review (structured output) |
| PostgreSQL | Review log, monitors, incidents, availability view |
| Telegram Bot API | Alerts |
| Docker Compose | Self-hosted deployment |

The model can be swapped in the **Chat Model** node. Bigger models review more deeply at higher cost.

---

## Repository Structure

```
.
├── README.md
├── sql/
│   └── schema.sql                          # pr_reviews, monitors, incidents, availability view
└── workflows/
    └── devops_pr_review_uptime.json        # n8n workflow, ready to import
```

---

## Prerequisites

- Docker and Docker Compose (or an existing self-hosted n8n instance)
- PostgreSQL 13 or newer
- A GitHub account with admin access to the repositories you want reviewed (to add webhooks)
- A GitHub personal access token (fine-grained recommended)
- An OpenAI API key
- A Telegram bot from [@BotFather](https://t.me/BotFather) and a chat or group for alerts
- A public **HTTPS** URL for n8n (required by GitHub and Telegram webhooks)

---

## Installation

### 1. Set up the infrastructure

Example `docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: change_to_a_strong_password
      POSTGRES_DB: n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data

  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_USER: n8n
      DB_POSTGRESDB_PASSWORD: change_to_a_strong_password
      GENERIC_TIMEZONE: Asia/Jakarta
      WEBHOOK_URL: https://n8n.your-domain.com/
      # needed by the signature check in the "Verify Signature" Code node
      GITHUB_WEBHOOK_SECRET: change_to_a_long_random_string
      NODE_FUNCTION_ALLOW_BUILTIN: crypto
      N8N_BLOCK_ENV_ACCESS_IN_NODE: "false"
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - postgres

volumes:
  postgres_data:
  n8n_data:
```

Generate a secret with `openssl rand -hex 32`.

> **Why these three variables?** The Code node must be allowed to use Node's `crypto` module, and to read `GITHUB_WEBHOOK_SECRET` from the environment. Newer n8n versions block environment access inside Code nodes by default. Turning that off means every Code node in your instance can read all environment variables, so only let people you trust edit workflows. If this trade-off is unacceptable, put the signature check in a small reverse-proxy or sidecar service and keep the rest of the workflow unchanged.

### 2. Create the tables

```bash
docker compose exec postgres psql -U n8n -c "CREATE DATABASE devops;"
docker compose exec -T postgres psql -U n8n -d devops < sql/schema.sql
```

Use `-d n8n` and skip the first command if you prefer a single database.

### 3. Create a GitHub token

Create a [fine-grained personal access token](https://github.com/settings/personal-access-tokens) limited to the repositories you want reviewed:

| Permission | Access | Used for |
|---|---|---|
| Pull requests | Read and write | Reading the diff and posting the review comment |
| Contents | Read | Reading private repositories |
| Metadata | Read | Required by GitHub |

A classic token with the `repo` scope also works, but grants far more access than needed.

### 4. Create credentials in n8n

| Credential | Main fields |
|---|---|
| GitHub API | The access token from step 3 |
| OpenAI API | API key |
| Telegram API | Bot token |
| Postgres | Host `postgres`, database `devops` (or `n8n`), user, password |

### 5. Import the workflow

1. In n8n choose **Workflows → Import from File** and select `workflows/devops_pr_review_uptime.json`.
2. Attach the matching credential to every node that needs one.
3. Replace `REPLACE_WITH_ALERT_CHAT_ID` in **Send PR Alert**, **Send Uptime Alert**, and **Send Error Alert**. (Get your chat ID from [@userinfobot](https://t.me/userinfobot). Group IDs are negative.)
4. Open **Workflow Settings** and set **Error Workflow** to this same workflow.
5. Switch the workflow to **Active**.

### 6. Add the GitHub webhook

In each repository (or organization) go to **Settings → Webhooks → Add webhook**:

| Field | Value |
|---|---|
| Payload URL | `https://n8n.your-domain.com/webhook/github-pr-review` |
| Content type | `application/json` |
| Secret | The same value as `GITHUB_WEBHOOK_SECRET` |
| Events | **Let me select individual events**, then **Pull requests** only |

GitHub sends a `ping` immediately. In **Recent Deliveries** you should see `200`. That confirms the signature check works end to end.

> The production URL contains `/webhook/`. The `/webhook-test/` URL only works while you are listening in the n8n editor.

### 7. Add monitors

```sql
INSERT INTO monitors (name, url, expected_status, expected_text, failure_threshold) VALUES
    ('Website',    'https://example.com',            200, 'Example Domain', 2),
    ('API health', 'https://api.example.com/health', 200, NULL,             2);
```

The next scheduled run starts checking them.

---

## Workflow Configuration

| Location | Placeholder | Replace with |
|---|---|---|
| **Send PR Alert**, **Send Uptime Alert**, **Send Error Alert** | `REPLACE_WITH_ALERT_CHAT_ID` | Telegram chat ID for alerts |
| All credentialed nodes | `REPLACE_ID` | Select the credentials you created |

Adjustable parameters:

| Parameter | Where | Default |
|---|---|---|
| Repository allow-list | **Parse PR Event**, `ALLOWED_REPOS` | empty (any repository that can reach the webhook) |
| Reviewed PR actions | **Parse PR Event**, `ACTIONS` | opened, synchronize, reopened, ready_for_review |
| Skip label | **Parse PR Event** | `skip-ai-review` |
| Skip bot authors | **Parse PR Event** | on (remove that line to review Dependabot and similar) |
| Ignored file patterns | **Prepare Diff**, `SKIP_PATH` | lock files, `*.min.*`, `dist/`, `vendor/`, `node_modules/`, images, archives |
| Secret patterns | **Prepare Diff**, `SECRET_PATTERNS` | 7 patterns (see below) |
| Size caps | **Prepare Diff**, `MAX_FILE`, `MAX_TOTAL` | 6,000 and 30,000 characters |
| Max findings in a comment | **Build Review Comment** | 12 |
| Review model and temperature | **Chat Model** | `gpt-4o-mini`, 0.1 |
| Review instructions | **Review PR (LLM)**, system message | see node |
| Uptime interval | **Uptime Schedule** | every 5 minutes |
| Checks run in parallel | **Run Checks**, `BATCH` | 10 |
| Per-monitor settings | `monitors` table | status 200, 10 s timeout, 2 failures to go down |

---

## Usage

### What a pull request gets

```markdown
## AI Review (advisory)

**Risk: HIGH** | 1 blocker(s) | 1 warning(s) | 0 nit(s)

> Adds a bulk import endpoint. The SQL query is built from user input.

### Possible secrets detected
- `src/config.js:14` AWS access key ID. Rotate it and remove it from git history.

### Blockers
- **`src/import.js:42`** [security] User-supplied table name is concatenated into the SQL string, which allows SQL injection.
  - _Suggestion:_ Validate the table name against an allow-list or use a parameterized query.

### Warnings
- **`src/import.js:77`** [error_handling] Errors from the batch insert are swallowed.
  - _Suggestion:_ Log and rethrow so the caller can retry.

Commit `a1b2c3d` | 4 file(s) reviewed, 1 skipped | Automated and advisory. Line numbers are approximate and findings may be wrong.
```

On the next push to the same PR, this comment is edited, not duplicated.

### Telegram alerts

```
PR REVIEW - HIGH RISK

acme/api#7: Add bulk import endpoint
by dev into main

Blockers: 1 | Warnings: 1 | Possible secrets: 1
https://github.com/acme/api/pull/7
```

```
DOWN: API health
https://api.example.com/health
Reason: Expected HTTP 200 but got 503
Failed 2 check(s) in a row
```

```
RECOVERED: API health
https://api.example.com/health
Downtime: ~12 min
```

### Controlling reviews

- Add the label **`skip-ai-review`** to a PR to skip it.
- Draft PRs are skipped until they are marked ready for review.
- To re-review a commit, delete its row and redeliver the webhook from GitHub:

```sql
DELETE FROM pr_reviews WHERE repo = 'acme/api' AND pr_number = 7 AND head_sha = '<sha>';
```

### Managing monitors

```sql
-- pause a monitor
UPDATE monitors SET enabled = FALSE WHERE name = 'Website';

-- require 3 failures before alerting
UPDATE monitors SET failure_threshold = 3 WHERE name = 'API health';

-- check for a keyword in the response
UPDATE monitors SET expected_text = 'ok' WHERE name = 'API health';

-- current state
SELECT name, status, last_status_code, last_latency_ms, last_checked_at FROM monitors ORDER BY name;

-- 30-day availability
SELECT * FROM monitor_availability_30d ORDER BY availability_percent;

-- recent incidents
SELECT m.name, i.started_at, i.ended_at, i.error
FROM incidents i JOIN monitors m ON m.id = i.monitor_id
ORDER BY i.started_at DESC LIMIT 20;

-- review statistics
SELECT risk_level, COUNT(*) FROM pr_reviews WHERE status = 'completed' GROUP BY 1;
```

---

## Workflow Details

### Pull request flow

| Node | Purpose |
|---|---|
| **GitHub Webhook** | Receives the event. Responds through a later node, and keeps the raw request body for signature checking. |
| **Verify Signature** | Computes `HMAC-SHA256(secret, raw body)` and compares it with `X-Hub-Signature-256` using `timingSafeEqual`. Fails closed if the secret or body is unavailable. |
| **Signature Valid? / Respond 200 / Respond 401** | Invalid requests get `401` and stop. Valid ones get `200` immediately, so GitHub never waits for the review. |
| **Parse PR Event** | Decides whether to review (action, draft, label, bot, repository name, allow-list) and extracts repo, number, title, author, base branch, and head SHA. |
| **Claim Review** | `INSERT ... ON CONFLICT DO NOTHING`. A duplicate delivery returns no row and ends the run. |
| **Fetch PR Diff** | GET the pull request with the `diff` media type. Continues on error so a failure becomes an alert. |
| **Prepare Diff** | Splits the diff per file, tracks new-file line numbers through hunk headers, scans added lines for secrets, redacts them, drops lock, generated, and binary files, and applies size caps. |
| **Has Reviewable Changes? / Skip Notice** | Lock-file-only PRs end silently. A failed diff fetch alerts you. |
| **Review PR (LLM)** | Basic LLM Chain with a Structured Output Parser. The PR text and diff are labeled untrusted. Continues on error. |
| **Build Review Comment** | Validates the model output, filters invented files, escapes text, applies risk rules, and renders the markdown. |
| **Should Post? / Review Failed Alert** | If the AI failed and no secrets were found, nothing is posted and you are alerted. If secrets were found, the secret scan is posted anyway. |
| **List PR Comments / Find Existing Comment** | Looks for a comment containing the hidden marker. |
| **Update Comment / Create Comment** | `PATCH` the existing comment, or `POST` a new one. |
| **Mark Review Done** | Stores risk, findings, blockers, and secrets count on the claimed row. |
| **Needs Alert? / Format PR Alert / Send PR Alert** | Telegram alert for high risk, blockers, or secrets. |

### Uptime flow

| Node | Purpose |
|---|---|
| **Uptime Schedule** | Every 5 minutes. |
| **Load Monitors** | Reads enabled monitors. |
| **Run Checks** | Sends the requests (10 at a time), measures latency, checks status code and optional expected text, and packages the results as one JSON array. Non-HTTP URLs are rejected before any request. |
| **Save Check Results** | One statement: updates every monitor, opens an incident on the transition to `down`, closes it on recovery, and returns only transitions. |
| **Format Uptime Alert / Send Uptime Alert** | One grouped Telegram message for all changes in that run. |

### Review rules

| Rule | Behavior |
|---|---|
| Secret in an added line | Risk is forced to `high`, an alert is sent, and the finding is listed with file and line. |
| Any `blocker` finding | Risk is forced to `high`. |
| Finding for a file that is not in the diff | Dropped. |
| Unknown severity, category, or risk value | Finding dropped, or the whole review treated as failed. |
| More than 12 findings | Kept by severity, then truncated. |
| Model text | HTML escaped, `@mentions` neutralized, backticks and line breaks removed. A forged marker cannot hijack the comment. |

**Secret patterns:** AWS access key ID, private key header, GitHub token, Slack token, Google API key, OpenAI-style key, and generic `api_key` or `password` or `token` assignments to a quoted string of 12 or more characters.

---

## Database Schema

```mermaid
erDiagram
    monitors ||--o{ incidents : "has"
    monitors {
        int id PK
        text name UK
        text url
        int expected_status
        text expected_text
        int timeout_ms
        int failure_threshold
        bool enabled
        text status
        int consecutive_failures
        timestamptz last_checked_at
        int last_status_code
        int last_latency_ms
        text last_error
    }
    incidents {
        bigint id PK
        int monitor_id FK
        timestamptz started_at
        timestamptz ended_at
        text error
    }
    pr_reviews {
        bigint id PK
        text repo
        int pr_number
        text head_sha
        text status
        text risk_level
        int findings_count
        int blockers
        int secrets_found
        timestamptz created_at
        timestamptz completed_at
    }
```

- `pr_reviews` has `UNIQUE (repo, pr_number, head_sha)`, which is what makes webhook redeliveries harmless.
- `incidents` has a partial unique index allowing at most one ongoing incident per monitor.
- `monitor_availability_30d` is a view that derives availability and downtime minutes from incidents.

---

## Security

- **Signature first.** Nothing is fetched, stored, or sent to the LLM until the HMAC matches. Comparison is constant time and fails closed.
- **Least-privilege token.** Use a fine-grained GitHub token limited to the needed repositories and permissions. It never leaves n8n's encrypted credential store.
- **Environment access trade-off.** The signature check needs `NODE_FUNCTION_ALLOW_BUILTIN=crypto` and `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`. Any Code node in that instance can then read environment variables. Restrict who can edit workflows.
- **Untrusted code and text.** PR titles, descriptions, and diffs can contain prompt injection. The prompt labels them untrusted, the model has no tools, and its output is validated, filtered, and escaped. At worst a malicious PR produces a misleading comment. It cannot approve, merge, or change anything.
- **Secrets never reach the model.** Matches are redacted before the diff is sent. Only file, line, and type are posted, never the secret value. If a real secret is found, rotate it. Removing it from the PR is not enough, because it stays in git history.
- **Code goes to the LLM provider.** The reviewed diff (up to 30,000 characters per PR) is sent to OpenAI. Do not enable this on repositories whose code you cannot send to a third party. Use `ALLOWED_REPOS` to restrict it.
- **Monitors are trusted input.** The `monitors` table decides which URLs n8n requests, including internal addresses. Only administrators should be able to write to it.
- **Parameterized SQL.** Repository names, SHAs, and check results are passed as query parameters.
- **Never commit credentials.** The workflow file contains placeholder credential IDs only.

---

## Testing

What has been verified for this repository:

- **Code nodes: 70 mock-data checks.** Signature verification (valid, wrong digest, wrong length, tampered body, missing header, missing secret, blocked environment access, unreadable body), event filtering, diff parsing (file filtering, line tracking, size caps, `+++` inside a hunk), secret scanning and redaction, review validation (invented files, invalid values, escaping, forged marker, risk rules, LLM failure with and without secrets), existing-comment lookup, alert formatting, and uptime checks with mocked HTTP (status, expected text, timeouts, invalid URLs, batching).
- **SQL: 30 checks on a real PostgreSQL engine.** The schema applies twice. Review claiming is idempotent per commit. The uptime state machine is exercised through blip, threshold, down, repeat failure, recovery with downtime, and unknown to down. Incidents, the unique open-incident index, constraints, and the availability view are verified.
- **Workflow JSON:** valid, unique node names, every connection and every `$('Node')` reference points to an existing node, query parameter counts match, and the `401` path stops processing.

The tests found and fixed two real bugs during development: a null downtime shown as "~0 min", and the availability view reporting 0% for monitors that never had an incident.

Not covered: the full flow against live GitHub, OpenAI, and Telegram. Before relying on it:

1. Send a GitHub `ping` (the webhook is created with one) and confirm `200`.
2. Open a test PR that adds a fake AWS key such as `AKIAABCDEFGHIJKLMNOP` and a SQL string concatenation, and confirm the comment and alert.
3. Push another commit and confirm the comment is **edited**, not duplicated.
4. Add a monitor for a URL you control, stop the service, and confirm the DOWN and RECOVERED alerts.

---

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| GitHub shows `401` for every delivery | The signature does not match. Check the secret in GitHub and `GITHUB_WEBHOOK_SECRET`, and confirm the content type is `application/json`. A proxy that rewrites the body also breaks the signature. |
| Delivery returns `401` and the execution shows "GITHUB_WEBHOOK_SECRET is not available" | The env var is not set, or `N8N_BLOCK_ENV_ACCESS_IN_NODE` is still `true`. Restart n8n after changing the environment. |
| Error "Cannot find module 'crypto'" or "Module 'crypto' is disallowed" | Set `NODE_FUNCTION_ALLOW_BUILTIN=crypto` and restart n8n. |
| Execution shows "raw body unavailable" | Your n8n version stores the raw body under a different binary property. Open the **GitHub Webhook** execution and check the binary tab, then adjust the property name `data` in **Verify Signature**. |
| Nothing happens on a PR | It was skipped. Drafts, bot authors, the `skip-ai-review` label, closed or edited events, and repositories outside `ALLOWED_REPOS` are ignored. Check the **Parse PR Event** output for `reason`. |
| Review runs but no comment appears | The token lacks Pull requests write permission (`403` or `404` on the comment call), or the AI failed and no secrets were found. In that case you get a Telegram alert instead. |
| `404` when fetching the diff | The token cannot see the repository. Add the repository to the fine-grained token. |
| `406` or "diff too large" | GitHub refuses very large diffs. You get a "diff could not be fetched" alert. Review that PR manually. |
| A new comment is posted every push | The existing comment was not found. Likely more than 100 comments on the PR, or someone edited out the hidden marker. |
| The same commit is not re-reviewed | By design. Delete its `pr_reviews` row and redeliver the webhook. |
| Secret scanner reports a false positive | Test values and placeholders can match. Tighten the pattern in **Prepare Diff** or ignore it. The comment only reports file, line, and type. |
| Review quality is weak | Use a stronger model in **Chat Model**, raise `MAX_FILE`, or add project-specific rules to the system message. |
| No uptime alerts although a site is down | The monitor is disabled, or it has not yet failed `failure_threshold` times in a row. Check `SELECT * FROM monitors`. |
| Repeated DOWN alerts | They should not repeat. If you see them, the monitor is flipping between up and down. Raise `failure_threshold`. |
| Everything is reported down | n8n cannot reach the internet or DNS from its container. Test from inside the container. |
| `relation "monitors" does not exist` | `schema.sql` was not run on the database used by the Postgres credential. |
| Error alerts never arrive | **Workflow Settings → Error Workflow** is not set to this workflow. |
| Version warnings after import | Your n8n version differs. Open the node, re-enter its parameters, or add the node again from the panel. |

---

## Limitations

- **Summary comment only.** The reviewer posts one comment, not inline review comments on specific lines. Line numbers come from the model and are approximate.
- **Advisory only.** It does not approve, request changes, or block merges.
- **Added lines only.** The secret scanner looks at lines added in this PR, not the repository history or existing files.
- **Regex-based secret detection.** It catches common formats, not every secret, and can produce false positives.
- **Large PRs are partially reviewed.** Files beyond the size caps are skipped, and the comment says so.
- **Context is the diff.** The model does not see the rest of the repository, so it can miss problems that depend on code outside the diff.
- **Comment lookup covers the first 100 comments** on a PR.
- **Uptime is single-location HTTP.** One vantage point (your n8n server), GET requests only, no TLS expiry checks, no latency alerts, and 5-minute granularity.
- **The monitor shares fate with n8n.** If n8n or its server is down, nothing is checked and nothing is alerted. Monitor n8n itself from outside.
- **No authentication on monitored URLs.** Checks send plain GET requests without custom headers.
- **Not load-tested.** Designed for small and medium teams.

---

## Roadmap

- [ ] Inline review comments on specific lines via the GitHub review API
- [ ] Custom review rules per repository (a `.ai-review.yml` file)
- [ ] Fetch surrounding file context for changed functions
- [ ] Commit status or check run (`success`, `neutral`, `failure`) based on risk
- [ ] Slack and Discord alert channels
- [ ] TLS certificate expiry and latency threshold monitors
- [ ] Custom headers, methods, and authentication for monitors
- [ ] Multi-region checks from several n8n instances
- [ ] Public status page driven by the `incidents` table
- [ ] Optional auto-restart of a failing service (SSH or Docker API) with a human approval step
- [ ] Weekly digest: review statistics, availability, and top incident causes
- [ ] Queue mode deployment (Redis and workers) for higher volume

---

## License

Choose a license that fits your needs (for example MIT) and add a `LICENSE` file to the repository.

---

## Contributing

Issues and pull requests are welcome. For larger changes, please open an issue first so the plan can be discussed.
