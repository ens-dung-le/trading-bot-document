# CLARITY Act - two-developer Git worktree implementation plan

> **Archived implementation plan.** The official-Senate lane described below is no longer active
> runtime scope. The active SOP-aligned design is RSS ingestion -> durable advisory intent ->
> Telegram delivery, with optional market-impact research. See `plan-clarity-act-telegram-alert.md`
> Sections 1 and 2 for the current contract.

Goal: implement the CLARITY Act official-alert and news-impact tracks in parallel without either
developer editing the other track's files or coupling analytics to trading execution.

Out of scope: automated position opening, closing, resizing, or using a news score as a trading
signal.

## 1. Required checkpoint before creating worktrees

The current CLARITY files are uncommitted. A new worktree created from `HEAD` will not contain them.
The integration owner must first review and commit one shared foundation checkpoint containing:

- migrations `0059` and `0060`;
- the current `src/event-alert/` and `src/news-impact/` scaffolding;
- the two worker entrypoints and healthchecks;
- the parser dependencies already recorded in `package.json` and `package-lock.json`;
- the Compose profile and tracked environment examples;
- the current CLARITY plan and this ownership plan.

Do not create either feature worktree from the old `local-test` `HEAD`. Do not use `git stash` as the
shared baseline: a stash is local mutable state and does not give both developers an auditable common
commit.

Suggested checkpoint commit:

```text
clarity-alert: checkpoint shared worker contracts
```

## 2. Worktree setup

Run these commands only after the checkpoint commit exists on `local-test`:

```bash
mkdir -p /Users/MAC/ai-consensus-trading-bot-worktrees

git worktree add \
  /Users/MAC/ai-consensus-trading-bot-worktrees/clarity-official \
  -b feature/clarity-official-alert \
  local-test

git worktree add \
  /Users/MAC/ai-consensus-trading-bot-worktrees/clarity-news-impact \
  -b feature/clarity-news-impact \
  local-test

git worktree list
```

Worktree ownership:

| Member | Branch | Worktree |
|---|---|---|
| Dev A | `feature/clarity-official-alert` | `/Users/MAC/ai-consensus-trading-bot-worktrees/clarity-official` |
| Dev B | `feature/clarity-news-impact` | `/Users/MAC/ai-consensus-trading-bot-worktrees/clarity-news-impact` |

The original worktree remains the integration worktree and stays on `local-test`.

### Parallel delivery waves

| Wave | Dev A | Dev B | Integration checkpoint |
|---|---|---|---|
| 0 | Review official contracts | Review advisory contracts | Commit the shared foundation and create both worktrees |
| 1 | Real Senate fixtures, classifier, state, priority outbox | Durable RSS ingestion, dedupe, base advisory intent | Official and unscored advisory fixture flows pass independently |
| 2 | Detail retry, schedule changes, correction alerts, health | Causal Gaussian context, outcomes, matched controls, queues | Official delivery stays healthy with analytics disabled or backlogged |
| 3 | Operator CLI, watchdog, recovery and deployment drills | Candidate/challenger/active-shadow lifecycle and score intents | Full suite, isolation gates, canary, and soak readiness pass |

Each wave is handed off as reviewable commits from the same two worktrees. Do not wait until every
phase is finished to create one large merge: Wave 1 establishes the event-critical MVP before the
longer analytics work continues.

## 3. Frozen cross-team contract

Both tracks share PostgreSQL but not application modules. This contract is frozen at the checkpoint:

```text
news-impact worker
  -> news_notification_intents
  -> clarity-alert-worker bridge
  -> event_notification_outbox
  -> Telegram
```

### Advisory intent contract

Dev B may insert into `news_notification_intents` only after the normalized live news event is
committed. Dev A is the sole owner of bridging an intent into the Telegram outbox.

Required fields:

```text
id             durable database identity
news_event_id  reference to one normalized logical news event
dedupe_key     news:<logical_event_id>:base:v<message_contract_version>
message_text   already bounded and HTML-escaped advisory message
created_at     database timestamp
bridged_at     null until Dev A commits the matching outbox handoff
```

Required behavior:

- Dev B never writes `event_notification_outbox` and never receives a Telegram token.
- Dev A never mutates news observations, classifications, outcomes, statistics, or models.
- The bridge and `bridged_at` update are one database transaction.
- Duplicate intent or bridge execution produces one logical outbox row.
- Missing or degraded analytics never blocks official Senate polling or delivery.
- Historical/backfilled events never create live notification intents.

### Message ownership

| Message | Producer | Sender |
|---|---|---|
| Confirmed Senate result | Dev A official lane | Dev A alert worker |
| Schedule change | Dev A official lane | Dev A alert worker |
| Needs review/correction | Dev A official lane | Dev A alert worker |
| Monitoring canary | Dev A operator path | Dev A alert worker |
| Unconfirmed news advisory | Dev B intent writer | Dev A alert worker |
| Shadow-context follow-up | Dev B active-shadow scorer | Dev A alert worker |

