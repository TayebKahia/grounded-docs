# Architecture

How Grounded Docs is put together, and why. The README covers what it does and how it was measured; [CODE_SAMPLES.md](CODE_SAMPLES.md) shows some of the code.

## Vocabulary

| Term | Meaning |
| --- | --- |
| **Document** | One file in the Document Source (a PDF, a Word file, a Google Doc). |
| **Document Source** | Where Documents live: a local folder or a Google Drive folder. |
| **Sync** | The process that keeps the Index in step with the Document Source. |
| **Index** | The Postgres database: Documents, their chunks, vectors, keyword indexes and Sync state. |
| **Question / Standalone Question** | What the person asked; a Follow-up is rewritten into a Standalone Question before it is searched. |
| **Answer** | The verified result of a Question: an Outcome, claims with Citations, Conflicts, what is missing. |
| **Outcome** | *Answered*, *Partially Answered* or *Not Found*. |
| **Citation / Excerpt** | Where a claim comes from (Document, Location, link) and the verbatim text that supports it. |
| **Location** | A page (PDF) or a heading path (Word, Google Docs) inside a Document. |
| **Conflict** | Two current, independent Documents that disagree; both sides are shown and cited. |
| **Unreadable** | A Document Sync cannot read: password-protected, scanned, too large or an unsupported type. |
| **Session** | The previous turns of a conversation. It lives in the browser tab or the Slack thread, never on the server. |
| **Gatekeeper** | The one door every Question passes, from the web and from Slack: rate limits, operator controls, the provider check, the concurrency cap, then the Assistant under the overall timeout. |
| **Refusal** | A Question the Gatekeeper turned away before any model call, with its reason (`rate_limited`, `overloaded`, `budget_exhausted`, `answering_off`) and a short message for the person. A failure (`timeout`, `provider_unavailable`, `database_unavailable`) is not a refusal. |
| **Answer event** | One row per request, answered, refused or failed: counts, timings, tokens, cost and versions, never any Question or Answer text. |
| **Operator controls** | Values the operator changes live with the `controls` command: the kill switch (answering on or off, with a message), the daily budget (a spending limit reset at 00:00 UTC) and the rate limits. `.env` holds their defaults. |
| **Vote / Review text** | A 👍 or 👎 on an Answer; with the opt-in setting, a 👎 also keeps that Answer's text for 30 days for a reviewer. |
| **SLO** | A service-level objective: a ratio of good to valid events (or a cost) and its target over 28 days rolling. Four: availability, latency, freshness, cost. |
| **Guard** | An alert on one of the protections rather than on an SLO: the budget spent or 80% used, a rate-limit spike, answering left off. |
| **Sync cycle** | One run of Sync, recorded as a row: when, which mode, ok or failed, counts or the error's class name. |
| **Eval run** | One answers evaluation, recorded from its summary file as a row in `eval_runs`: its targets, versions and costs. |
| **Synthetic traffic** | Dev Questions asked on a timer by the `synthetic` service, so a demo's dashboards have data; its events are marked and kept out of every SLO. |

## Modules and ports

The core is three deep modules with small interfaces: `Assistant.ask(question, previous_turns) -> Answer`, `Sync.run_once(mode) -> SyncReport` and `Gatekeeper.ask(question, previous_turns, caller) -> Answer | Refused | Failed`, which both interfaces call instead of the Assistant. Each external system sits behind a port (a `typing.Protocol` in `ports.py`); the core never imports an adapter, so tests swap in fakes and providers change by configuration.

```mermaid
flowchart TB
    subgraph "Interfaces"
        API["api: FastAPI app (page, /api/ask, /status, /healthz)"]
        SLACK["slack: Slack bot (Socket Mode)"]
        SYNCSVC["sync: SyncLoop"]
        SYNSVC["synthetic: SyntheticTraffic (optional profile)"]
        CLI["CLI: migrate, dev, sync, reindex, eval, controls"]
    end
    subgraph "Core"
        GATE["Gatekeeper"]
        ASSIST["Assistant"]
        SYNC["Sync"]
        INDEX["Index (all SQL)"]
    end
    subgraph "Ports"
        LM["LanguageModel"]
        EMB["Embedder"]
        SRC["DocumentSource"]
        THREADS["SlackThreads"]
        SINK["AnswerEventSink"]
        CYCLES["SyncCycleSink"]
    end
    subgraph "Adapters"
        OAI["OpenAI (Responses + Embeddings, plain REST)"]
        HEDGE["HedgedLanguageModel (wraps the LanguageModel)"]
        LOCAL["LocalFolderSource"]
        DRIVE["GoogleDriveSource (Drive API v3, plain REST)"]
        SLACKAPI["Slack Web API"]
        FAKES["Fakes (tests and uv run dev)"]
        QUEUED["PostgresEventSink (bounded queue, one writer thread)"]
        CYCLEPG["PostgresSyncCycleSink (one insert per cycle)"]
    end
    PG["Postgres 17 + pgvector"]

    API --> GATE
    SLACK --> GATE
    SYNSVC --> GATE
    SYNCSVC --> CYCLES
    CYCLES -.-> CYCLEPG
    CYCLEPG --> PG
    GATE --> ASSIST
    GATE --> SINK
    SLACK --> THREADS
    SYNCSVC --> SYNC
    CLI --> SYNC
    CLI --> ASSIST
    ASSIST --> LM
    ASSIST --> EMB
    ASSIST --> INDEX
    SYNC --> SRC
    SYNC --> EMB
    SYNC --> INDEX
    INDEX --> PG
    LM -.-> HEDGE
    HEDGE -.-> OAI
    EMB -.-> OAI
    SRC -.-> LOCAL
    SRC -.-> DRIVE
    THREADS -.-> SLACKAPI
    LM -.-> FAKES
    EMB -.-> FAKES
    SINK -.-> QUEUED
    QUEUED --> PG
```

