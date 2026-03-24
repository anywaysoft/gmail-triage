# Gmail Triage — Technical Specification (v0.1)

**Last updated:** 2026-03-24 (America/New_York)

This document captures the current product/engineering design for a CLI-first Gmail triage tool.

It is intentionally detailed so a future session can pick up quickly and implement the same system without re-deriving decisions.

---

## 1) Problem statement

The user receives dozens/hundreds of emails per day, most of which are spam/subscriptions/marketing updates they never read.

Current manual workflow (weekly/biweekly):
- Scroll inbox newest→oldest page-by-page.
- Decide mostly from **sender** and sometimes **subject**.
- Bulk-delete almost everything.
- Keep:
  - Bills (Phase 2: extract + create tasks)
  - Emails from real people the user has communicated with (not necessarily in contacts)
  - Rare “interesting” opportunities (e.g., recruiter email)
- Stop once reaching the area previously processed.

Pain points:
- Gmail filters are brittle (“too precise”) and become stale.
- User is worried about overly aggressive automation causing missed important mail.
- User strongly prefers **not** to pay for a model reading the body of every email.

Goal: automate/simplify this cleanup with a system that:
- Mimics the user’s “sender/subject first” behavior.
- Learns from user actions over time.
- Operates primarily on metadata.
- Is safe, reversible, and understandable via a digest with “whys”.

---

## 2) Goals / non-goals

### Goals
- **CLI-first**: single-shot command suitable for cron (no daemon required for MVP).
- **Local or cloud**:
  - Local mode: SQLite (easy backups).
  - Cloud mode: Postgres.
  - Same schema/SQL subset so switching is configuration-only (`DATABASE_URL`).
- **Metadata-first**:
  - Default fetch is Gmail “metadata” (headers + labels), not full bodies.
  - Body/snippet is optional and should be used sparingly (future/optional).
- **Stream-driven triage**:
  - Automatically group emails into “streams”.
  - Apply stream labels in Gmail (user wants a label per stream).
  - Streams can be automatically created/split over time.
- **Triage actions**:
  - High-confidence junk: label + (optionally) move to Trash.
  - Uncertain: label and **leave in Inbox, unread**.
- **Digest always produced**:
  - Each run outputs a Markdown digest to stdout and/or a file.
  - Digest includes “why” explanations and summary of actions.
- **Learning**:
  - Learn from user behavior + label diffs.
  - Opening an email alone is neutral.
  - Untrash is a strong signal that auto-trash was wrong.
  - Changing/removing stream labels should provide feedback and drive better clustering.
- **Reset/troubleshooting**:
  - A reset mode that makes a time window look like triage never ran (Gmail + DB).

### Non-goals (MVP)
- Full GUI inside Gmail (extension/add-on) — later.
- Gmail push notifications — later (polling/cron is fine).
- Perfect category ontology (“marketing crap”, “clickbait”, etc.).
  - Streams emerge automatically; user doesn’t maintain a taxonomy.
- Phase 2: bill extraction + task creation.

---

## 3) High-level architecture

### Components
1. **Core engine (pure-ish)**
   - Input: message metadata + stream state
   - Output: stream assignment + decision + confidence + why[]

2. **Gmail adapter (side effects)**
   - Fetch message IDs (bounded by `--since`)
   - Fetch per-message metadata headers/labels
   - Apply labels / move to trash / untrash (when requested)
   - Create/rename/delete Gmail labels as needed

3. **State store (DB)**
   - Streams, policies, run history, decisions, feedback events, label snapshots
   - Works with SQLite or Postgres

4. **CLI runner**
   - Orchestrates: select → classify → label → (optional) move → digest → learn

### Execution model
- One-shot CLI invoked by cron or manually.
- No background daemon required.
- A future UI can wrap the same engine via an HTTP service, but the engine remains usable standalone.

---

## 4) Terminology

- **Message**: Gmail message (by message ID).
- **Stream**: a cluster of emails that should usually receive the same disposition.
  - Roughly “Netflix promos” rather than “marketing in general”.
- **Policy**: per-stream default action (AutoTrash vs Review vs Keep) + thresholds.
- **Decision**: per-message result for a run.
- **Why**: short human-readable explanation of a decision.
- **Feedback**: signals derived from user actions (untrash, label changes, stars, replies, etc.).

---

## 5) Gmail label taxonomy

All tool-managed labels are under a single prefix.

### Required labels
- `Triage/Processed`
  - Marks messages the tool has already evaluated within the selected window.
  - Used for idempotence.

- `Triage/AutoTrash`
  - “This is safe to trash.”
  - In non-destructive runs, items can remain in Inbox with this label.

