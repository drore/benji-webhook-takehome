---
title: Provisional SDD-TDD iteration sequence for the webhook take-home
version: 0.1-draft
date_created: 2026-09-23
last_updated: 2026-09-23
owner: Dror Elovits
tags: [process, sdd, tdd, iterations, take-home]
---

# Introduction

This document sketches the possible order for Dror's specification-driven development (SDD) and test-driven development (TDD) approach to the [Benji webhook assignment](https://even-foxglove-53c.notion.site/Sr-Software-Engineer-380b9f151bae81218487d84c42741d17). **It is not the detailed development plan and does not authorize implementation.** First finalize the [system design spec](spec-design-webhook-notifications.md) using the [user-story readiness review](spec-design-user-stories.md); then write the detailed plan. No slice has been implemented.

## 1. Purpose and scope

Build in small, working iterations at **two scales**:

1. **Whole-system scale:** establish a thin, real path from the dashboard to the API, local persistence, HTTP delivery, a local receiver, and visible status. This is the first crossing between the banks.
2. **Feature scale:** for each new behavior, specify its observable result, write a failing test, make it work with the smallest implementation, then improve its design while tests remain green.

The order favors the large shape of the system before refining a small detail. A polished component does not count as progress if the complete user path is still broken. The one-day assignment limit remains the timebox.

## 2. Definitions

| Term | Meaning |
| --- | --- |
| SDD | Specify behavior and acceptance evidence before implementation. |
| TDD | Write a meaningful failing test, implement enough to pass, then refactor with the test green. |
| Slice | A small, demonstrable increment that crosses the layers needed for its behavior. |
| System rope | The smallest honest end-to-end path through the whole product. |
| Feature rope | The smallest working form of one behavior before its edge cases or presentation are refined. |

## 3. Requirements, constraints, and guidelines

- **PRO-001 (Chosen):** Every iteration names the user-visible behavior, minimum interfaces, acceptance evidence, and what is deliberately deferred.
- **PRO-002 (Chosen):** For a behavior change, create a failing test at the lowest useful level, implement the minimum, then refactor. Add end-to-end evidence when the slice crosses processes or UI boundaries.
- **PRO-003 (Chosen):** Keep a runnable path across the whole system after the first product slice. Avoid isolated polish that does not improve or verify that path.
- **PRO-004 (Chosen):** Complete the broad functional shape before refining visual details, animation, metrics, or abstraction.
- **PRO-005 (Chosen):** End an iteration with a demonstrable result, passing relevant tests, updated spec/journal, and an explicit next slice.
- **CON-001 (Required):** The completed submission must satisfy the assignment within one day of focused work and run locally without external infrastructure.

## 4. Interfaces and data contracts

