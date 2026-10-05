# AccessFlow 

A LangGraph-based agent that processes natural-language **access requests** (e.g. *"I need finance reporting system access for 10 days"*) and applies a deterministic, tier-based access policy to decide whether to approve, escalate, reject, or ask for clarification.

An LLM is used **only** to extract structured fields from free text. Every decision (user validation, resource resolution, duration math, duplicate detection, policy) is made by plain Python code, so outcomes are reproducible and auditable.

---

## Features

- **Structured extraction** of `user_id`, resource reference(s), duration, and `thread_id` from free text via Groq (`openai/gpt-oss-20b`, `temperature=0`)
- **Deterministic duration conversion** (days / weeks / months to days), never LLM arithmetic
- **Resource resolution** against a catalog by exact name or alias match
- **Clarification loop** for ambiguous resources and missing durations, with state preserved per thread
- **Precedence checks** before tier policy: inactive user and duplicate access
- **Tier-based policy engine** (tiers 1-4) with auto-approval, manager approval, security approval, or rejection
- **Approval workflow** that simulates submitting a request and later receiving an approve/deny response
- **Interleaved multi-thread handling** via a per-thread `InMemoryStore`
- **Audit log** with one entry per processed request

---

## Architecture

```
START
  |
  +-- approval_id present? --yes--> approval_handler --> log_updater --> END
  |
  no
  v
llm_call  (structured extraction)
  v
initiator  (set thread_id / user_id)
  v
initiator_conditional_router
  |-- prior status = approval_pending ----> approval_handler
  |-- prior status = awaiting_clarification -> clarification_handler --> validator
  `-- otherwise ---------------------------> validator
                                                |
                                  validation_passed? 
                                  no --> log_updater --> END
                                  yes
                                                v
                                         duplicate_checker
                                                |
                                  already has grant? 
                                  yes --> log_updater --> END
                                  no
                                                v
                                         policy_resolver --> log_updater --> END
```

### Nodes

| Node | Responsibility |
|---|---|
| `llm_call` | Extracts `user_id`, `resource_reference`, `duration_value`, `duration_unit`, `thread_id` into an `AccessRequestExtraction` model |
| `initiator` | Copies `thread_id` and `user_id` into graph state |
| `validator` | Checks user exists / is not inactive, resolves the resource, converts duration to days, and flags ambiguity or missing duration |
| `clarification_handler` | Merges a follow-up message (e.g. *"10 days."* or *"the finance one."*) with the stored state of the same thread |
| `duplicate_checker` | Rejects requests where the user already holds an active grant for the exact resource |
| `policy_resolver` | Applies tier policy and submits an approval request when needed |
| `approval_handler` | Processes `approved` / `denied` responses for a pending approval ID |
| `log_updater` | Appends a `LogEntry` for the request |

---

## Access Policy

### Precedence rules (checked before tier policy)

| Condition | Outcome |
|---|---|
| User ID not found, or user status is `inactive` | `rejected_invalid_user` |
| User already has an active grant for the exact resource | `duplicate_request` |

### Resolution failures

| Condition | Outcome |
|---|---|
| Reference matches nothing in the catalog | `not_found` |
| Reference matches multiple resources | `clarification_required` (`..._ambiguous_resource`), candidates listed |
| Duration cannot be determined | `clarification_required` (`..._missing_duration`), never defaulted |

### Tier policy

| Tier | Rule | Outcome |
|---|---|---|
| 1 | Duration <= 30 days | `auto-approved`, otherwise `manager_approval_required` |
| 2 | Duration <= 7 days **and** user's department matches resource owner | `auto-approved`, otherwise `manager_approval_required` |
| 3 | Always | `manager_approval_required` |
| 4 | Always | `security_approval_required`, or `rejected_probation` if the user is on probation |

### Duration conversion

| Unit | Days |
|---|---|
| days | 1 |
| weeks | 7 |
| months | 30 |

---

## Sample Data

The notebook ships with an in-memory dataset:

- **Users** `U001`-`U009` across Engineering, Finance, HR, Security, and Sales, with statuses `active`, `probation`, and `inactive`
- **Resources** `R001`-`R009` (Marketing Analytics Dashboard, Finance Reporting System, HR Payroll System, Production Database ReadOnly/Admin, Customer PII Vault, Internal Wiki, Sales Reporting System, and more), each with aliases, a sensitivity tier (1-4), and an owner department
- **Existing grants**: `(U002, R003)` and `(U008, R001)`

Note that `"reporting system"` is a deliberate alias of both `R003` (Finance) and `R009` (Sales), which exercises the ambiguity path.

---

## Getting Started

### Prerequisites

- Python 3.10+ (the notebook was run on 3.13)
- A [Groq API key](https://console.groq.com/)

### Installation

```bash
git clone <your-repo-url>
cd <your-repo-name>

python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install langgraph langchain langchain-groq python-dotenv pydantic ipython jupyter
```

### Configuration

Create a `.env` file in the project root:

```env
GROQ_API_KEY=your_groq_api_key_here
```

### Run

```bash
jupyter notebook prac.ipynb
```

Run all cells in order. The final cells replay a set of interleaved test events and print the audit log.

---

## Usage

Each request is a single message containing the user ID, thread ID, and request text:

```python
from langchain.messages import HumanMessage

agent.invoke({
    "messages": [HumanMessage(content="U007, T3: PII vault for 3 days")]
})
```

Expected log output:

```
user_id='U007' resource_id='R007' decision='security_approval_required'
reason='tier_4_requires_security_approval'
```

### Handling a pending approval

When a request ends in `approval_pending`, an approval ID (e.g. `AP-001`) is created. Respond to it with:

```python
agent.invoke({"approval_id": "AP-001", "outcome": "approved"})   # or "denied"
```

### Clarification flow

```text
T1  U003: "I need finance reporting system access for a while."
    -> clarification_required (missing duration)
T1  U003: "10 days."
    -> manager_approval_required (tier 2, duration > 7 days)
```

```text
T2  U001: "I need access to the reporting system for 5 days."
    -> clarification_required (ambiguous: R003, R009)
T2  U001: "the finance one."
    -> follow-up resolved against the stored thread state
```

---

## Project Structure

```
.
|-- prac.ipynb     # Full implementation: data, policy, graph, and test events
|-- .env           # GROQ_API_KEY (not committed)
`-- README.md
```

---

## Known Limitations

- **In-memory only**: `InMemoryStore`, `approval_store`, and `log` reset when the kernel restarts
- **Exact-match resource resolution**: names and aliases must match exactly after lowercasing and whitespace normalization, so phrases like *"the finance one"* depend on the LLM extracting a matching reference
- **Thread ID comes from the message text**, extracted by the LLM, rather than being passed as graph config
- **Approvals are simulated**: there is no real approver identity, notification, or expiry
- **Grants are not updated** when a request is approved; `existing_grants` is static
- **Prototype-level error handling** in `llm_call`

---

## Tech Stack

- [LangGraph](https://github.com/langchain-ai/langgraph) for stateful graph orchestration
- [LangChain](https://github.com/langchain-ai/langchain) with `langchain-groq` for the model interface
- [Groq](https://groq.com/) running `openai/gpt-oss-20b`
- [Pydantic](https://docs.pydantic.dev/) for structured output and log models
- Python 3.13, Jupyter