- `Triage/Review`
  - “Needs human review.”
  - Must remain **in Inbox** and **unread**.

- `Triage/Stream/<slug>`
  - One label per stream.
  - `<slug>` is friendly and human-readable.
  - If collisions occur, disambiguate with `-2`, `-3`, etc.

- `Triage/Why/<code>`
  - Exactly one “primary why” per message (optional but recommended).
  - Small fixed set of codes to avoid label explosion.

### Optional/advanced labels
- `Triage/NewStreamCandidate`
  - Applied when the engine creates a brand-new stream.

- `Triage/Override/*`
  - Explicit user overrides (strong feedback and always respected).
  - Examples:
    - `Triage/Override/Keep`
    - `Triage/Override/Review`
    - `Triage/Override/AutoTrash`

Notes:
- Label diffs (remove/replace stream labels) are also treated as feedback, but explicit override labels are the cleanest.

---

## 6) “Stop” mechanism (avoid scanning 10-year history)

The tool must **not** wander into deep history by default.

### Selection window
- CLI accepts `--since` (duration or timestamp).
- Default behavior:
  - If a prior successful run exists, default to “since last run” (minus small overlap).
  - Otherwise default to a bounded window (e.g., 14 days).

### Gmail query strategy (MVP)
- Use Gmail search query that is time-bounded, e.g.:
  - `in:inbox is:unread newer_than:14d -label:Triage/Processed`
- Additionally enforce a hard cap:
  - `--max-messages` (safety) and/or `--max-pages`.

Rationale:
- Time window is safer than “stop at first processed” (paging/sorting is not guaranteed stable).

---

## 7) Stream model

### Stream granularity
Default stream should be close to “stable source”:
- Prefer `List-ID` header (best for mailing lists even when From changes).
- Else prefer from-domain (eTLD+1).
- Subject-based splitting is optional and should only happen when necessary.

### Stream creation
When encountering a message that does not map confidently to an existing stream:
- Create a new stream in the DB.
- Create `Triage/Stream/<slug>` label.
- Apply `Triage/NewStreamCandidate` label to the message (optional).

### Stream splitting (automated)
User wants maximal automation. Splitting should be automatic when evidence is strong:
- If many messages assigned to stream `Foo` are later rejected/moved and share a consistent differentiator (e.g., subject-template), create `Bar` automatically and start assigning similar items to `Bar`.
- Splits should be recorded in the digest.

### Label explosion mitigation
Because user prefers a label per stream:
- Provide pruning tooling:
  - `triage prune-labels --older-than 180d` deletes unused/empty stream labels.
- Provide a configurable cap:
  - `--max-stream-labels` (fails safe by collapsing excess into `_new`).

---

## 8) Decision model (MVP)

### Decisions
For each scanned message, the engine emits one of:
- `AUTO_TRASH`
- `REVIEW`
- `SKIP` (rare; do nothing except record in DB/digest)

### Key behavior constraints
- `REVIEW` items stay in **Inbox** and **unread**.
- `AUTO_TRASH` items:
  - Always get the `Triage/AutoTrash` label.
  - Are moved to Trash only in destructive mode.

### Signals/features (metadata-first)
Potential features:
- Headers: `From`, `Reply-To`, `To`, `Cc`, `Subject`, `Date`
- Bulk/list indicators: `List-ID`, `List-Unsubscribe`, `Precedence`, `Auto-Submitted`
- Gmail system labels: `IMPORTANT` (weak), categories (if available)
- Behavior graph (from history/state):
  - “known person” if previously replied/sent to this address
  - stream-level open/delete/untrash rates

### “Important” handling
User observed:
- Many emails Google marks as IMPORTANT still get deleted.
- Almost everything they keep is marked IMPORTANT.

Interpretation:
- Treat “not important” as a mild negative signal (toward trash).
- Treat IMPORTANT as weak positive at most.

---

## 9) Apply modes (two-step runs)

The user wants the ability to run non-destructively.

### `--apply=none`
- No Gmail changes.
- Produce digest with planned actions.
- Optionally still writes a run record (implementation choice; for MVP, yes).

### `--apply=labels`
- Apply labels only:
  - `Triage/Processed`
  - decision label (`Triage/AutoTrash` or `Triage/Review`)
  - stream label
  - why label
- Do not move messages to Trash.

### `--apply=trash`
- First apply labels as in `labels`.
- Then move all messages labeled `Triage/AutoTrash` to Trash.

Rationale:
- Enables a workflow: label everything → optionally review → apply trash.

---

## 10) Feedback & learning

### Feedback sources (priority)
Strong negative:
- **Untrash** of an `AUTO_TRASH` message

Strong positive:
- Replies
- Stars
- Explicit override labels