Product behavior and externally visible contract semantics are finalized in the specification before writing the detailed plan. The detailed plan will choose concrete API paths, field names, tests, and slice boundaries. The first slice will need only enough fields to identify an event, a delivery, its endpoint, and its current outcome; later slices can extend the same path while preserving the approved behavior. The [system design spec](spec-design-webhook-notifications.md#4-behavior-and-data-contracts-specified-for-review) records the current behavior and conceptual entities.

Each new contract must state how failure appears to callers and how the dashboard reflects the server's authoritative state. A frontend-only simulation does not satisfy an integration slice.

## 5. Planned iterations

| Iteration | Smallest complete behavior | Acceptance evidence | Deferred until later |
| --- | --- | --- | --- |
| **0. Working skeleton** | Start Python API, Vue dashboard, and local receiver with one documented command sequence. | Health/readiness checks and a browser page load. | Delivery behavior, visual polish. |
| **1. Whole-system rope** | From the dashboard, submit one test event; the API persists it and performs one real HTTP POST to one fixed local receiver; the dashboard shows the server-reported result. | One integration test plus a manual UI-to-receiver walkthrough. | Configurable endpoints, retry, signing, rich workflow. |
| **2. Configurable fan-out** | Create multiple endpoints and subscriptions; a matching event creates a delivery only for each eligible endpoint; enable/disable affects new routing and pauses existing queued work. | Routing and pause/resume tests plus a UI walkthrough with two receivers. | Automatic retry implementation. |
| **3. Identity and trust** | Re-submitting an event ID creates no new deliveries; each event–endpoint pair is unique; receiver verifies a signed request. | Duplicate/conflict and signature-valid/tampered tests; one-time secret display. | Failure recovery and visual polish. |
| **4. Failure and recovery** | Failed HTTP attempts remain visible, retry on a bounded schedule, and exhausted delivery can be replayed without losing history. | Deterministic retry, replay, timeout, and restart tests; fail-once and always-fail receiver walkthrough. | Detailed dashboard refinement. |
| **5. Operational experience** | Event-centered fan-out view, four specified header signals, status polling, payload/attempt details, and loading, empty, stale, and error states. | Browser-level flows for status change and investigation; accessibility and usability review. | Decorative motion and speculative health metrics. |
| **6. Submission finish** | Polish the established workflow using Benji-inspired visual language; complete README, plan, AI-usage note, and final demonstration. | Run all required checks on the final candidate and rehearse the full local demo. | Production-scale features documented as cuts. |

The table is a proposed ordering, not a demand for seven large releases. If time is tight, reduce optional polish before cutting required reliability, signing, replay, or inspection. Tests accompany each slice; iteration 6 is final verification, not the first testing pass.

The [security and scalability boundaries](spec-architecture-security-and-scale.md) apply at the moment a slice introduces a risk: destination policy with configurable URLs, secret handling and signature verification with identity/trust, and worker concurrency and recovery with retries. The first rope uses one fixed loopback receiver. Local concurrency and load checks show behavior and bottlenecks; they do not establish production capacity.

## 6. Acceptance criteria

- **AC-001:** After iteration 1, a reviewer can see the same event travel from the dashboard through the backend to a separate local receiver and back to a visible status, using actual HTTP.
- **AC-002:** At each later iteration, the prior end-to-end path still works and the new behavior has a failing-then-passing test recorded in the development process.
- **AC-003:** An iteration is not declared complete from mocked UI state alone when its contract requires real backend or receiver behavior.
- **AC-004:** Before submission, all assignment capabilities have demonstrated evidence and the final README reproduces the walkthrough from a clean local setup.

## 7. Test automation strategy

Use focused unit tests for policy and signing, integration tests for SQLite/worker/receiver boundaries, and a small number of browser tests for observable dashboard behavior. Test meaningful failure modes as they arise in their slice. Record test commands and results in the journal; do not add tests that merely mirror implementation structure.

## 8. Rationale and context

The bridge analogy applies to the whole product and to each feature. The marble-statue analogy clarifies sequencing: establish the large form before refining a small visual or technical detail. Both imply frequent working checkpoints after the spec and detailed plan are ready. Once finalized, the spec may still be revised when implementation evidence changes a decision; record that change before the affected slice proceeds.

## 9. Dependencies and external integrations

- **EXT-001:** No hosted external service is required for the local workflow.
- **INF-001:** The first delivery slice needs a separate local receiver and durable local state.
- **PLT-001:** Python and Vue 3/TypeScript are the target stack agreed for this assignment.

## 10. Examples and edge cases

For the first system rope, one fixed receiver and one event type are sufficient if the browser triggers a real API request, the backend performs a real HTTP POST, and the UI reads the actual result. The next slice replaces the fixed receiver with customer configuration without changing the core path. For a later retry feature rope, first prove one failed attempt schedules another; then add bounded backoff, restart recovery, and status presentation.

## 11. Validation and related material

Before starting any slice, complete the [story-based spec readiness gate](spec-design-user-stories.md#5-spec-readiness-gate) and detailed development plan. During implementation, verify the tested local path and record what remains unproven. Keep the [developer journal](../process/developer-journal.md) current and maintain the corresponding Obsidian copy. See the [assignment](https://even-foxglove-53c.notion.site/Sr-Software-Engineer-380b9f151bae81218487d84c42741d17) and [system design spec](spec-design-webhook-notifications.md).