Any required change to this contract is proposed as a small integration commit before either member
continues against the changed version. Neither member edits the other member's migration to change the
contract silently.

## 4. Dev A - official Senate alert and Telegram delivery

### File ownership

Dev A owns:

```text
src/event-alert/**
scripts/run-clarity-alert-worker.ts
scripts/check-clarity-alert-health.ts
migrations/operational/0059_clarity_act_alert_pipeline.sql
.env.clarity-alert.example
```

Dev A must not edit `src/news-impact/**` or migration `0060`.

### A1. Real Senate fixtures and adapters

1. Capture representative real vote-menu XML/HTML and vote-detail XML/HTML fixtures.
2. Parse Congress/session metadata inherited from the document root as well as row-local fields.
3. Support allowlisted same-host redirects and fail closed on redirect to another host.
4. Distinguish unavailable source, valid empty response, parse-contract failure, and detail publication
   lag.
5. Match only H.R. 3633 cloture on the motion to proceed above the persisted arming baseline.

Exit gate: positive fixtures parse successfully; final passage, amendments, old votes, wrong bills,
unknown results, and malformed pages do not match. A syntactically valid page yielding no parseable
vote is unhealthy rather than a successful empty poll.

### A2. Official event state machine

1. Persist menu and detail observations independently with payload hashes and source URLs.
2. Implement `AWAITING_DETAIL` with retry offsets `0`, `2`, `5`, and `10` seconds plus the configured
   confirmation deadline and grace.
3. Require menu/detail agreement before `PASSED` or `FAILED`.
4. Create `NEEDS_REVIEW` for disagreement, unknown result, overdue detail, or terminal correction.
5. Preserve the last confirmed result when recording disputed/correction evidence.
6. Add schedule discovery and `SCHEDULED`, `RESCHEDULED`, `POSTPONED`, and `CANCELLED` transitions.

Exit gate: every committed state transition has immutable evidence and history; a menu-only candidate
can never create a confirmed-result alert.

### A3. Durable Telegram outbox

1. Add explicit priority so confirmed results/corrections are claimed before advisory messages.
2. Commit state, history, evidence links, and the unique notification in one transaction.
3. Keep confirmed results and corrections non-expiring.
4. Parse Telegram `429 retry_after`, retry timeouts/5xx, and classify permanent configuration errors.
5. Fence delivery completion by claim token/version and verify that the conditional update affected
   one row.
6. Mark readiness false when a critical outbox row is overdue or Telegram identity/permission checks
   fail.

Exit gate: restart, stale lease, concurrent claim, `429`, `5xx`, malformed success JSON, and
post-send/pre-commit crash tests cannot lose a confirmed message. Advisory backlog cannot delay a
confirmed result.

### A4. Worker operations

1. Keep session-level advisory leadership on a reserved PostgreSQL connection.
2. Run official polling, advisory bridging, and delivery as independent failure lanes.
3. Add signal-aware shutdown and durable health updates after complete cycles.
4. Add authenticated operator commands for arm, status, canary, audited override, resolution, and
   stop.
5. Add heartbeat integration for an independently hosted watchdog.
6. Prove the runtime environment and database role cannot access exchange credentials, orders, or
   positions.

Exit gate: kill/restart recovery, second-worker leadership rejection, source outage, Telegram outage,
database reconnect, and Docker health tests pass.

### Dev A verification

```bash
npm run typecheck
npx vitest run src/event-alert
npm run build
docker compose config --quiet
```

Dev A handoff must include the exact source-probe result and must not claim production readiness while
the Senate endpoint remains inaccessible or the real fixtures do not pass.

## 5. Dev B - advisory ingestion and news-impact analytics

### File ownership

Dev B owns:

```text
src/news-impact/**
scripts/run-news-impact-worker.ts
scripts/check-news-impact-health.ts
migrations/operational/0060_news_impact_analytics.sql
.env.news-impact.example
```

Dev B must not edit `src/event-alert/**`, migration `0059`, or call Telegram directly.

### B1. Durable live advisory ingestion

1. Complete the Google News RSS adapter at a 30-second cadence.
2. Normalize source ID, canonical URL, publisher, title, bounded summary, publication time, first-seen
   time, payload hash, and timing quality.
3. Deduplicate repeat observations and syndicated copies into a stable logical event.
4. Store every valid approved-feed observation, including unmatched items needed for later taxonomy
   evaluation.
5. Create one base advisory intent only for a live event matching the frozen notification rule.
6. If Make.com/Zapier is selected, implement raw-body HMAC, timestamp, UUID-v4 or >=16-byte base64url
   nonce validation, replay window, schema, size limit, rate limit, and replay protection.

Exit gate: RSS and webhook representations normalize to the same event contract; restarts and
re-signing do not duplicate logical events or intents; historical rows cannot alert.

### B2. Classification and causal market alignment

1. Persist versioned deterministic keyword/category classification.
2. Freeze event time, first-seen evidence, classifier version, source reliability, and taxonomy
   version on each job.
