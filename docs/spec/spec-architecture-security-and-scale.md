---
title: Security and scalability boundaries for webhook delivery
version: 0.2-review
date_created: 2026-09-23
last_updated: 2026-09-23
owner: Dror Elovits
tags: [architecture, security, scalability, testing, webhook]
---

# Introduction

This document records security and scalability requirements that shape the [Benji webhook take-home](https://even-foxglove-53c.notion.site/Sr-Software-Engineer-380b9f151bae81218487d84c42741d17) from the first working slice. It distinguishes what the local demo can verify from what would be needed before a production rollout. The local behavior in [SPEC-001..011](spec-design-webhook-notifications.md#4-behavior-and-data-contracts-specified-for-review) is specified for owner review; it has not been implemented or tested, and no production readiness is claimed.

## 1. Purpose and scope

The take-home must run locally without external infrastructure. It should still make unsafe behavior hard to introduce and expose the migration points for higher volume, multiple workers, and multiple customers. Security checks belong in the iteration that introduces the risk; scalability checks should exercise concurrency and backlog behavior locally while stating their limits.

## 2. Definitions

| Term | Meaning |
| --- | --- |
| SSRF | Server-side request forgery: a customer-provided URL causes the server to call an internal or otherwise forbidden destination. |
| At-least-once delivery | A logical delivery can produce multiple HTTP attempts after failures or uncertain timeouts. |
| Backpressure | Bounded intake or dispatch so slow receivers cannot consume unbounded worker, memory, or storage capacity. |
| Queue lag | Time between a delivery becoming due and an attempt beginning. |
| Local evidence | Behavior reproduced on the local stack, receiver, and test data. It does not establish production throughput or availability. |

## 3. Requirements, constraints, and guidelines

### Security requirements

- **SEC-001 (Specified for local review; production gap):** Endpoint URL policy is enforced at registration and dispatch. The local demo permits only the configured loopback receiver origin, with differing paths, as in SPEC-003. A production policy would require HTTPS and block loopback, private, link-local, metadata, and other non-public targets. It must validate every resolved address and ensure the connection uses a validated address rather than resolving an unchecked one later; redirects are not followed. The local policy must fail closed outside local mode. This addresses the webhook-specific SSRF risk described by [OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html).
- **SEC-002 (Proposed):** Each endpoint has a distinct signing secret. Each attempt signs a fresh timestamp, stable delivery ID, and the exact raw request body using an unambiguous byte format. The receiver verifies the signature, allowed timestamp window, and delivery-ID idempotency. Reject modified bodies or delivery IDs, stale signatures, and malformed headers. Raw-body integrity is a known requirement of webhook signature verification, as illustrated by [Stripe's documentation](https://docs.stripe.com/webhooks/signature).
- **SEC-003 (Proposed):** Creation responses show the secret once. Subsequent API reads and logs omit it. A production version needs managed secret protection, rotation, and revocation; these are not implied by the local database.
- **SEC-004 (Specified for local review):** Enforce the concrete limits in SPEC-002/004/007/011 for endpoint count, payload, event creation rate, HTTP timeout, retry count, and response excerpt. Never render payload HTML or include unbounded response bodies in the dashboard.
- **SEC-005 (Production gap):** Authenticate operators and authorize every event, endpoint, delivery, and replay action by customer. A single-customer local demo must be labeled as such and must not treat a browser-supplied customer ID as authority.
- **SEC-006 (Proposed):** Bound and redact sensitive payload/response data in diagnostic logs. Define retention and deletion before production use.

### Reliability and scalability requirements

- **SCL-001 (Proposed):** Persist event acceptance and its eligible logical deliveries together before dispatch so a crash cannot leave accepted events without work. Enforce uniqueness in the database, not only in application reads.
- **SCL-002 (Proposed):** Keep HTTP intake separate from outbound delivery execution. A local worker may share a process, but its interface should allow multiple workers and a different durable store later.
- **SCL-003 (Specified for local review; production extension):** Bound concurrent attempts to four globally and one per endpoint, with finite 2s/5s retries and exhausted failures visible for replay. A slow receiver must not block all other receivers. Add jitter and tune limits for production traffic; deterministic timing serves the local demonstration.
- **SCL-004 (Specified for local review):** Claim due work with the SPEC-008 lease so two workers do not intentionally start the same attempt; recover abandoned claims after a crash. Remote processing remains uncertain after a timeout or crash, so stable delivery IDs support receiver idempotency.
- **SCL-005 (Proposed):** Observe intake rate, success/failure counts, retry volume, oldest due delivery, queue lag, attempt latency, and per-endpoint error streaks. These signals should ground dashboard status and production alerts.
- **SCL-006 (Production gap):** Choose capacity targets, latency objectives, retention, and regional availability before claiming a production architecture. The local SQLite store is useful for a self-contained demo, but SQLite WAL permits one writer at a time and is limited to processes on one host; it is not evidence for a horizontally scaled database deployment. See [SQLite's WAL documentation](https://www.sqlite.org/wal.html).

## 4. Interfaces and data contracts

Keep a narrow boundary for destination validation, durable delivery storage, and HTTP transport. The first permits only the configured local receiver origin; the production version must enforce public HTTPS destinations and network controls. Storage owns atomic event/fan-out writes, unique delivery identity, due-work claims, and attempt history. Transport receives a validated destination and a signed byte payload with fixed timeout and redirect behavior. The detailed plan will choose exact code interfaces; avoid abstractions that exist only for hypothetical providers.

Every attempt records a stable delivery ID and an attempt ID. A retry or manual replay creates a new signature and timestamp while retaining the delivery ID. Receiver idempotency is keyed by the delivery ID, not by the attempt ID.

## 5. Acceptance criteria

- **AC-SEC-001:** Local mode accepts only documented `/webhooks/` routes on the configured loopback receiver origin; tests reject receiver administration paths, unsupported schemes, credentials in URLs, fragments, other hosts/ports, and redirects. Production destination-policy tests follow when that policy is designed, rather than claiming it exists in the local demo.
- **AC-SEC-002:** The receiver accepts a valid signed raw body and rejects a changed body or delivery ID, bad signature, stale timestamp, or unknown endpoint secret.
- **AC-SEC-003:** Secret values are absent from endpoint list/detail responses and ordinary logs after creation.
- **AC-SCL-001:** Concurrent submission of the same event identity creates one event and at most one delivery per eligible endpoint.
- **AC-SCL-002:** A slow or failing receiver does not prevent another endpoint's delivery from progressing within the configured concurrency limit.
- **AC-SCL-003:** Restarting a worker after a claimed or timed-out attempt preserves the delivery and attempt history and eventually makes due work eligible again.
- **AC-SCL-004:** A local load exercise reports actual event count, fan-out, concurrency, queue lag, latency, failures, and machine/runtime settings; it makes no extrapolated production throughput claim.

## 6. Test automation strategy

| Level | Local test | What it establishes |
| --- | --- | --- |
| Policy unit tests | Inject URL parser, DNS answers, clock, and transport outcomes; test forbidden destinations, signing, timestamps, limits, and state transitions. | Deterministic decisions and failure modes. |
| Integration tests | Use temporary SQLite data and local success, slow, fail-once, and always-fail receivers; run duplicate publishers and worker restart scenarios. | Persistence, real HTTP behavior, uniqueness, retry, and recovery in the local architecture. |
| Browser tests | Trigger events, inspect live status and attempt history, and verify disabled controls and error states. | User-visible investigation behavior. |
| Local load exercise | Generate a documented event/fan-out mix, including one slow receiver; record queue lag, latency, throughput, and failures. | A reproducible local baseline and bottleneck clues only. |

Run the focused checks as each slice introduces its risk. A small local load exercise follows the required capabilities if the one-day timebox allows; it does not replace them. A final clean-setup walkthrough and relevant full suite are required before submission. Tests are planned here; none have been run yet.

## 7. Rationale and context

The first whole-system rope uses a fixed loopback receiver, avoiding arbitrary destination input. Configurable endpoints introduce SSRF and are gated on URL-policy tests. Signing and one-time secret handling arrive with the identity/trust slice; retry and worker concurrency checks arrive with failure recovery. This preserves small SDD-TDD steps without leaving the new risk untested until the end.

## 8. Dependencies and external integrations

- **EXT-001:** No hosted service is required for the take-home.
- **INF-001:** Local SQLite and a local receiver support deterministic demonstrations. Production storage and dispatch choices remain open.
- **REF-001:** [OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html).
- **REF-002:** [Stripe webhook signature verification guidance](https://docs.stripe.com/webhooks/signature) is an implementation reference, not a requirement to copy Stripe's protocol.
- **REF-003:** [SQLite WAL documentation](https://www.sqlite.org/wal.html) documents local concurrency limits.

## 9. Examples and edge cases

A configured endpoint redirects to an internal address: delivery fails without following the redirect. A public hostname resolves to a private IP at dispatch time: the production policy rejects the connection. A receiver times out after processing: the sender retries the same delivery ID with a fresh signature, and the receiver treats the repeated delivery ID idempotently. A slow receiver accumulates backlog: per-endpoint concurrency and queue-lag signals make the effect visible without blocking unrelated endpoints.

## 10. Validation criteria

Before calling a slice complete, verify the security and concurrency tests associated with its newly exposed surface. Before any production claim, replace local-only assumptions with a threat review, customer isolation, operational controls, measured capacity targets, and load testing on the intended production architecture. Keep local proof, remote CI, and production behavior distinct.

## 11. Related specifications

- [Webhook system design](spec-design-webhook-notifications.md)
- [SDD-TDD iteration plan](spec-process-sdd-tdd-iterations.md)
- [Developer journal](../process/developer-journal.md)