| Module | Responsibility |
| --- | --- |
| `assistant.py` | The answer pipeline: rewrite, search, gate, draft, verify, decide the Outcome. Callers never see chunks, scores or prompts. |
| `sync.py`, `sync_loop.py` | One Sync cycle (incremental or full), and the schedule that runs cycles. |
| `index.py` | Every SQL statement: hybrid search, Document replacement, Sync state, status counts. |
| `parsing.py`, `chunking.py` | PDF and DOCX to Located sections, or a reason the file is Unreadable; sections to chunks of about 600 tokens. |
| `contracts.py` | Pydantic models for everything that crosses a boundary: the Answer, the model's draft, the API request, Sync reports, status. |
| `ports.py` | The ports, and the one failure every provider adapter raises: `ProviderUnavailable`. |
| `adapters/`, `sources/` | OpenAI, Google Drive and local-folder adapters; fakes. |
| `hedging.py` | A LanguageModel wrapper that cuts tail latency on the answer call. |
| `usage.py` | Usage per request: every billed provider call made for one Question, collected for it (a hedge's and an abandoned call's included) and priced. |
| `pricing.py` | The price table and `cost_usd`, shared by the running service (as billed) and the evaluation (cold). |
| `gatekeeper.py` | Every Question's path: rate limits, controls, the slot, the worker, the timeout, the event. Returns an Answer, a `Refused` or a `Failed`; the interfaces only map that to their output. |
| `events.py` | Answer events: the builders (from an Answer and its diagnostics, a refusal or a failure) and `PostgresEventSink`, which writes them off the request's path. |
| `controls.py`, `controls_command.py` | The operator controls: the registry and its validation, `OperatorControls` (cached reads with today's spend), the Postgres store, and the `controls` command. |
| `rate_limits.py`, `refusals.py` | Token buckets with a daily count, per identity, in memory; the refusal wording and HTTP status shared by the web and Slack. |
| `feedback.py`, `recent_answers.py` | Votes (`Feedback` over a `VoteStore` port and its Postgres adapter), and the API's memory of the last hour's Answers, with their text only when review text is on. |
| `retention.py` | The nightly purge of review text, old answer events and their votes, and old Sync cycles. |
| `sync_cycles.py` | `run_recorded`: runs one Sync cycle and records it; `PostgresSyncCycleSink`. |
| `slo.py` | The SLOs and guards: `slo.toml`'s typed model, its loader, and the SQL rendered from it (SLIs, burn rates, alerts, guards). |
| `dashboards.py`, `alerting.py` | The Operations and Quality & Evaluation dashboards, and the alert rules, contact point and notification policy, rendered as Grafana provisioning files. |
| `grafana_reader.py` | The read-only role Grafana connects as: its grants, password and default budget, set by `migrate`. |
| `synthetic.py` | Synthetic traffic: one dev Question per interval through a Gatekeeper of its own. |
| `status.py` | The status report, its Protection part included (`ProtectionReport`). |
| `clock.py` | The injectable clock, and midnight UTC for the budget and the daily limits. |
| `loadtest.py` | The load test's fake providers, replaying recorded call times (never used by a running deployment). |
| `providers.py`, `settings.py` | Wiring: settings from the environment, adapters from settings. The only place that picks an adapter. |
| `api/`, `slack_bot/` | Thin interfaces over the Assistant and the status queries. |
| `evaluation/` | The `eval` harness: retrieval, answer and Sync measurements over the demo corpus; `history.py` loads answers summaries into `eval_runs` (`eval record`). |
| `migrations/` | Numbered SQL files and the `migrate` command. |

## The answer pipeline

`Assistant.ask` enforces grounding in code, in three layers: a relevance gate, a strict structured draft, and word-for-word Excerpt verification.

**Why drop rather than retry:** retrying with feedback on a failed check roughly doubles response time on exactly the hardest Questions, and rejecting the whole Answer throws away claims that did verify. Dropping a failed claim and downgrading the Outcome keeps both speed and the verified part.

```mermaid
flowchart TD
    Q["Question + up to 3 previous turns"] --> RW{"Previous turns?"}
    RW -->|"yes"| R["LLM: rewrite into a Standalone Question"]
    RW -->|"no"| S["Standalone Question"]
    R --> S
    S --> E["Embed the Standalone Question"]
    E --> HS["One SQL query: top 20 keyword + top 20 vector chunks, fused by reciprocal rank"]
    HS --> G{"Gate: best similarity above the threshold, or an exact code matched?"}
    G -->|"no"| NF["Not Found + Closest Documents, no LLM call"]
    G -->|"yes"| D["LLM: strict JSON draft from the top 12 chunks (hedged call)"]
    D --> V["Verify each Excerpt word for word against the chunk it cites"]
    V --> DROP["Remove absence claims; drop failed claims; keep Conflicts with 2+ verified sides"]
    DROP --> O["Outcome from what survived, not from what the model said"]
    O --> P{"Partially Answered?"}
    P -->|"yes"| PC["LLM: do the claims answer any part? If not, Not Found"]
    P -->|"no"| A["Answer"]
    PC --> A
```

- **Search** is hybrid: Postgres full-text (English stems, plain words, and one whole lexeme per code such as `LQA-MSA-2025-014`, so a code does not match its siblings) and pgvector cosine distance over an HNSW index, fused by reciprocal rank in a single query. Chunks are embedded and keyword-indexed together with their Document title and Location.
- **The gate** stops a Question before any model call when nothing retrieved is close enough. The threshold was tuned for `text-embedding-3-small` on the dev split of the evaluation set and is retuned with `eval screen` for another embedding model.
- **The draft** is a strict structured output: an Outcome, claims each carrying a Citation (a chunk number and an Excerpt), Conflicts and missing parts. Free prose is never accepted.
- **Verification** compares each Excerpt, after normalisation (Unicode NFKC, typographic quotes and dashes, whitespace), against the text of the chunk it cites. A claim that fails is dropped and the Answer says some evidence could not be verified. A claim that only says the Documents are silent is removed: the model sees a few chunks, not the Documents, so such a claim cannot be verified. When versions of one document disagree (v3 and v4, DRAFT and FINAL), the latest one answers: FINAL over DRAFT, then the later effective date, then the higher version number. File modification times are not used, because they reflect copying, not which version is in force. Independent current Documents that disagree are shown as a Conflict, both sides cited.
- **No streaming.** Text reaches the person only after it has been verified.

Follow-ups need no server-side history: the browser keeps the Session in `sessionStorage`, the Slack bot reads it back from the thread, and the Assistant uses the latest 3 turns.

## Sync

Sync keeps the Index in step with the Document Source by polling.

**Why polling:** Drive push notifications need a publicly reachable HTTPS webhook, and the assistant is meant to run privately with nothing exposed. Polling every 2 minutes plus a nightly full reconciliation keeps it private; a new Document is findable within about 2 minutes plus indexing time.

**Why a Document Source port:** a public demo cannot ship Drive credentials, so `docker compose up` runs on a local folder with no credentials, while the Drive adapter is fully implemented and tested against a fake Drive. Both report changes in the same terms, so indexing never sees a Drive quirk.

```mermaid
flowchart LR
    LOOP["SyncLoop: every 2 min, full on start and nightly"] --> RUN["Sync.run_once(mode)"]
    RUN --> CUR{"Change cursor usable?"}
    CUR -->|"yes"| INC["Apply Added / Modified / Removed, then retry earlier failures"]
    CUR -->|"no"| FULL["Full reconciliation: diff a complete listing against the Index"]
    INC --> DOC["Per Document: fetch, parse, chunk, embed"]
    FULL --> DOC
    DOC --> TX["Replace the Document in one transaction"]
    DOC --> UNR["Or record it as Unreadable, with the reason"]
    DOC --> FAIL["Or record the failure; the previous version stays searchable"]
```

- **One vocabulary for changes.** Both adapters report Added, Modified and Removed. Drive's quirks (trash, permanent deletion, unsharing, moves into and out of the folder, folders entering the tree, Google Docs exported as DOCX) are translated inside `GoogleDriveSource`; the local folder has no change feed, so every cycle there is a reconciliation by content hash.
- **Atomic replacement.** A Modified Document's old row and chunks are deleted and the new ones inserted in one transaction, so a Question never sees it half-indexed or twice.
- **Failure isolation.** A Document whose fetch, parse or embedding fails is recorded in `sync_failures`, keeps its previous version and is retried every cycle; the change cursor moves on without it. A listing that fails (a missing or unshared folder) fails the cycle rather than removing everything.
- **Embedding model guard.** Vectors from two embedding models are not comparable. Every Document records the model that embedded it; `api` refuses to start and `sync` pauses when the Index holds another model, until `reindex` rebuilds it.

## Protection and measurement

The Assistant answers well, but on its own it runs unguarded: a script or a burst of people can run up a bill nobody sees coming, and a timed-out Question's model call is abandoned, not killed, so it keeps running and costing. Every Question therefore passes one Gatekeeper, and every request leaves one answer event behind.

### The Gatekeeper

```mermaid
flowchart TD
    WEB["api: POST /api/ask (Caller: web, client IP)"] --> RL
    SLACK["slack: mention or DM (Caller: slack, user, workspace)"] --> RL
    subgraph "Gatekeeper.ask"
        RL{"Rate limits: every identity allows?"}
        CTRL{"Controls: answering on? budget not spent?"}
        KEY{"Provider configured?"}
        CAP{"Slot free? (per process)"}
        WORKER["Worker thread: Assistant.ask_measured, usage collected"]
        WAIT["Caller waits up to ASK_TIMEOUT_SECONDS"]
    end
    REFUSED["Refused: rate_limited, answering_off, budget_exhausted, overloaded"]
    FAILED["Failed: provider_unavailable, database_unavailable, timeout"]
    SINK["AnswerEventSink"]
    RECENT["RecentAnswers (api only)"]

    RL -->|"no"| REFUSED
    RL -->|"yes"| CTRL
    CTRL -->|"off, or spent"| REFUSED
    CTRL -->|"spend unknown, budget set"| FAILED
    CTRL -->|"ok"| KEY
    KEY -->|"no"| FAILED
    KEY -->|"yes"| CAP
    CAP -->|"no"| REFUSED
    CAP -->|"yes"| WORKER
    WORKER --> WAIT
    WAIT -->|"in time"| ANSWER["Answer"]
    WAIT -->|"too late"| FAILED
    REFUSED -->|"event at once, cost 0"| SINK
    FAILED -->|"event"| SINK
    WORKER -->|"on finish: event, then slot freed"| SINK
    WORKER -->|"answered in time"| RECENT
```

- **Order.** Rate limits come first, so every Question counts toward its limit, even one a later check refuses. Then the operator controls, then the provider check, then the slot. A refusal makes no model call.
- **The slot is held until the worker really ends.** The worker is a daemon thread (a thread cannot be killed). After a 504 it carries on, keeps its slot and writes its event when it finishes, marked `timed_out`, with the late calls' cost. So the cap counts the model calls still running and costing, not the requests still waiting. A shared flag decides the race between the caller's timeout and the worker's finish, so a 504 always has `timed_out` on its event and an Answer never does.
- **The worker's order** is: build and record the event, register the Answer with `RecentAnswers` (`on_answered(AnsweredQuestion)`, `api` only, skipped after a 504, whose client never got the id), resolve the caller, free the slot. A vote can therefore never arrive before its Answer is known. A failure to build or record the event, or to register, is logged by type and never fails the Answer.
- **Per-request usage.** The worker starts in a copy of the caller's `contextvars` context and opens its own `UsageCollector`. Every billed provider call made for that Question lands there, including a hedge's (`hedging.ThreadRunner` copies the context into its threads too) and an abandoned one's, priced as billed with the evaluation's price table (`pricing.py`). An unpriced model costs NULL, with one warning per process. `hedged` / `hedge_won` come from the hedge decorator through a context variable of their own (`hedging.observing_hedge`).
- **Why a counter, not a queue.** Beyond the cap, a Question is refused at once (`overloaded`) instead of waiting: under a burst, a queue only turns into timeouts that still cost. Slots are a counter under a lock rather than a semaphore, so `in_flight` is readable for `/status`.
- **The cap's value.** `API_MAX_IN_FLIGHT` stays below anyio's 40 worker threads (at most 32). Its default, 16, comes from the load test (below). Each process's Index pool is `max_in_flight + 2`: the 2 spare connections serve the status queries, so the cap, not the pool's 5 s wait, refuses a burst. The event writer, the controls, the votes and the purge each have a connection of their own.

The interfaces only map the result: `api` to a status and an `ErrorResponse` (`{"error", "message"}`, `Retry-After` where the reason has one), Slack to an edit of "Searching…". The wording lives in one table, `refusals.py`, and the page inserts it with `textContent`, never as HTML.

### Answer events

`AnswerEventSink.record(event)` never raises. `PostgresEventSink` puts the event on a bounded queue (`EVENT_QUEUE_SIZE`) and returns; one writer thread inserts it over its own connection (2 s connect and statement timeouts; a connection that died while idle is retried once). A full queue drops the event with a warning and a running count; a failed insert is logged by exception type only, since a database error can quote the row. On SIGTERM `serve` closes the sink, which drains for up to 2 s.

The `request_id` is the id in the logs and the Answer's `answer_id`, so a 👎 or an alert leads to its log lines. The table has no free-text column: a schema test fails if one appears beyond the enumerated identifiers and version strings, and `CHECK` constraints hold every enumerated column. A gated Question, and a failure where both answer calls failed, record `hedged` as NULL. The load test's events are marked `synthetic` (`Versions.synthetic`, passed once to the Gatekeeper with the fake models' names).

### Operator controls

`operator_controls` holds one row per control changed with `controls`; a missing row means the `.env` default (key/value rows, so a new control needs no migration). `OperatorControls.current()` reads every value with its source and today's spend (the sum of `cost_usd` since 00:00 UTC, NULLs ignored) in one query on its own connection, cached for 5 s. Past the cache window one caller reads while the others keep the cached values, so a slow database holds at most one Question. On a failed read the last good values hold for 60 s; after that the `.env` defaults apply with the spend unknown, and with a budget set Questions fail closed (`database_unavailable`): with the spend unknown, nothing proves the budget allows them. With no budget set, answering continues.

`controls` validates a value before saving it, writes the row and an `operator_control_changes` row in one transaction (`SELECT … FOR UPDATE`, a change row only when the value differs), and logs `operator control changed` as a warning. With a budget set, an unpriced configured model pauses `api` and `slack` at start, naming it, and `controls` refuses to set a budget.

### Rate limits

`RateLimiter.check(identities, limits)`: a token bucket per identity (a burst of the per-minute limit passes, then one more every 60/N s) and a daily count reset at 00:00 UTC. Every identity must allow; a Question one of them refuses spends none of them. The limits are live controls read from the same cached snapshot; 0 turns one off, a lowered limit clamps at once, a raised one refills at its rate. Idle identities are swept at most once a minute. The counts live in memory, per process: there is one `api` and one `slack` process, and a restart resets them.

- Web: the client IP. `TRUSTED_PROXY_IPS` (IPs or networks) is passed to uvicorn's `forwarded_allow_ips`; empty, `proxy_headers` is off, so a forged `X-Forwarded-For` changes nothing, even from 127.0.0.1.
- Slack: the user (`event["user"]`; a DM and a channel share it) and the workspace that installed the bot (bolt's `context.team_id`, which in a Slack Connect channel is the installation, not the asker's own team). An event without a user is limited by its workspace alone.
- Votes: `Feedback` has its own limiter per client IP (`feedback_per_minute`).

Identities are never recorded or logged.

### Feedback and review text

`POST /api/feedback` goes through `Feedback.vote`: validation, the vote limit, then whether the Answer is known, either in `RecentAnswers` (the latest 1,000 Answers this API gave in the last hour, in memory) or with an `answered`, not timed-out row in `answer_events`. An unknown id is a 404, so an id probe still spends the limit. `answer_votes` is append-only; the latest row is the current vote, and a vote equal to it inserts nothing. "Read the latest, insert if different" runs under a per-Answer advisory lock, so concurrent votes store one row. Votes use one connection of their own; any database error drops it and answers 503.

Review text is opt-in (`REVIEW_TEXT_ON_DOWNVOTE`, off by default): it is the one place the server stores what people asked. `RecentAnswers(keep_text=…)` keeps the id always and the `AnsweredQuestion` (the Question as typed and the Answer) only when it is on, and only for an hour from the Answer. A new 👎 vote row with text in memory upserts `review_texts` (Question as typed, Standalone Question, the Answer's JSON as shown) in the vote's own transaction; any 👍 deletes it. The earlier turns never reach the hook, so they cannot be stored.

### Retention

`SyncLoop(purge=…)` runs `PostgresRetention.purge(now)` after the first cycle after start, then after the first cycle at or after `FULL_SYNC_HOUR` each night, whatever the cycle did. One transaction on a connection of its own (60 s statement timeout) deletes review text older than 30 days (from the 👎), answer events older than `EVENT_RETENTION_DAYS` with their votes and review text, and votes past the retention whose event was never written, by their own time. A failed purge is logged by type and tried the next night; it never fails a cycle. A restart purges again, which is idempotent. Only the `sync` service's loop purges: while it is paused (no provider key), or when Sync runs as `sync --once` / `--full` from cron, nothing is purged.

### `/status` Protection section

`ProtectionReport` assembles answering on or off (and the message), today's spend against the budget and its source, today's refusals by reason, and this API's in-flight count against its cap (the Slack bot's slots are in another process, so the page says "API only"). The controls come from the Gatekeeper's own cache; the refusal counts are one `GROUP BY` per status request. A spend or count that cannot be read is `null`, shown as unknown. Answering off, a spent budget, or an unknown spend makes the page say "Needs attention". `/healthz` ignores all of it: a switched-off service is still alive.

### Load test

`loadtest/ask.js` (k6, image pinned by digest) asks the dev split's Questions (`loadtest/questions.json`, generated from `eval/questions.toml`) in steps of 1, 2, 4, 8, 16 and 32 at once, 60 s each, and polls `/api/status` every 2 s. `docker-compose.loadtest.yml` runs it under its own project, database volume and image tag, never reading `.env`. Its `api` runs `python -m grounded_docs.loadtest serve --recording …`, the only way to start the fakes: the answer call sleeps a duration drawn (seeded) from the recorded call times in `tests/fixtures/hedge_timings.json`, each replayed as answered at the shipped hedge delay (`HedgeRecording`, `ReplayedLatency`, `SlowLanguageModel`); embedding sleeps 300 ms and a rewrite 1.5 s (`SlowEmbedder`); the cap is 32 so steps over 8 measure answers rather than 503s, the rate limits are off and there is no budget. `loadtest_app` reuses `dev.synced_app`, the same sync-then-serve body as `uv run dev`. The Questions are typed `LoadtestQuestion`s (id, Question, previous turns), and a test keeps the committed file equal to the evaluation set's dev split. Each step's p95 is reported against 8 s in the summary's own column; the script always exits 0.

On v1.1.0 (12 threads, 15 GB), no step's p95 passed 8 s (4.48 s at 1, 5.15 s at 16, 4.83 s at 32), so the stack's own work is not the limit up to 32, and the default is N = 16: one step below the first p95 over 8 s, and with none over it, 32 minus one step. A separate check with the cap lowered to 2 refused 48 of 50 Questions `overloaded` at once, each with its event. A fake Answer has one claim where real ones have several, so verification work per Question is somewhat lower than in production.

## Seeing and evaluating

Answer events alone are only rows. Nobody sees them, and without a stated meaning of "working" there is nothing to alert on. This layer adds the objectives, the history the objectives need, and a read-only view of it all. Nothing on the answering path changes: Grafana only reads, `sync_cycles` gets one insert per cycle, and `eval_runs` is written by a CLI command.

```mermaid
flowchart LR
    TOML["observability/slo.toml"] --> BUILD["build.py: grounded_docs.slo, dashboards, alerting"]
    BUILD --> GEN["generated/*.sql"]
    BUILD --> DASH["dashboards/operations.json, quality.json"]
    BUILD --> RULES["alerting/rules.yaml, contact point, policy"]
    DASH --> GRAF["Grafana (profile observability)"]
    RULES --> GRAF
    GRAF -->|"grafana_reader"| PG["Postgres"]
    GRAF -->|"notifications"| HOOK["Slack incoming webhook"]
    SYNCL["SyncLoop, sync --once / --full"] -->|"sync_cycles"| PG
    REC["eval record, eval answers --record-to"] -->|"eval_runs"| PG
    SYN["synthetic service"] -->|"answer_events, interface synthetic"| PG
```

### SLOs, written once

Four objectives over 28 days rolling, on non-synthetic events:

| SLO | Good ÷ valid | Target |
| --- | --- | --- |
| Availability | `answered` ÷ (`answered` + `failed` + `refused` `overloaded`) | 99% |
| Latency | `answered` with `duration_ms ≤ 8000` ÷ `answered` | 95% |
| Freshness | fresh minutes ÷ minutes | 99% |
| Cost | cost of every request ÷ priced `answered` × 1,000 | ≤ $5.00 |

Refusals by policy (`rate_limited`, `answering_off`, `budget_exhausted`) are excluded from availability: a limit or the operator chose them. `overloaded` is bad: the server could not take the Question. A minute is **fresh** when an ok Sync cycle finished at most 5 minutes before it, so a paused `sync` (which writes no cycles) and failed cycles are stale. That is what a person sees. NULL costs (an unpriced model) are left out of both sides of the cost ratio and counted separately.

**Why one file, generated:** every target appears in four places (the README, its panel, two alert windows), and hand-copied numbers drift. `observability/slo.toml` holds every target, window, threshold and severity, typed by `grounded_docs.slo` (Pydantic: durations, shares and SQL conditions validated). `observability/build.py` renders the SQL into `observability/generated/*.sql`, then both dashboards and the alert rules, which embed that SQL as it is. All the output is committed, and a test fails when any of it is stale. The SQL tests load the generated files, never a copy, and run them as `grafana_reader` on real Postgres at a fixed `now`. Burn math is `numeric`, so 14.4× and 6× hold exactly at the boundary. Intervals are rendered in hours and minutes, never days, because a `timestamptz` day depends on the session's time zone.

**Why multiwindow burn rates:** a threshold on the raw error rate either pages on one bad minute or misses a slow leak. A fast pair (1 h and 5 min, both at or over 14.4× the budget rate) catches an outage within minutes, and the short window ends the alert soon after the outage ends. A slow pair (6 h and 30 min at 6×) catches a leak. Each needs at least 20 valid events in its long window, because one failure among three Questions at night is not an outage. Below that count, the query returns no row, which Grafana reads as OK.

### Sync cycles

`sync_cycles` (migration 0010) gets one row per cycle: started, finished, mode as it ran, `ok`, and either the five counts or the exception's class name. It never stores the message, which can name hosts, users or Documents. `sync_cycles.run_recorded` is the one place the rule lives: `SyncLoop` and `sync --once` / `--full` all run their cycle through it. It returns the report or re-raises, so the cycle's result never depends on the sink. `PostgresSyncCycleSink` inserts each row on a connection of its own (2 s timeouts): at one row every two minutes, a queue would add nothing. A failed insert is logged by error type, and that minute may read as stale, which errs toward alerting. A class name the table's pattern refuses (Python allows non-ASCII ones) is recorded as `UnnamedError`. A paused loop records nothing. The nightly purge deletes cycles older than `EVENT_RETENTION_DAYS`, in the events' transaction.

### Eval runs

`eval_runs` (migration 0011) keeps one row per answers run, keyed by the summary file's name: split, start, commit, models, prompt versions, repeats, each target's value (invented and accuracy with their spread), targets met, judge and indexing cost, the Outcome mix, the gate rate, and the summary as `jsonb`, with its per-question rows removed (no Question text; a test checks every dev and held-out Question against the stored rows). `source` is `harness`, `ci` or `readme`.

`evaluation/history.py` reads a summary leniently (the oldest lack later fields), rejects anything that is not an answers summary with a reason naming fields, never values, and inserts with `ON CONFLICT (run_key) DO NOTHING`, all rows in one transaction. `eval record` is therefore idempotent: a second run reports every file as present. Commits renamed by a history rewrite are recorded under their current names from a committed map. Older runs get their gate rate from the `llm_called` flags of the `-raw.json` file beside them, and nothing else is read from that file. The one held-out run whose JSON was lost is one row typed from the README's table (`--readme-run`, `summary` NULL).

**Why in the production database:** that is where Grafana reads, so quality over time sits beside production's proxies, with one data source. The evaluation's own database is a throwaway.

### The reader role

Grafana's SQL editor runs any query its data source allows, for anyone who can edit a dashboard, and the database also holds every Document's text and opt-in review text. So Grafana connects as `grafana_reader`, which `migrate` creates or updates whenever `GRAFANA_DB_PASSWORD` is set, under the migration lock, in one transaction. It revokes every grant on the schema's tables and sequences, then grants `SELECT` on exactly `answer_events`, `answer_votes`, `sync_cycles`, `sync_state`, `eval_runs`, and five columns of `operator_control_changes`. It sets a 10 s statement timeout, and keeps `DAILY_BUDGET_USD` as a role setting (`grounded_docs.default_budget_usd`), because dashboard JSON takes no environment variables and Grafana cannot read the app's `.env`.

- **Column grants, not views:** `operator_control_changes`' `old_value` / `new_value` can hold the kill switch's message, free text. Two guards need a value from them, so migration 0012 adds generated, typed columns (`budget_usd numeric`, `answering_off boolean`) that are granted instead. The writers are unchanged, and no text column is readable.
- **The password is never sent:** `ALTER ROLE` takes no bind parameters, so `migrate` computes the SCRAM-SHA-256 verifier client-side and sends that as a quoted literal. The plaintext never reaches the server or its statement log.
- **Kept honest by tests:** the role reads each granted table and is denied every table with text. A granted column that could hold text outside an explicit per-table list of identifiers fails a schema test.

### Grafana and its dashboards

Grafana OSS (`grafana/grafana` 13.2.4, pinned by digest) runs in the Compose profile `observability`, on `127.0.0.1:3000` only, with anonymous access, sign-up, analytics and update checks off. Everything comes from read-only provisioning files: the data source, both dashboards, the alert rules, the contact point and the policy. Its own state is a named volume.

- **The passwords are env-sourced Compose secrets.** A `${VAR:?}` would break the default stack, which Compose interpolates with every profile. A secret's variable is read only when its service starts: unset, Compose refuses to start `grafana`; empty, no file is mounted and `observability/grafana/entrypoint.sh` refuses, since Grafana would otherwise fall back to admin/admin. Neither shows in `docker inspect`.
- **SLO panels ignore the time picker.** Each embeds its generated query with `%(now)s` replaced by `$__timeTo()`, and overrides the panel's range with the SLO window (28 days), so the number is always the objective's. The other panels follow the picker through Grafana's macros.
- **Synthetic apart.** Every traffic and quality panel has a synthetic twin. On Quality & Evaluation, the synthetic Outcome mix, gate rate and p95 are drawn against the latest dev `eval_runs` row as dashed series read in SQL (p95 shaded from that run's fastest to slowest repeat), and a table names that run. No baseline number is written in the JSON. Real traffic gets cohorts (release · prompt · model) and no band, because its mix is not the evaluation set's. There are no dashboard variables, so a viewer's selection can never change what a panel means.
- **Proxies, never accuracy.** Production has no labels. Accuracy stays an offline evaluation number, plotted in the eval-history panels beside the 5th held-out run, named by its `run_key`.

### Alert rules and notifications

`grounded_docs.alerting` renders one rule per SLO burn pair, Sync stale, cost over target, and the four guards, evaluated every minute. A rule's query returns `firing` (0 or 1) and the window's numbers, or no row. No row is OK for every rule except Sync stale, whose query always returns one, because no cycles at all is the outage. Synthetic events are excluded except from the two budget guards. Labels carry the severity (`page` or `ticket`) and the SLO or guard. One template formats every message: the alert, its severity, its SLO or guard, the numbers, and a link to the Operations dashboard (each rule is also tied to its panel there). There are no rules on quality proxies: with no labels online, a threshold would alert on noise.

Notifications go to a Slack incoming webhook of their own (`GRAFANA_SLACK_WEBHOOK_URL`), not the bot's token, so alerts still arrive when the bot is paused or broken, and the bot's scopes stay unchanged. With the variable empty, the entrypoint provisions a copy of the folder without the contact point and the policy, plus a file that deletes any an earlier start provisioned. The rules still evaluate and show in Grafana, and nothing is sent. The 80% budget guard has a route of its own that repeats once a day.

**Alerts exist only while Grafana runs.** There is no second evaluator. This is accepted for a one-server deployment and documented.

### Synthetic traffic

`SyntheticTraffic` asks the dev split's Questions in turn (`synthetic/questions.json`, typed `LoadtestQuestion`s from the load test's generator, follow-ups with their turns), one per `SYNTHETIC_INTERVAL_SECONDS`, measured from the start of the last. It asks in-process, through a Gatekeeper of its own: `Versions.synthetic=True`, `Caller(interface="synthetic", identities=())` (no identity, so no rate limit), `max_in_flight=1`. The real providers, the operator controls, the budget and the timeout apply as for any Question, so it costs money and counts toward the budget. Migration 0013 widens `answer_events.interface` to accept `synthetic`, so its events are told apart by interface as well as by the `synthetic` flag.

**Why in-process, not over HTTP:** an HTTP client would need a way to mark itself synthetic, a token, and so an authentication surface before sign-in exists. **Why its own profile:** turning the dashboards on must never spend money. The service runs `migrate` first, and stays up, paused, logging why, without a provider key, with a missing or invalid Questions file, on an embedding model mismatch, or with a budget set and an unpriced model. Its logger is named `grounded_docs.synthetic`, because `python -m` runs the module as `__main__`.

### Evaluation in CI

`.github/workflows/eval.yml` runs `eval answers --split dev --repeats 3` with the real providers on Mondays at 06:00 UTC and on `workflow_dispatch`, on `master` only. It runs against a Postgres service container, indexing the corpus each run, with a key from a dedicated OpenAI project capped at $5 a month (`OPENAI_API_KEY_EVAL`). It is **report-only**. `eval/ci_summary.py render` writes the targets table and the change since the previous run into the job summary, with one warning annotation per missed target, and never fails. The job fails only if the run crashes. A missing secret ends it with a notice and success. The previous run is found by listing the unexpired `eval-summary` artifacts from `master`, not by listing workflow runs: a run without the key succeeds with no artifact. `eval/ci_summary.py artifact` uploads the summary without its per-question rows (90 days), checked to load as `eval record --source ci` loads it. Production is unreachable from CI, so the operator downloads it and records it by hand.

**Why report-only:** a blocking gate needs margins, and margins need the run-to-run variance that these runs are there to measure.

## Storage

One Postgres database holds the whole Index.

**Why one Postgres:** hybrid search needs vectors and keyword matching together, and Sync must swap a Document without it ever being half-indexed or present twice. With vectors, full-text indexes and Sync state in one database, each swap is one transaction and hybrid search is one query. A separate vector database would add a service and lose that single transaction. The trade-off: Postgres keyword ranking is weaker than true BM25, which the evaluation's exact-ID questions check.

| Table | Holds |
| --- | --- |
| `documents` | One row per indexed Document: source id, title, version, link, embedding model. |
| `chunks` | Text, Location, a generated `tsvector` (with whole-code lexemes) under a GIN index, and a `vector(1536)` under an HNSW index. |
| `unreadable_documents` | Documents that could not be read, the reason and the version seen. |
| `sync_failures` | Documents whose last attempt failed, with the entry to retry from. |
| `sync_state` | The Source's change cursor and the last successful incremental and full Sync times. |
| `answer_events` | One row per request (migration 0006): status, reason, Outcome, counts, stage timings, tokens, cost, versions, `synthetic`, the interface (`web`, `slack` or, since 0013, `synthetic`), the Slack workspace id. No Question or Answer text. |
| `operator_controls`, `operator_control_changes` | The controls changed with `controls`, and every change (0007); each change's typed `budget_usd` and `answering_off`, generated from its value (0012). |
| `answer_votes` | Votes, append-only (0008). |
| `review_texts` | Review text for 👎 Answers, opt-in, deleted after 30 days (0009). |
| `sync_cycles` | One row per Sync cycle: times, mode, ok, counts or the error's class name; no Document names or ids (0010). |
| `eval_runs` | One row per recorded answers evaluation: targets, versions, costs, the summary without per-question rows (0011). |
| `schema_migrations` | Applied migrations. |

Grafana reads only `answer_events`, `answer_votes`, `sync_cycles`, `sync_state`, `eval_runs` and five columns of `operator_control_changes`, through `grafana_reader` (above).

Migrations are numbered SQL files, each applied in its own transaction under an advisory lock, so `api` and `sync` can start together. Every connection sets a statement timeout. No Question or Answer text is stored unless review text is turned on, and then only for 👎 Answers: the server keeps no conversation, and logs carry text only when `LOG_QUESTIONS=true`.

## Deployment

One image runs four Docker Compose services: `api` (web page, HTTP API, status page), `sync` (the Sync loop), `slack` (the Slack bot, over Socket Mode, so nothing is exposed to the internet) and `postgres`. Services run as a non-root user. A service that cannot work (no API key, no Slack tokens, an embedding model mismatch) stays up and paused, logging why, instead of restart-looping; `api` never waits for `sync` or `slack`.

Two optional Compose profiles add one service each, and the default stack's services are identical with or without them (a test compares `docker compose config` with the file from before the profiles existed):

| Profile | Service | What it adds |
| --- | --- | --- |
| `observability` | `grafana` | Grafana on `127.0.0.1:3000`, provisioned read-only from `observability/grafana/`; refuses to start without both passwords. |
| `synthetic` | `synthetic` | The same image, running `migrate`, then `python -m grounded_docs.synthetic serve`; `synthetic/questions.json` is bind-mounted read-only, because the image holds only `src/`. |

`api` and `sync` both run `migrate` at start (it takes an advisory lock), so the reader role follows `GRAFANA_DB_PASSWORD` whichever starts first.

## Failure modes

The rule throughout: **never a fallback Answer.** A failure is reported as a failure, never dressed up as an Answer.

| Failure | Behaviour |
| --- | --- |
| OpenAI slow or failing | Each call times out after 10 s; 408, 409, 429 and 5xx, timeouts and connection errors are retried up to 3 times with exponential backoff and full jitter. Quota errors and other 4xx are not retried. After that the adapter raises `ProviderUnavailable`, and `/api/ask` answers 503. |
| Malformed structured output | Requested once more, then treated as a provider failure. |
| Slow answer call (tail latency) | If the answer call has not returned after 5 s, one identical hedge call starts; the first success wins. At most two calls in flight. |
| Partial-answer check unavailable | Fail-open: the partial Answer stands, because its claims are already verified. |
| Question takes too long overall | `/api/ask` answers 504 after 25 s; the Slack bot shows the unavailable message. The worker keeps its slot until its calls end, then writes its event, `timed_out`, with the full cost. |
| Burst over the cap | `503 overloaded` at once, no model call, an event written. |
| Over a rate limit | `429 rate_limited` with `Retry-After` (Slack: the same message), no model call, an event written. |
| Answering off, or the daily budget spent | `503 answering_off` with the operator's message, or `503 budget_exhausted` with `Retry-After` to 00:00 UTC; no model call; `/healthz` stays green. |
| Spend passes the budget mid-flight | Questions in flight finish; the overshoot is at most one Question's cost per Question in flight. |
| Controls unreadable | The last good values for 60 s; then the `.env` defaults (the kill switch back to "on"), and with a budget set, `503 database_unavailable` (fail closed on spending). |
| Unpriced model | With a budget: `api` and `slack` pause at start, naming the model. Without: cost NULL on the event, one warning. |
| Event insert fails, or the queue is full | Logged (type only) or dropped with a running count; the Answer is unaffected. |
| Votes' database unreachable | `/api/feedback` answers 503 `database_unavailable`; asking is unaffected. |
| Process restart | Rate-limit counts, `RecentAnswers` and slots reset; votes on older Answers still work through their events, but their text can no longer be kept. |
| Forged `X-Forwarded-For` | Ignored unless the peer is in `TRUSTED_PROXY_IPS`. |
| Purge fails | Logged by type; tried the next night; Sync unaffected. |
| Database unreachable or stuck | Pool and connect timeouts of 5 s, a 5 s statement timeout; `/api/ask` and `/api/status` answer 503 `database_unavailable`, `/healthz` reports it. |
| No API key configured | The stack starts; `/api/ask` answers 503 (a `failed` event); `sync` and `slack` pause (so nothing is purged); the status page says why. |
| A Document fails to sync | Its previous version stays searchable; listed on the status page with the error; retried every cycle. |
| A Document cannot be read | Listed as Unreadable with its reason; named in Answers it might have answered. |
| Sync cycle fails | Logged and recorded in `sync_cycles` with its class name; the next cycle runs on schedule; a failed full reconciliation is retried an hour later. |
| `sync_cycles` insert fails | Logged by type; the cycle's result is unaffected; that minute may read as stale (errs toward alerting). |
| Grafana down, or the profile off | Nothing on the answering path changes; alerts are not evaluated. |
| Slack webhook unset or failing | The rules evaluate and show in Grafana; Grafana logs the delivery error. |
| A Grafana password unset or empty | Unset: Compose refuses to start `grafana`; empty: its entrypoint refuses. Without `GRAFANA_DB_PASSWORD`, `migrate` creates no role. |
| A dashboard editor queries a text table | Permission denied: `grafana_reader` has no grant on it. |
| Low traffic | Burn and spike rules need their minimum counts; below them, no data, which is OK (except Sync stale). |
| `eval record` on a bad or non-answers file | Rejected with a reason naming fields, never values; the other files are still recorded; exit code 1. |
| `eval record` twice | Every file reported present; nothing duplicated. |
| CI evaluation: secret missing | A notice; the job succeeds and uploads nothing. |
| CI evaluation: targets missed | Warning annotations; the job succeeds. |
| CI evaluation: provider failure or crash | The job fails (its only failure); no artifact. |
| Synthetic traffic refused (budget, kill switch) | Refused like any Question, its event written, logged by kind and request id; it asks again at the next interval. |
| Synthetic traffic without a key or a valid Questions file | Up and paused, logging why. |

Provider adapters log operations, statuses, attempts and token counts: never prompts, Document text, model output or provider error messages, which can quote the API key.

## Where the tests sit

| Layer | Location | What it covers |
| --- | --- | --- |
| Unit | `tests/unit/` | Parsing, chunking, contracts, settings, prompts, hedging (on a scripted timeline), the Slack bot (fake Slack), the status API, the Sync loop (fake clock), evaluation scoring. Adapter contract tests run against recorded OpenAI responses and a fake Drive. Protection, with fakes and a fake clock: the Gatekeeper (`test_gatekeeper.py`: each refusal with its event and no model call, a slow fake holding every slot, the 504 that keeps its slot, two concurrent requests' usage), usage per request (`test_usage.py`), event builders and the queued sink (`test_answer_event_builder.py`, `test_answer_events.py`), the controls' cache and fallback (`test_operator_controls.py`), rate limits (`test_rate_limits.py`), trusted proxies through uvicorn's own config (`test_api_serve.py`), refusals through the API and Slack (`test_ask_failures.py`, `test_slack_bot.py`), votes (`test_feedback.py`, `test_feedback_api.py`, `test_recent_answers.py`), the Protection section (`test_status_api.py`, `test_protection_status.py`), and the load test's fakes and Questions (`test_loadtest_latency.py`, `test_loadtest_questions.py`). Seeing and evaluating: `slo.toml`'s model and the stale-file test on the generated SQL (`test_slo.py`), both dashboards and the alert provisioning rendered and committed as rendered (`test_dashboards.py`, `test_quality_dashboard.py`, `test_alerting.py`), the webhook switch in the entrypoint (`test_alerting_entrypoint.py`), both profiles' Compose shape (`test_compose_observability.py`, `test_compose_synthetic.py`), `run_recorded` (`test_sync_cycles.py`), summaries to `eval_runs` records on real summaries (`test_eval_history.py`), the CI summary and artifact (`test_ci_summary.py`), and synthetic traffic with a fake Gatekeeper and clock (`test_synthetic.py`). |
| Integration | `tests/integration/` | The Assistant and Sync end to end against a real Postgres with pgvector (a testcontainers container locally, a service container in CI), with fake providers: grounding and downgrades, hybrid search, follow-ups, Sync changes and failures, migrations, query timeouts, the HTTP API. Protection on real Postgres: events read back field by field with the no-text schema test and `CHECK`s (`test_answer_events.py`), one event per Answer from both interfaces under the logged id (`test_gatekeeper_events.py`), the `controls` command and both Gatekeepers seeing a change (`test_operator_controls.py`), votes (`test_feedback.py`), review text (`test_review_texts.py`), the purge (`test_retention.py`), refusal counts (`test_status.py`), and the load-test app (`test_loadtest.py`). Seeing and evaluating on real Postgres: every SLI, burn-rate and guard query loaded from the generated files at a fixed time (`test_slo_queries.py`), every dashboard and alert query read from the committed JSON and YAML and run as `grafana_reader`, with values asserted (`test_operations_dashboard.py`, `test_quality_dashboard.py`, `test_alerting_queries.py`), the role's grants and denials (`test_grafana_reader.py`), Sync cycles and their purge (`test_sync_cycles.py`, `test_retention.py`), `eval record` (`test_eval_history.py`), synthetic events through its Gatekeeper (`test_synthetic_traffic.py`, `test_synthetic_interface.py`), and, with Docker, the default stack unchanged by the profiles and Grafana refusing to start without its password (`test_compose_observability.py`). |
| Live | `tests/live/` | A few calls to the real OpenAI API; skipped unless a key is set, never run in CI. |
| Evaluation | `uv run eval` | Not a test: 99 questions over the demo corpus with the real providers, split into dev (for tuning) and held-out (for reporting). Results are in the README; the dev split also runs weekly in CI, report-only. |
| Observability smoke | `observability/smoke.sh` | Against a running stack with the profile: Grafana healthy, both dashboards listed, every dashboard query run through the provisioned data source, every alert rule provisioned and evaluating, a query on `review_texts` denied. |
| Load test | `loadtest/ask.js` | Not in CI: a script and a documented run over fake providers (above). |

CI runs three jobs on every push. The first runs lint, format check, strict mypy and the full test suite against a Postgres service, with branch coverage as a table in the job summary (report-only: no threshold, no badge). The second, a Compose smoke test, builds the image, starts the stack from a clean checkout with no provider keys, and checks `/healthz`, `/api/status` and that every service stays running. The third, an observability smoke test, checks that Grafana refuses to start without its admin password, then starts the profile with throwaway passwords and runs `observability/smoke.sh`. A second workflow, `eval.yml`, runs the weekly evaluation (above).
