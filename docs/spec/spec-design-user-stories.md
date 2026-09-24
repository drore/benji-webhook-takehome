---
title: User stories and specification readiness
version: 0.2-review
date_created: 2026-09-23
last_updated: 2026-09-24
owner: Dror Elovits
tags: [design, user-stories, traceability, readiness, take-home]
---

# User stories and specification readiness

These stories test whether the [system specification](spec-design-webhook-notifications.md) is precise enough to plan and build the Benji exercise. **Status: workflow grouping, cutover, explicit Disable, and server-generated queue event IDs with separate submission keys are agreed; the one-active-version policy is provisional and other working choices await review.** No implementation or detailed plan has begun.

## 1. Actors and scope

The local exercise has three actors: a **customer/operator** who configures named workflows and their endpoint versions and investigates delivery, a **receiver** that verifies and processes HTTP requests, and a **reviewer** who runs the system and inspects evidence. Production evolution is considered separately; no production deployment is in scope.

## 2. User stories and acceptance examples

| Story | Observable outcome | Spec coverage and testable edge outcome |
| --- | --- | --- |
| **US-01 Configure receivers** | As an operator, I create a named workflow, add immutable endpoint versions with URLs and event-type subscriptions, and activate one version for new events. A new secret appears once; later reads omit it. I can see which version replaced which and can explicitly disable or resume a version. | **Agreed, cardinality provisional:** SPEC-002/003/005/006; AC-007/008/015/016. Cutover drains accepted work; Disable pauses it and blocks replay. Revisit one-active policy before finalization. |
| **US-02 Submit an event** | As an operator, I submit a key, type, and JSON payload. The server returns a queue event ID when it accepts the event. If the submission response is lost or times out, I can resend that request with the same key and content; it returns the original ID without new deliveries. A fresh key creates a distinct event even when the content is identical. | **Identity behavior agreed:** SPEC-004/005/011; AC-002/009. This submission resend is separate from worker delivery retries. Changed type/value under one key conflicts; no-match event persists with an ID; exact input bounds remain for review. |
| **US-03 Route once per endpoint** | As an operator, I see one logical delivery to each eligible workflow's active endpoint version. Concurrent submissions with the same key, retries, and replay never create a second event–endpoint delivery; attempts belong to the existing delivery. | **Partly agreed:** SPEC-001/002/005/009; AC-001/008/010. Cutover changes future routing; A's accepted work stays assigned to A. |
| **US-04 Verify a request** | As a receiver, I verify the endpoint signature and reject a changed body, bad signature, or stale timestamp. A stable delivery ID lets me suppress repeated side effects after uncertain timeouts. | **Specified for review:** SPEC-010; AC-005/014. Header names, signed bytes, 300-second window, and receiver secret handoff are explicit. |
| **US-05 Recover automatically** | As an operator, I see a failed request and later retry on the same delivery. A fail-once receiver can succeed; an always-failing receiver reaches a terminal state. A slow receiver does not block every other delivery. | **Specified for review:** SPEC-007/008; AC-003/011. Three-attempt cycle, timeout, classifier, concurrency, and lease recovery are defined. |
| **US-06 Investigate and replay** | As an operator, I inspect a failed delivery's payload and full attempt history, replay it, and see a new attempt without losing prior evidence. Invalid replay is rejected in the API and unavailable in the UI. | **Specified for review:** SPEC-006/009, dashboard action states; AC-004/012/016. Cutover alone does not block replay to A; explicit Disable does until resumed. |
| **US-07 Understand current state** | As an operator, I select an event and see its type, payload summary, receiver branches, and text/icon/color statuses. A branch opens full JSON and attempts. Changes appear while the page stays open. | **Specified for review:** dashboard workflow outcome, header definitions, and polling contract; AC-006/009/013. Mixed, no-match, stale, and many-branch cases have stated presentations. |
| **US-08 Complete actions confidently** | As an operator, endpoint creation, event submission, and replay show pending state, prevent accidental double action, and report recoverable errors consistently. Empty views suggest the next action. | **Specified for review:** SPEC-011 and dashboard action states; AC-012/013. Safe error codes and retry affordances are defined. |
| **US-09 Demonstrate locally** | As a reviewer, I start the app and an independent receiver from documented commands. Through the UI I show success, retry, terminal failure, replay, signature verification, and deduplication, then run meaningful tests. | **Specified for review:** demo receiver procedure; AC-014. Exact commands will be selected in the detailed plan and recorded in the README. |
| **US-10 Understand production limits** | As a reviewer, I can distinguish locally exercised safeguards from work needed for multi-customer production, larger workloads, and secure network delivery. | **Specified for review:** SPEC-001/003/008 and [security and scale boundaries](spec-architecture-security-and-scale.md). Local checks cannot establish production capacity or auth. |
| **US-11 Compare endpoint versions** | As an operator, I open “Updating loyalty points — Company A,” see A and B in the same workflow with B's predecessor link, and inspect each version's delivery history. Activating B does not rewrite A's past events or send the same new event to both versions. | **Agreed in principle:** DEC-003, SPEC-002/005, AC-015. The relationship and histories support later before/after evaluation; a scored comparison interface is outside the one-day scope. |