3. Read only closed candles available at the event cutoff.
4. Freeze Gaussian `4h` structure plus `1h` confirmation using the existing classifier contract.
5. Mark missing, late, or gapped candle context unavailable instead of backfilling future knowledge.
6. Compute approved `15m`, `1h`, `4h`, and `24h` outcomes plus MFE, MAE, volatility lift, regime
   transitions, matched controls, and abnormal returns.

Exit gate: replaying the same event and frozen versions produces identical context and outcomes, with
no look-ahead candle access.

### B3. Durable analytics queues

1. Add separate high- and low-priority job classes with leases, claim tokens, retry scheduling, dead
   letters, and audited replay generations.
2. Reserve capacity for live ingestion, classification, scoring, and advisory intent creation.
3. Pause backfill, recalibration, and reports automatically under queue or resource pressure.
4. Freeze every algorithm/config/dependency version on enqueue; incompatible retries dead-letter
   instead of silently using new logic.

Exit gate: process crashes and concurrent workers do not duplicate outcomes, statistics, model builds,
or notification intents; low-priority work cannot starve live ingestion.

### B4. Candidate, challenger, and active-shadow lifecycle

1. Build immutable per-horizon candidate artifacts using walk-forward validation and embargo.
2. Record baseline comparison, calibration, sample size, concentration, stability, and drift evidence.
3. Permit only an authenticated analytics operator to move `candidate -> challenger`.
4. Score challengers into audit rows only during the approved post-cutoff shadow window.
5. Promote or roll back one complete horizon bundle with a fenced `pointer_version` transaction.
6. Create a scored follow-up intent only when the frozen job role is `active_shadow` and a conditional
   pointer check plus intent insert succeeds in the same transaction.
7. Fall back to the base unscored advisory whenever context, model, bundle, or pointer validation
   fails.

Exit gate: challengers can never create Telegram intents; concurrent promotion cannot let a stale
active job create an intent; rollback is atomic and fully audited.

### Dev B verification

```bash
npm run typecheck
npx vitest run src/news-impact
npm run build
docker compose config --quiet
```

Dev B handoff must identify which work is live-advisory-ready and which remains historical/shadow-only.
No model is called `active_shadow` before the approved sample and live-shadow gates pass.

## 6. Files reserved for the integration owner

Neither feature member edits these files during parallel development:

```text
package.json
package-lock.json
docker-compose.yml
.gitignore
docs/plan-clarity-act-telegram-alert.md
docs/clarity-act-two-developer-worktree-plan.md
```

If a member needs a dependency, Compose variable, shared contract, or spec correction, they record the
request in their PR description. The integration owner applies all accepted shared-file changes once,
after both feature branches are ready. This avoids resolving the same lockfile or Compose block twice.

## 7. Commit and handoff rules

Each commit stays inside the member's ownership boundary. Recommended commit shapes:

```text
clarity-official: parse real Senate vote fixtures
clarity-official: persist confirmed result and priority outbox
clarity-official: harden Telegram delivery health

news-impact: normalize durable RSS events
news-impact: align events with causal Gaussian context
news-impact: add versioned shadow model lifecycle
```

Before handoff, each member supplies:

1. commit hashes in merge order;
2. tests executed and their results;
3. migrations added or changed;
4. configuration inputs still required;
5. known gaps and explicit no-go conditions;
6. confirmation that no out-of-ownership file was changed.

## 8. Integration order

Use the original `local-test` worktree for integration:

1. Verify both worktrees are clean and every handoff commit is pushed or otherwise recoverable.
2. Merge Dev A first so the official alert path and sole Telegram authority are established.
3. Run migration `0059` against a disposable test database and execute Dev A's database tests.
4. Merge Dev B without resolving conflicts by copying whole files from either branch.
5. Run migrations `0059` then `0060` against a fresh disposable database.
6. Apply requested dependency and Compose changes once in the integration worktree.
7. Run the complete typecheck, test suite, build, Compose validation, credential-isolation checks, and
   end-to-end fixture flow.
8. Run official result tests with analytics disabled, then advisory tests with official polling
   simulated unavailable, proving the lanes remain independent.
9. Apply migrations to the real database only after the disposable-database and rollback/runbook
   checks pass.
10. Provision secrets, run the production-host Senate/RSS probes, send a real Telegram canary, and
    begin the required soak.

Suggested integration commands after reviewing the commit hashes:

```bash
git switch local-test
git merge --no-ff feature/clarity-official-alert
git merge --no-ff feature/clarity-news-impact

npm run typecheck
npm test
npm run build
docker compose config --quiet
```

## 9. Independent completion gates

Dev A is complete when the confirmed official-result track can operate with the entire news-impact
worker disabled and still pass source, restart, delivery, health, and isolation tests.

Dev B is complete when ingestion, historical outcomes, and model lifecycle tests pass without a
Telegram token, exchange credentials, or write access to the Telegram outbox. Its only live delivery
output is an immutable notification intent consumed by Dev A.

Integration is complete only when both independent gates pass and the intent bridge works without
allowing analytics failure or backlog to delay a confirmed official result.
