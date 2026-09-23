---
title: User stories and specification readiness
version: 0.1-draft
date_created: 2026-09-23
last_updated: 2026-09-23
owner: Dror Elovits
tags: [design, user-stories, traceability, readiness, take-home]
---

# User stories and specification readiness

These stories test whether the [system specification](spec-design-webhook-notifications.md) is precise enough to plan and build the Benji exercise. **Status: the product specification is not yet ready for a detailed development plan.** A *partial* story has a covered happy path but an unresolved outcome that could change the API, persistence model, worker, or dashboard. Candidate answers remain proposals until recorded in the system spec.

## 1. Actors and scope

The local exercise has three actors: a **customer/operator** who configures endpoints and investigates delivery, a **receiver** that verifies and processes HTTP requests, and a **reviewer** who runs the system and inspects evidence. Production evolution is considered separately; no production deployment is in scope.

## 2. User stories and acceptance examples

| Story | Observable outcome | Coverage and remaining question |
| --- | --- | --- |
| **US-01 Configure receivers** | As an operator, I register multiple URLs and event-type subscriptions, then enable or disable endpoints. A new secret appears once; later reads omit it. A disabled endpoint receives no new delivery. | **Partial:** REQ-001/006, AC-001/007. What URL policy applies locally? Does disabling pause existing queued/retrying work? Can later edits affect an existing delivery? |
| **US-02 Submit an event** | As an operator, I enter an event ID, type, and JSON payload and see accepted or deduplicated feedback. Submitting the same ID and content again creates no new delivery. | **Partial:** REQ-007, AC-002. What if content changes for the same ID? What if no endpoint matches? What input bounds apply? |
| **US-03 Route once per endpoint** | As an operator, I see one logical delivery to each eligible endpoint. Concurrent duplicate submissions, retries, and replay never create a second event–endpoint delivery; attempts belong to the existing delivery. | **Partial:** REQ-001/005, DES-002. Are URL, subscription, and secret snapshotted when an event is accepted? What is the local customer boundary? |
| **US-04 Verify a request** | As a receiver, I verify the endpoint signature and reject a changed body, bad signature, or stale timestamp. A stable delivery ID lets me suppress repeated side effects after uncertain timeouts. | **Partial:** REQ-003, AC-005. Specify exact signing bytes, headers, timestamp tolerance, and one-time secret handoff. Does replay keep the same delivery ID? |
| **US-05 Recover automatically** | As an operator, I see a failed request and later retry on the same delivery. A fail-once receiver can succeed; an always-failing receiver reaches a terminal state. A slow receiver does not block every other delivery. | **Partial:** REQ-002/004, AC-003. Which outcomes retry, with what timeout/delay/max? How does a restarted worker recover `in_progress` work? |
| **US-06 Investigate and replay** | As an operator, I inspect a failed delivery's payload and full attempt history, replay it, and see a new attempt without losing prior evidence. Invalid replay is rejected in the API and unavailable in the UI. | **Partial:** REQ-004/008, AC-004. May succeeded or disabled-endpoint deliveries replay? What happens if replay is requested twice while work is pending? |
| **US-07 Understand current state** | As an operator, I select an event and see its type, payload summary, receiver branches, and text/icon/color statuses. A branch opens full JSON and attempts. Changes appear while the page stays open. | **Partial:** REQ-008, DEC-001, AC-006. Define mixed outcome aggregation, no-match and polling-error views, many-receiver layout, and included header metrics. |
| **US-08 Complete actions confidently** | As an operator, endpoint creation, event submission, and replay show pending state, prevent accidental double action, and report recoverable errors consistently. Empty views suggest the next action. | **Partial:** REQ-009. Define API error categories and UI responses for validation, conflict, network, and background delivery errors. |
| **US-09 Demonstrate locally** | As a reviewer, I start the app and an independent receiver from documented commands. Through the UI I show success, retry, terminal failure, replay, signature verification, and deduplication, then run meaningful tests. | **Partial:** CON-001/002, AC-001..007. Specify receiver secret setup and reproducible commands/assertions without hidden state or external services. |
| **US-10 Understand production limits** | As a reviewer, I can distinguish locally exercised safeguards from work needed for multi-customer production, larger workloads, and secure network delivery. | **Partial:** DES-006, [security and scale boundaries](spec-architecture-security-and-scale.md). Specify local auth/customer scope, representative load/failure checks, and supported claims. |

## 3. Requirement traceability

*Mapped* means the intent appears in a story; it does not mean the behavior is fully specified.

| Requirement | Stories | Readiness |
| --- | --- | --- |
| REQ-001 subscribed endpoints | US-01, US-03 | Partial: edits and existing deliveries |
| REQ-002 retries | US-05 | Partial: policy and recovery |
| REQ-003 authenticated delivery | US-04, US-09 | Partial: wire protocol and setup |
| REQ-004 failures, inspection, replay | US-05, US-06 | Partial: replay and recovery |
| REQ-005 unique logical delivery | US-02, US-03 | Partial: concurrency and snapshots |
| REQ-006 endpoint UI, secret once | US-01 | Partial: disable and URL policy |
| REQ-007 trigger, payload, deduplication | US-02, US-09 | Partial: conflicting duplicate and no match |
| REQ-008 live state, detail, replay | US-06, US-07 | Partial: aggregation and failure display |
| REQ-009 async UX and separation | US-08 | Partial: errors and recovery |
| CON-001 local and one-day scope | US-09, US-10 | Partial: clean setup and boundary |
| CON-002 plan, tests, README, AI trail | US-09, US-10 | Partial: evidence after spec closure |

**Coverage result:** every assignment requirement has a story. **Readiness result:** the normal paths are described, but every story still has at least one open behavior or proof question. This is a coverage pass, not spec approval.

## 4. Decisions needed before the detailed plan

Resolve these in the [system specification](spec-design-webhook-notifications.md) with explicit expected outcomes. The order reflects how many later choices each answer affects.

1. **Identity and routing:** customer scope, conflicting duplicate, no-match event, and endpoint configuration snapshot (US-01..03).
2. **Delivery lifecycle:** retryable outcomes, timeout, schedule, maximum, disable behavior, restart recovery, and replay eligibility/concurrency (US-01, US-05, US-06).
3. **Trust boundary:** accepted local URLs, exact signing protocol and tolerance, and independent receiver secret setup (US-01, US-04, US-09).
4. **Operator contract:** aggregate event status, first-version header signals, errors, payload bounds/presentation, and polling failure behavior (US-02, US-07, US-08).
5. **Proof boundary:** clean-start walkthrough, local concurrency/failure checks, and explicit production limitations (US-09, US-10).

Exact CSS values, decorative motion, and implementation-specific API path names can wait for the plan because their choice does not change these user outcomes. If a header metric is promised in the first version, its calculation and drill-down must be specified first.

## 5. Spec readiness gate

The spec is ready for a **detailed development plan** when each story has one unambiguous expected outcome for its relevant normal, duplicate, failure, and recovery paths; every assignment requirement maps to accepted criteria; the local versus production boundary is explicit; and the independent demo can be described from a clean start. The detailed plan can then choose exact interfaces, iteration tasks, tests, estimates, and cuts. Implementation follows that plan in small SDD-TDD slices.

The current [iteration sequence](spec-process-sdd-tdd-iterations.md) is provisional. No implementation has begun.