## 3. Requirement traceability

*Specified for review* means the intent has an expected outcome in the working spec. It does not mean implementation proof or owner approval.

| Requirement | Stories | Readiness |
| --- | --- | --- |
| REQ-001 subscribed endpoints | US-01, US-03, US-11 | Agreed, cardinality provisional: SPEC-002/005/006 |
| REQ-002 retries | US-05 | Specified for review: SPEC-007/008 |
| REQ-003 authenticated delivery | US-04, US-09 | Specified for review: SPEC-010 |
| REQ-004 failures, inspection, replay | US-05, US-06 | Specified for review: SPEC-007..009 |
| REQ-005 unique logical delivery | US-02, US-03 | Specified for review: SPEC-004/005/009 |
| REQ-006 endpoint UI, secret once | US-01 | Agreed, cardinality provisional: SPEC-002/005/006 |
| REQ-007 trigger, payload, deduplication | US-02, US-09 | Specified for review: SPEC-004/005 |
| REQ-008 live state, detail, replay | US-06, US-07 | Specified for review: SPEC-009 and dashboard contract |
| REQ-009 async UX and separation | US-08 | Specified for review: SPEC-011 and dashboard action states |
| CON-001 local and one-day scope | US-09, US-10 | Specified for review: local/demo and production boundary |
| CON-002 plan, tests, README, AI trail | US-09, US-10 | Specified for review: deliverable criteria; artifacts still pending |
| USR-001 named workflow and version lineage | US-01, US-11 | Agreed; one-active rule provisional: DEC-003, SPEC-002/005, AC-015 |
| USR-002 server-generated queue event ID and submission key | US-02, US-03 | Agreed: SPEC-004/005, AC-002/010; exact bounds remain for review |

**Coverage result:** every assignment requirement and the workflow/version relationship have a story. **Readiness result:** explicit Disable and paused-replay outcomes are specified, and server-generated queue IDs and submission-key deduplication are traced to US-02. The one-active-version policy still needs review, along with the other working choices. The behavior draft is not final; tests and runtime evidence remain future work.

## 4. Decisions for owner review before the detailed plan

The [system spec's decision table](spec-design-webhook-notifications.md#11-validation-criteria-and-review-decisions) records workflow/version cutover, explicit Disable, and event identity as agreed. Review the remaining delivery, trust, operator, and proof choices one at a time; revisit the one-active-version policy before finalization. A change to any material choice updates its acceptance criterion and affected story before planning.

Exact CSS values, decorative motion, implementation-specific API paths, and command syntax can wait for the detailed plan because their choice does not change these user outcomes.

## 5. Spec readiness gate

The spec is ready for a **detailed development plan** when Dror has reviewed the material working choices; each story has one accepted outcome for its relevant normal, duplicate, failure, and recovery paths; every assignment requirement maps to accepted criteria; the local versus production boundary is explicit; and the independent demo can be described from a clean start. The detailed plan can then choose exact interfaces, iteration tasks, tests, estimates, and cuts. Implementation follows that plan in small SDD-TDD slices.

The current [iteration sequence](spec-process-sdd-tdd-iterations.md) is provisional. No implementation has begun.