Weak/neutral:
- Opened/read alone (neutral)

Weak negative:
- Message was `REVIEW` and user deletes without reading

### Label-diff feedback (explicit goal)
User wants:
- If something is labeled `Triage/Stream/Foo` and user removes or replaces that label, the system should treat it as feedback.

Mechanism:
- Store per-message label snapshot after each run.
- Next run, re-fetch labels for recently touched messages and diff.

Semantics:
- Remove `Triage/Stream/Foo` → `stream_reject(Foo)`.
- Replace Foo→Bar → `stream_move(Foo→Bar)`.
- Remove `Triage/AutoTrash` → `decision_reject(AUTO_TRASH)` (unless it happened via `triage reset`).

### Automated re-clustering
When rejects cluster around a differentiator:
- Auto-create a new stream and start reassigning.
- Record the split in the digest.

---

## 11) Reset / troubleshooting

The user requirement:
> reset should mean “make everything (gmail and db) look like triage has never run within this time window”.

### `triage reset --since <window>`
- Gmail:
  - Remove all `Triage/*` labels from messages within the window.
- DB:
  - Delete run/decision/action/feedback records for that window.

### Optional: undo side effects
- `triage reset --since <window> --undo-moves`
  - Untrash messages the tool moved within the window.
  - (Potentially also re-add `INBOX` if tool archived — future.)

Safety:
- Undoing moves is riskier; default is label/db-only reset.

---

## 12) Digest format

Digest should be Markdown and always emitted.

Suggested sections:
- Run summary (time window, apply mode, limits)
- Counts (scanned, autotrash, review, skipped)
- Top streams by volume
- New streams created (if any)
- Auto-splits performed (if any)
- Actions taken (with samples)
- Review queue summary
- Feedback detected since last run (untrash, label moves)
- Errors/warnings

Outputs:
- `--digest stdout` (default)
- `--digest <path>` writes file

---

## 13) Data model (SQLite/Postgres compatible)

### Tables (sketch)
- `streams`
  - `id` (UUID)
  - `slug` (unique)
  - `gmail_label_name` (unique)
  - `source_type` (`LIST_ID` | `DOMAIN` | `ADDRESS`)
  - `source_key` (string)
  - `status` (`ACTIVE` | `MERGED` | `PRUNED`)
  - `created_at`, `updated_at`

- `runs`
  - `id` (UUID)
  - `started_at`, `ended_at`
  - `since` (timestamp/duration representation)
  - `apply_mode` (`NONE` | `LABELS` | `TRASH`)
  - `version` (engine version)

- `messages`
  - `id` (Gmail message ID string, PK)
  - `thread_id`
  - `internal_date` (timestamp)
  - minimal cached metadata used for stream assignment

- `decisions`
  - `run_id`, `message_id`
  - `stream_id`
  - `decision` (`AUTO_TRASH` | `REVIEW` | `SKIP`)
  - `confidence` (0..1)
  - `primary_why_code`

- `label_snapshots`
  - `message_id`
  - `captured_at`
  - `labels_json` (stored as TEXT; avoid JSONB-only features)

- `feedback_events`
  - `id` (UUID)
  - `message_id` (nullable)
  - `stream_id` (nullable)
  - `type` (`UNTRASH`, `STREAM_REJECT`, `STREAM_MOVE`, `DECISION_REJECT`, `STAR`, `REPLY`, ...)
  - `created_at`
  - `details` (TEXT)

Notes:
- Derived stream stats can be computed from `decisions` + `feedback_events` (avoid counters that are hard to roll back during reset).

---

## 14) Implementation notes (Node/TypeScript)

Language: **TypeScript/Node** for backend/CLI.

Suggested libraries (subject to final selection):
- Gmail API client: `googleapis`
- CLI: `commander` or `yargs`
- DB: `kysely` or `drizzle` (must support SQLite + Postgres)
- Migrations: tool-specific (keep dialect-neutral)

OAuth/token storage:
- Local: file in a `data/` directory with strict permissions or OS keychain integration.
- Cloud: secrets manager / environment secret.

---

## 15) Future phases

### Phase 2 (bills)
- Detect bills/statement emails and extract amount/due date.
- Create tasks (provider TBD).

### Phase 3 (UI)
- Chrome extension / Gmail sidebar that shows:
  - streams, policies, suggested actions
  - one-click override and merges/splits

### Phase 4 (optional AI)
- Only for uncertain items or for better clustering.
- Default remains metadata-first.

---

## 16) Open questions

- Exact default `--since` window (14d?) and overlap.
- How aggressively to auto-split streams (thresholds).
- How to compute subject templates without model calls.
- Whether to apply any labels to “SKIP/keep” messages.

