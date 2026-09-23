---
title: Webhook notification system and operations dashboard
version: 0.1-draft
date_created: 2026-09-23
last_updated: 2026-09-23
owner: Dror Elovits
tags: [design, architecture, webhook, dashboard, take-home]
---

# Introduction

This is the living specification for the [Benji senior software engineer take-home](https://even-foxglove-53c.notion.site/Sr-Software-Engineer-380b9f151bae81218487d84c42741d17). It records the assignment's requested outcomes, our current design direction, and decisions still to make. It is intended to guide implementation and the final AI-assisted plan; it is not a claim that the system has been built.

**Status terms:** **Required** comes from the assignment. **Chosen** is agreed in discussion. **Proposed** is a working design subject to review. **Open** needs a decision or implementation evidence.

## 1. Purpose and scope

Build a local, one-day fullstack demonstration in which a customer registers multiple webhook endpoints, selects event types, triggers events, and inspects reliable, authenticated delivery. The audience is Benji's reviewers and an operator investigating delivery issues.

The assignment prefers Python for the backend and Vue 3 with TypeScript for the frontend. Dror is comfortable with that stack (**Chosen**). The solution must run locally without external infrastructure (**Required**). The [public repository](https://github.com/drore/benji-webhook-takehome) currently contains planning documents; code, plan, tests, run instructions, and an AI-usage note remain required for submission. Work follows the [SDD-TDD iteration plan](spec-process-sdd-tdd-iterations.md) (**Chosen process**).

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
- **DES-002 (Proposed):** Enforce a database uniqueness constraint on `(event_id, endpoint_id)`. Retries and manual replay append attempts to the existing delivery. HTTP transmission is at least once when failures or uncertain timeouts occur; receiver-side idempotency uses the stable delivery ID. Do not claim exactly-once remote processing.
- **DES-003 (Proposed):** Sign the timestamp, stable delivery ID, and exact raw request body with an endpoint-specific secret using HMAC-SHA256 and an unambiguous byte format. Send the timestamp, signature, and delivery ID as headers. The demo receiver verifies the signature and timestamp.
- **DES-004 (Proposed):** Show an event's type, ID, time, and a small payload summary on its card; open formatted full JSON on selection. A receiver branch shows a textual state, icon, and status color. A pulse represents an actual HTTP attempt; the logical-delivery branch remains stable across retries.
- **DES-005 (Proposed):** Poll the delivery API while the dashboard is open to update statuses without manual refresh. Keep event and delivery state authoritative on the server.
- **DES-006 (Proposed):** Apply the [security and scalability boundaries](spec-architecture-security-and-scale.md) as each slice introduces a new surface. The local demo does not imply production readiness.

## 4. Interfaces and data contracts

The following is a conceptual contract. Exact paths, field names, retry intervals, and error shapes remain open until the implementation plan is settled.

| Entity | Minimum fields or invariant |
| --- | --- |
| Event | `id`, `customer_id`, `type`, `payload` (JSON), `created_at`; repeated submission of the same customer/event ID does not fan out again. |
| Endpoint | `id`, `customer_id`, `url`, nonempty `event_types`, `enabled`, signing secret; list/read responses omit the secret. |
| Delivery | `id`, `event_id`, `endpoint_id`, `status`, `next_attempt_at`; unique `(event_id, endpoint_id)`. |
| Attempt | `id`, `delivery_id`, number, start/end times, HTTP status or transport error, bounded response excerpt. |

**Candidate API capabilities:** create/list/update endpoints; submit/list/read events; read delivery with attempts; replay a failed delivery. The event submission response reports whether the event was newly accepted or deduplicated. The delivery detail response contains enough data to render the workflow and investigation panel without relying on browser-side guesses.

**Candidate status vocabulary:** `pending`, `in_progress`, `retrying`, `succeeded`, `failed`. An attempt moves a delivery through these states; replay moves a failed delivery back to pending while retaining its history. Status names and restart recovery behavior require finalization.

## 5. Dashboard and interaction design

### Layout

1. **Header:** concise operational signals, each opening a relevant filtered view.
2. **Recent events:** select an event; show useful empty and loading states.
3. **Selected event workflow:** event card on the left, eligible receiver branches on the right, and a concise outcome summary.
4. **Investigation panel:** click a branch to see URL, state, payload, attempt timeline, HTTP outcomes, next retry, and replay when applicable.
5. **Management actions:** create or enable/disable endpoints and fire a test event without leaving the dashboard context.

### Candidate header signals (**Proposed**)

| Signal | Definition | Drill-down |
| --- | --- | --- |
| Needs attention | Deliveries that exhausted automatic retries. | Failed deliveries and replay action. |
| Retrying | Deliveries with another attempt scheduled; show the nearest retry time. | Scheduled deliveries. |
| Delivery rate, last 24 hours | Succeeded / completed logical deliveries; show numerator and denominator so small samples are clear. | Underlying deliveries. |
| Latest event | Event type, age, and combined outcome. | Select that event's workflow. |

**Open:** Whether the fourth tile should instead be "Receivers at risk." That label requires a precise rule based on repeated failure signals; it must not imply predictive health from one failure. "Sentiment" has no source in the assignment's webhook data and is excluded from the first version.

### Visual language (**Chosen direction, implementation tokens proposed**)

The [public Benji site](https://withbenji.com/) observed on 2026-09-23 uses Lexend for prominent headings, Inter for body copy, dark navy text (`#071437` observed on the hero heading), vivid purple (`#9900FF` observed on a primary button), white surfaces, rounded controls, light borders, and generous spacing. The [public Pilot dashboard documentation](https://docs.withbenji.com/pilot/dashboard) describes overview, activity, breakdown, and highlight patterns. These are references; the private product's design system has not been inspected.

- Use purple for selection and primary actions, with navy and neutral surfaces as the base.
- Use status colors only with text and icons: gray pending, blue in progress, amber retrying, green succeeded, red failed.
- Use a fixed fan-out layout that can scroll or collapse when many receivers exist. Avoid a freeform graph editor.
- Represent the payload as a safe text summary and formatted JSON; do not render payload content as HTML.

## 6. Demonstration and acceptance criteria

**Demo receiver (Proposed):** Run a separate local HTTP service with success, fail-once, and always-fail routes. It verifies signatures and exposes a minimal received-request view. Register its URLs through the main dashboard, using the one-time secrets to configure verification. Actual HTTP traffic and independent receiver evidence supplement automated tests.

- **AC-001:** Given two endpoints subscribed to the same type, when a matching event is submitted, then each has one logical delivery and visible status.
- **AC-002:** Given an event ID already submitted by a customer, when it is submitted again, then the dashboard reports deduplication and no new deliveries are created.
- **AC-003:** Given a receiver that fails once, when retry time arrives, then the same delivery records a second attempt and can succeed.
- **AC-004:** Given exhausted retries, when an operator replays the failure, then the same delivery gains another attempt and its earlier failures remain visible.
- **AC-005:** Given a signed request, when the receiver verifies the timestamp and raw body, then an unchanged request passes and a modified body or signature fails.
- **AC-006:** Given the dashboard remains open, when backend status changes, then visible workflow status updates without manual refresh.
- **AC-007:** Given endpoint creation succeeds, then the secret appears once; later endpoint reads do not reveal it.

## 7. Test automation strategy

- **Unit:** event-to-endpoint matching, duplicate submission, delivery state transitions, signing and verification, retry schedule.
- **Integration:** SQLite uniqueness and persistence, worker-to-local-receiver HTTP outcomes, replay retaining attempt history, and restart recovery once the worker design is finalized.
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

Cases to settle or verify: same event ID with different payload; disabled endpoint during a scheduled retry; malformed or oversized payload; unreachable or slow receiver; server restart during `in_progress`; replay after an endpoint is disabled; event with no matching endpoints; polling interruption; a zero-denominator delivery rate.

## 11. Validation criteria and open decisions

Implementation can begin with the first thin end-to-end slice once that slice's contracts and acceptance evidence are specified. Later decisions are resolved before the slice that needs them; they do not block the whole-system rope. The implementation is ready for submission only after the acceptance criteria are verified against the final local code and the required plan, README, tests, and AI-usage note are present.

| ID | Decision needed | Current position |
| --- | --- | --- |
| OPEN-001 | Retry schedule, maximum attempts, and timeout. | Use a finite deterministic schedule that is quick enough to observe in a demo. |
| OPEN-002 | Meaning of disabling an endpoint for queued/retrying deliveries. | Decide before implementing the worker and UI state. |
| OPEN-003 | Same event ID with a different payload. | Prefer a conflict response rather than silently accepting different content. |
| OPEN-004 | Replay when the endpoint is disabled or a delivery already succeeded. | Define server-side authorization and UI availability. |
| OPEN-005 | Exact header metric set and "at risk" rule. | Start with needs attention, retrying, delivery rate, and latest event. |
| OPEN-006 | Receiver secret setup for the local walkthrough. | Keep creation through the dashboard and independent receiver verification. |

## 12. Related material

- [Assignment](https://even-foxglove-53c.notion.site/Sr-Software-Engineer-380b9f151bae81218487d84c42741d17)
- [Benji public site](https://withbenji.com/)
- [Benji Pilot dashboard overview](https://docs.withbenji.com/pilot/dashboard)
- [SDD-TDD iteration plan](spec-process-sdd-tdd-iterations.md)
- [Security and scalability boundaries](spec-architecture-security-and-scale.md)
