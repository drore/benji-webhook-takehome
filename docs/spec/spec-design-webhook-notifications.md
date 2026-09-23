---
title: Webhook notification system and operations dashboard
version: 0.2-review
date_created: 2026-09-23
last_updated: 2026-09-23
owner: Dror Elovits
tags: [design, architecture, webhook, dashboard, take-home]
---

# Introduction

This is the living specification for the [Benji senior software engineer take-home](https://even-foxglove-53c.notion.site/Sr-Software-Engineer-380b9f151bae81218487d84c42741d17). It records the assignment's requested outcomes, our current design direction, and decisions still to make. It is intended to guide implementation and the final AI-assisted plan; it is not a claim that the system has been built.

**Status terms:** **Required** comes from the assignment. **Chosen** is agreed in discussion. **Specified for review** is a concrete working behavior selected here so the stories can be checked; it is not Dror's approval or implementation evidence. **Proposed** is a design option. **Open** still needs a decision.

## 1. Purpose and scope

Build a local, one-day fullstack demonstration in which a customer registers multiple webhook endpoints, selects event types, triggers events, and inspects reliable, authenticated delivery. The audience is Benji's reviewers and an operator investigating delivery issues.

The assignment prefers Python for the backend and Vue 3 with TypeScript for the frontend. Dror is comfortable with that stack (**Chosen**). The solution must run locally without external infrastructure (**Required**). The [public repository](https://github.com/drore/benji-webhook-takehome) currently contains planning documents; code, a detailed plan, tests, run instructions, and an AI-usage note remain required for submission. Work follows SDD-TDD in small iterations (**Chosen process**). The [user-story readiness review](spec-design-user-stories.md) must close before a detailed development plan is written; the [iteration sequence](spec-process-sdd-tdd-iterations.md) is provisional.

## 2. Definitions

| Term | Meaning |
| --- | --- |
| Event | An occurrence identified by an event ID, type, payload, and creation time. |
| Endpoint | A customer-configured URL, event-type subscription set, enabled state, and signing secret. |
| Logical delivery | The durable association of one event with one eligible endpoint. |
| Attempt | One actual HTTP request for a logical delivery. Retries and replay add attempts. |
| Replay | An operator action that schedules another attempt for a failed logical delivery. |
| Receiver | A service at the endpoint URL that accepts and verifies webhook requests. |
| HMAC | Hash-based message authentication code; the proposed method for signing requests. |

## 3. Requirements, constraints, and working guidelines

### Assignment requirements

- **REQ-001:** A customer can register multiple endpoints, each subscribed to one or more event types.
- **REQ-002:** Failed deliveries are retried through a documented mechanism.
- **REQ-003:** A receiver can verify that a webhook request came from the sender.
- **REQ-004:** Failures and attempt history are persisted, inspectable, and replayable.
- **REQ-005:** A single event has at most one logical delivery to a given endpoint.
- **REQ-006:** The dashboard registers endpoints, chooses event types, enables or disables endpoints, and displays a signing secret once on creation.
- **REQ-007:** The dashboard triggers test events with a chosen payload and clearly reports deduplicated resubmission.
- **REQ-008:** Delivery status changes appear without manual refresh; a delivery exposes its payload and attempts; failed deliveries can be replayed.
- **REQ-009:** Frontend requests have loading, empty, disabled-while-pending, and consistent error states. API interaction, application state, and presentation are separated.
- **CON-001:** The working solution runs locally with no external infrastructure and is scoped to no more than one day of focused work.
- **CON-002:** The deliverables include an implementation plan with tradeoffs and cuts, meaningful tests, a run/test README, and a brief AI-usage trail.

### Agreed interpretation and proposed design

- **DEC-001 (Chosen):** The main dashboard experience is event-centered: a selected event visibly fans out to its eligible receivers. Selecting a branch opens its delivery and attempt details.
- **DEC-002 (Chosen):** Use a familiar operations-dashboard layout and draw visual cues from Benji's public brand. Prefer clear status labels and controls over elaborate animation.
- **DES-001 (Proposed):** Use a Python API, SQLite for durable local state, a small scheduled delivery worker, and a Vue 3/TypeScript dashboard.
- **DES-002 (Specified for review):** Enforce a database uniqueness constraint on `(event_id, endpoint_id)`. Retries and manual replay append attempts to the existing delivery. HTTP transmission is at least once when failures or uncertain timeouts occur; receiver-side idempotency uses the stable delivery ID. Do not claim exactly-once remote processing.
- **DES-003 (Specified for review):** Sign the timestamp, endpoint ID, stable delivery ID, and exact raw request body with an endpoint-specific secret using the format in SPEC-010. The demo receiver verifies the signature and timestamp.
- **DES-004 (Specified for review):** Show an event's type, ID, time, and a small payload summary on its card; open formatted full JSON on selection. A receiver branch shows a textual state, icon, and status color. A pulse represents an actual HTTP attempt; the logical-delivery branch remains stable across retries.
- **DES-005 (Specified for review):** Poll the delivery API while the dashboard is visible to update statuses without manual refresh. Keep event and delivery state authoritative on the server.
- **DES-006 (Proposed):** Apply the [security and scalability boundaries](spec-architecture-security-and-scale.md) as each slice introduces a new surface. The local demo does not imply production readiness.

## 4. Behavior and data contracts (**Specified for review**)

These are product and externally observable contracts. The later development plan may choose exact paths, internal module boundaries, and storage schema without changing these outcomes.

| ID | Contract |
| --- | --- |
| **SPEC-001 Local scope** | The demo represents one fixed customer. The API and dashboard bind to loopback; the API accepts the configured dashboard origin only. The browser does not choose a customer ID. No operator authentication is claimed. Production requires customer authentication and authorization on every resource and action. |
| **SPEC-002 Endpoints** | An operator creates up to 20 endpoints with a URL and nonempty set of event types; each starts enabled. Endpoint ID, URL, types, and secret are immutable in the demo. The operator can enable or disable an endpoint; changing its URL/types means creating a new endpoint and disabling the old one. The secret appears only in the creation response and is absent from later reads and normal logs. |
| **SPEC-003 Destination policy** | Configurable URLs must use HTTP and match the one receiver origin configured on the server for local mode (default `127.0.0.1` and the documented receiver port). Only documented `/webhooks/` receiver routes are accepted; receiver administration routes are excluded. Reject credentials, fragments, unsupported schemes, and other hosts/ports. Revalidate at dispatch, pin the connection to the allowed origin, and never follow redirects. Public HTTPS destinations require a separate production policy and network controls. |
| **SPEC-004 Event identity** | The local customer supplies a case-sensitive event ID of 1–128 ASCII letters, digits, `_`, or `-`, an event type of 1–64 such characters, and any valid JSON payload up to 32 KiB encoded as UTF-8. The server compares a canonical JSON representation, so object key order and whitespace do not change identity. Subject to validation and intake limits, the first `(customer, event ID)` submission accepts and persists the event. The same ID, type, and JSON value returns `deduplicated=true` without new deliveries, even if the new-event rate limit is full. The same ID with different type or JSON value returns `409 event_conflict` without modifying the original. |
| **SPEC-005 Fan-out** | At acceptance, match endpoints that are enabled and subscribe to the event type. Persist the event and all corresponding logical deliveries in one transaction. Enforce unique `(event_id, endpoint_id)` in storage, including under concurrent submissions. An accepted event with no matches persists with zero deliveries and displays `No receivers`; a later subscription change does not add deliveries retroactively. Since URL/types/secret are immutable, existing deliveries retain their original destination and signing identity. |
| **SPEC-006 Disable** | Disabling blocks new fan-out and pauses any existing `pending` or `retrying` delivery for that endpoint without erasing its due time or attempt history. An already running HTTP attempt may finish; a failure that would retry becomes paused. Re-enabling restores paused work; overdue work becomes immediately eligible, with no retry-budget reset. A completed `succeeded` or `failed` delivery retains its terminal status. |
| **SPEC-007 Automatic attempts** | The initial attempt is due immediately. A 2xx response succeeds. Transport errors, the 2-second total timeout, HTTP 408/429, and 5xx retry after 2 seconds and then 5 seconds measured from the prior attempt's completion, for at most three attempts per cycle. Other non-2xx responses fail terminally without automatic retry. Redirects are not followed and 3xx is terminal. Every attempt records start/end, HTTP status or classified error, and at most 1 KiB of response text. These short deterministic delays make the local walkthrough reproducible; production would revisit policy and add jitter. |
| **SPEC-008 Dispatch and recovery** | A worker claims due work with a 10-second lease, at most four concurrent HTTP attempts globally and one per endpoint. A claim interrupted by a crash becomes eligible after lease expiry. Its attempt is marked `interrupted` and counts against the current cycle's three-attempt budget because remote processing is uncertain. It retries with the same delivery ID if budget remains; otherwise the delivery becomes `failed` and can be replayed. Event acceptance and delivery creation do not wait for outbound HTTP. |
| **SPEC-009 Replay** | Only a terminal `failed` delivery whose endpoint is enabled may be replayed. Replay starts a new cycle of up to three attempts on the same logical delivery, keeps monotonically numbered old attempts and the same delivery ID, and schedules the next attempt immediately. Replay of a succeeded, paused, pending, in-progress, retrying, or disabled-endpoint delivery returns `409 replay_unavailable`. Concurrent replay requests yield at most one new cycle; the UI disables unavailable actions. |
| **SPEC-010 Signed request** | Each endpoint has a cryptographically random 32-byte secret shown once as `whsec_` plus unpadded base64url. For each attempt, send the stored event JSON bytes as the body and headers `X-Webhook-Endpoint-Id`, `X-Webhook-Delivery-Id`, `X-Webhook-Timestamp` (Unix seconds), and `X-Webhook-Signature` (`v1=` plus lowercase hex HMAC-SHA256). The exact signed bytes are UTF-8 `v1\n{timestamp}\n{endpoint_id}\n{delivery_id}\n` followed by the raw body bytes. Generated endpoint/delivery IDs have an unambiguous ASCII format. The receiver uses constant-time signature comparison, rejects malformed headers and timestamps outside ±300 seconds, and deduplicates side effects by delivery ID; replay uses fresh timestamp/signature but the same delivery ID. |
| **SPEC-011 Limits and errors** | Local intake permits at most 60 new events per rolling minute for the fixed customer; exceeding it returns `429 rate_limited`. Invalid URL/ID/type/JSON/size returns `400 validation_error`. A conflicting event or unavailable replay returns 409 with the codes above. Unexpected server errors use `500 internal_error` without secrets or payloads. API errors share `{code, message, details?}`; the frontend distinguishes these from transport failures and never invents delivery success. |

**Entity minimums:** Event stores ID, fixed customer, type, canonical JSON bytes, creation time. Endpoint stores ID, immutable URL/type set/secret and enabled flag. Delivery stores ID, event/endpoint IDs, status, due time, attempt cycle, and claim lease. Attempt stores ID, delivery ID, monotonic number, timing, outcome, and bounded response excerpt. Persist the signing secret locally but never return it after creation; production storage and rotation require a separate design.

**Required API capabilities:** create/list endpoints; enable/disable endpoint; submit/list/read events with deduplication result; read delivery and attempts; replay failed delivery. Creation and deduplication can use `201` and `200` respectively. The dashboard reads authoritative state; it does not infer persistence from a successful click.

**Delivery states:** `pending`, `in_progress`, `retrying`, `paused`, `succeeded`, `failed`. `pending` includes an accepted delivery and a replay scheduled immediately. `retrying` has a future due time. `paused` is gated by endpoint disable. `succeeded` and `failed` are terminal until eligible replay moves `failed` to `pending`.

## 5. Dashboard and interaction design

### Layout

1. **Header:** concise operational signals, each opening a relevant filtered view.
2. **Recent events:** select an event; show useful empty and loading states.
3. **Selected event workflow:** event card on the left, eligible receiver branches on the right, and a concise outcome summary.
4. **Investigation panel:** click a branch to see URL, state, payload, attempt timeline, HTTP outcomes, next retry, and replay when applicable.
5. **Management actions:** create or enable/disable endpoints and fire a test event without leaving the dashboard context.

### First-version header signals (**Specified for review**)

| Signal | Definition | Drill-down |
| --- | --- | --- |
| Needs attention | Current count of terminal `failed` logical deliveries, including immediate nonretryable failures. | Failed deliveries and replay action. |
| Retrying | Current count of `retrying` deliveries and the earliest due time. Paused work is excluded. | Scheduled deliveries. |
| Delivery rate, last 24 hours | Current `succeeded` / (`succeeded` + `failed`) logical deliveries whose events were created in the trailing 24 hours; show counts. If denominator is zero, show `—`, not 0%. | Underlying terminal deliveries. |
| Latest event | Most recently created event type, age, and derived outcome. | Select that event's workflow. |

`Receiver at risk` is deferred until there is a useful, testable rule from a larger history. `Sentiment` has no source in the assignment's webhook data and is excluded from the first version.

**Workflow outcome:** Show counts by delivery state alongside every event. The headline is `No receivers` for zero deliveries; `Needs attention` when any delivery is terminally failed; `Delivering` when any delivery is pending, in progress, or retrying; `Paused` when all unresolved deliveries are paused; otherwise `Delivered` when all deliveries succeeded. Counts remain visible when a headline combines different branch states. A disabled endpoint is not rendered as a branch for an event accepted after disable; a paused existing branch remains visible.

**Live and failure behavior:** Poll authoritative status every two seconds while the dashboard tab is visible. After a polling failure, retain the last known state, show a `Status may be stale` banner and last successful update time, and retry automatically on the next interval. Never animate an attempt that was not observed from server state. For an accepted no-match event, show the event card and a `No receivers matched` explanation. With many branches, use a vertically scrollable receiver list; selecting a branch preserves context in the investigation panel. Payload cards show type/ID/time and a short escaped JSON preview; full formatted JSON is available on selection, never rendered as HTML.

**Action states:** Creation, submission, and replay controls show pending state and disable repeat clicks until a response. Validation and conflict errors are shown next to the affected action with the server's safe message; transport/server errors show a retry action. A background delivery failure is shown on its branch and in `Needs attention`, not as a failed event-submission request.

### Visual language (**Chosen direction, implementation tokens proposed**)

The [public Benji site](https://withbenji.com/) observed on 2026-09-23 uses Lexend for prominent headings, Inter for body copy, dark navy text (`#071437` observed on the hero heading), vivid purple (`#9900FF` observed on a primary button), white surfaces, rounded controls, light borders, and generous spacing. The [public Pilot dashboard documentation](https://docs.withbenji.com/pilot/dashboard) describes overview, activity, breakdown, and highlight patterns. These are references; the private product's design system has not been inspected.

- Use purple for selection and primary actions, with navy and neutral surfaces as the base.
- Use status colors only with text and icons: gray pending, blue in progress, amber retrying, green succeeded, red failed.
- Use a fixed fan-out layout that can scroll or collapse when many receivers exist. Avoid a freeform graph editor.
- Represent the payload as a safe text summary and formatted JSON; do not render payload content as HTML.

## 6. Demonstration and acceptance criteria

**Independent demo receiver (Specified for review):** Start a separate loopback HTTP service with success, fail-once, always-fail, and slow routes plus a small received-request view. Create each endpoint through the dashboard, then copy its displayed endpoint ID and one-time secret into the receiver's local configuration form; keep this receiver mapping in memory only. The form never sends the secret to the sender API. The receiver shows verified requests, deduplicated delivery IDs, and signature failures. A clean-start walkthrough starts receiver, API/worker, and frontend, configures the secrets, submits events, and observes actual HTTP traffic and dashboard state. The later plan/README will supply exact commands.

- **AC-001:** Given two endpoints subscribed to the same type, when a matching event is submitted, then each has one logical delivery and visible status.
- **AC-002:** Given an event ID already submitted, when the same type/JSON value is submitted again (regardless of key order), then the dashboard reports deduplication and no new deliveries are created; different type/value receives 409 without changing the original.
- **AC-003:** Given a receiver that fails once, when retry time arrives, then the same delivery records a second attempt and can succeed.
- **AC-004:** Given exhausted retries, when an operator replays the failure, then the same delivery gains another attempt and its earlier failures remain visible.
- **AC-005:** Given a signed request, when the receiver verifies the timestamp, IDs, and raw body, then an unchanged request passes; modified body/ID/signature, malformed headers, or a stale timestamp fail.
- **AC-006:** Given the dashboard remains open, when backend status changes, then visible workflow status updates without manual refresh.
- **AC-007:** Given endpoint creation succeeds, then the secret appears once; later endpoint reads do not reveal it.
- **AC-008:** Given an endpoint is disabled before acceptance, it receives no new delivery; existing queued work pauses, an in-flight attempt may finish, and re-enable resumes overdue work without resetting attempt history.
- **AC-009:** Given an event matches no enabled endpoint, it persists with zero deliveries and the dashboard says `No receivers`.
- **AC-010:** Given concurrent duplicate event submissions and replay requests, storage preserves one event–endpoint delivery and at most one active replay cycle.
- **AC-011:** Given a worker interruption during an attempt, after lease expiry the recorded attempt becomes `interrupted` and the same delivery resumes within its remaining budget or becomes replayable failure.
- **AC-012:** Given an unavailable replay, the API responds with `409 replay_unavailable` and the UI does not offer an enabled replay control.
- **AC-013:** Given polling fails, the dashboard retains the last known state, identifies it as stale, and resumes updates automatically when the API recovers.
- **AC-014:** Given a clean local setup, a reviewer can configure receiver secrets via the one-time display and demonstrate success, retry, terminal failure, replay, deduplication, and receiver-side signature rejection without external infrastructure.

## 7. Test automation strategy

- **Unit:** event-to-endpoint matching, duplicate submission, delivery state transitions, signing and verification, retry schedule.
- **Integration:** SQLite uniqueness and persistence, worker-to-local-receiver HTTP outcomes, replay retaining attempt history, disabled-endpoint behavior, and restart recovery.
- **Frontend:** focused behavior checks for loading, empty, request-in-flight, deduplication notice, status update, and replay error handling. Use the project's chosen test tooling once initialized.
- **Manual walkthrough:** run the local receiver and dashboard; show success, retry, terminal failure, replay, signature verification, and dashboard drill-down. Record exact commands in the README.

## 8. Rationale, tradeoffs, and cuts

The fixed workflow makes delivery routing and failure diagnosis legible while staying within the one-day constraint. A durable local store keeps attempt history inspectable after a process restart. Polling is sufficient for the requested no-refresh behavior at local-demo scale. A separate receiver makes the HTTP and verification boundary visible.

**Planned cuts (Proposed):** distributed workers, externally hosted queues, multi-tenant authentication, secret rotation, broad analytics, and a general graph layout. The final plan must state which were actually cut and what production design would require. Any security limits of the local exercise must be documented accurately.

## 9. Dependencies and external integrations

- **EXT-001:** No hosted external service is required to run or demonstrate the system.
- **INF-001:** Local persistent storage is required for events, endpoints, deliveries, and attempts.
- **INF-002:** A local HTTP receiver is proposed for the walkthrough and integration tests.
- **PLT-001:** Python backend and Vue 3/TypeScript frontend are the chosen target stack, matching the assignment's preference.

## 10. Examples and edge cases

Example: `campaign_updated` with event ID `evt_123` matches endpoints A and B. Two delivery rows are created. A succeeds on its first attempt. B times out, retries, and succeeds; B still has one delivery row and two attempt rows. Resubmitting `evt_123` creates no additional delivery rows. A timeout leaves remote processing uncertain, so a receiver should deduplicate by delivery ID.

Specified edge outcomes to verify: changed payload under the same ID returns conflict; disabling pauses scheduled retries; malformed or oversized payload is rejected; unreachable/slow receivers follow the bounded retry policy; restart recovers expired claims; replay of disabled work is rejected; no-match events persist visibly; polling interruption shows stale state; a zero-denominator delivery rate shows `—`.

## 11. Validation criteria and review decisions

**Current status: behavior draft complete for owner review; spec not yet finalized.** The story gaps now have explicit working outcomes in SPEC-001..011, the dashboard contract, and AC-001..014. Before calling the spec final, Dror should review the material product/security choices below and the [story coverage check](spec-design-user-stories.md). A detailed development plan and implementation follow only after that review. The implementation is ready for submission only after the final local code passes the acceptance criteria and the required plan, README, tests, and AI-usage note are present.

| Review topic | Working choice | Why it matters |
| --- | --- | --- |
| Endpoint changes and disable | Immutable URL/types/secret in the demo; disabling pauses queued work. | Keeps existing deliveries stable but prevents editing an endpoint in place. |
| Event idempotency | Same ID/type/JSON value deduplicates; changed content conflicts. | Makes accidental ID reuse visible rather than silently ignoring a different event. |
| Retry and replay | Retry transport/408/429/5xx up to three attempts at 2s/5s; failed, enabled deliveries can be replayed. | Gives a short reproducible demo; other 4xx/3xx fail immediately. |
| Local trust boundary | One fixed customer without login, loopback-only API and configured receiver origin; HMAC protocol in SPEC-010. | Clearly limits the demo and requires production auth, destination controls, and secret management later. |
| Operational view | Four defined header signals, status counts, two-second polling, stale-state banner. | Keeps the workflow interpretable without predictive health or unsupported sentiment claims. |

Exact API routes, concrete UI components, package choices, and shell commands belong in the detailed plan; they do not change the behavior above. No independent review has yet challenged this draft.

## 12. Related material

- [Assignment](https://even-foxglove-53c.notion.site/Sr-Software-Engineer-380b9f151bae81218487d84c42741d17)
- [Benji public site](https://withbenji.com/)
- [Benji Pilot dashboard overview](https://docs.withbenji.com/pilot/dashboard)
- [Provisional SDD-TDD iteration sequence](spec-process-sdd-tdd-iterations.md)
- [User stories and specification readiness](spec-design-user-stories.md)
- [Security and scalability boundaries](spec-architecture-security-and-scale.md)
