# AccessFlow

A progressive, hands-on LangGraph learning project: an internal IT access-request processor built in phases to surface — and force a real decision on — the core architectural questions LangGraph is designed to solve.

## What It Is

AccessFlow resolves free-text employee requests ("I need the finance dashboard for two weeks") against a deterministic policy engine, manages multi-turn clarification and asynchronous human approval through persistent thread-scoped state, provisions access via an unreliable external service with retry/fallback, and supports LLM-judged policy-exception appeals.

The project is intentionally built to **not** be fully agentic. Some parts are strict deterministic logic (policy evaluation, retry counts, authorization checks). Some parts genuinely need an LLM (resolving ambiguous free text, judging whether an appeal justification meets exception criteria). Figuring out which is which — and where the boundary between orchestration and agent reasoning actually sits — is the point of the project.

## Phases

| Phase | Adds |
|---|---|
| **1** | Core request pipeline: resolve free text → structured facts (user, resource, duration) → apply a deterministic policy table → produce a decision and log entry. |
| **2** | Multi-turn continuation and async human approval: thread-scoped state that survives across messages, correlating follow-ups and approval responses back to the correct pending request without ever using `user_id` as a correlation key. |
| **3** | Provisioning via an unreliable external service: fixed retry/fallback logic, resource-level secondary confirmation, and an LLM-generated audit justification that has zero influence on control flow. |
| **4** | Multi-resource requests (one message → multiple independent sub-requests) and an appeals process: a fixed, content-blind eligibility gate followed by genuine LLM judgment against prose policy-exception criteria. |

Each phase extends the same system — by the end, it's one coherent application, not four separate exercises.

## Scope of This Repo / Chat Structure

This project was deliberately developed across three separate roles, kept strictly apart:

1. **Project definition** — phases, requirements, mock data, test cases, completion criteria. No implementation, no hints, no revealing which LangGraph concept a phase is "supposed" to teach.
2. **Learning support** — conceptual explanations and unblocking, without supplying complete solutions.
3. **Verification** — implementation review, test execution, and pass/fail determination against each phase's stated requirements.

The architecture (state design, graph structure, nodes, edges, where persistence/interrupts live) is intentionally **not prescribed** anywhere in the phase specs — that's the part left for implementation to discover.

## Status

Phase definitions 1–4 complete. Implementation in progress.
