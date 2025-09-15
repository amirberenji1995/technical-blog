# 📝 Why Finite State Machines Matter in Real-World Software

#### 📢 Attention! This post is co‑authored by GPT‑4o from OpenAI.

---

## 🔰 What Is a Finite State Machine?

A **Finite State Machine (FSM)** is a design model used to describe the behavior of systems through **a set of defined states and rules for transitioning between them**. It consists of:

1. A **set of states** (`S`)
2. A **set of transitions** (`T`) between states
3. A **starting (initial) state**
4. One or more **final (terminal) states**

FSMs are generally visualzied using diagrams similar to the below one:

```
Abstract FSM Diagram

  [Start]
     |
     v
  [State A] ---> [State B]
     |               |
     v               v
  [State C] ---> [State D]
                     |
                     v
               [Final State]
```

* States are in brackets `[STATE]`
* Arrows `→` represent valid transitions

Great insight — that section definitely benefits from better context-setting, narrative flow, and visual clarity. Here's the **enriched** version of the section, incorporating:

* A clear narrative introduction to the loan processing use case
* The **updated FSM diagram** (from earlier)
* Transitions and business rules in a well-explained flow
* Technical examples tightly coupled with the FSM reasoning

---

## 🏗️ Modeling Loan Request Processing with FSM

Let’s bring the theory of Finite State Machines (FSMs) into the real world through a domain-driven case study: **processing a bank loan request**.

### 🏦 The Use Case

Imagine a digital platform where customers apply for loans and bank staff handle their evaluation. A loan doesn’t get approved in one go — it passes through multiple business gates:

1. Background verification
2. Document certification
3. Risk evaluation
4. Approval and disbursement

At any stage, the request could also be rejected based on validation results, missing documentation, or failed risk assessments. This makes it a **perfect fit** for an FSM — each step represents a **state**, and transitions are governed by business rules.

---

### 🔁 The Finite State Machine in Practice

Below is the FSM diagram describing the process:

```
LoanRequest FSM Diagram

  [START]
     |
     v
 [SUBMITTED]--------------------------------+
     |                                      |
     v                                      |
[BACKGROUND_CHECK]----------------------+   |
     |                                  |   |
     v                                  |   |
[CERTIFICATED]----------------------+   |   |
     |                              |   |   |
     v                              V   V   V
[RISK_ASSESSED]------------------> [REJECTED]
     |
     v
[APPROVED]
     |
     v
[ALLOCATED]
     |
     v
 [COMPLETED]
```

✅ Highlights:

* Every stage clearly maps to a **discrete state**
* Possible **rejections** are handled gracefully via skip transitions
* Final states: either `COMPLETED` (success) or `REJECTED` (failure)

---

### 🧱 The Core Entity: `LoanRequest`

Here’s the model representing a loan request in your application:

```python
from enum import StrEnum
from pydantic import BaseModel


class LoanRequestStatus(StrEnum):
    SUBMITTED = "SUBMITTED"
    BACKGROUND_CHECK = "BACKGROUND_CHECK"
    CERTIFICATED = "CERTIFICATED"
    RISK_ASSESSED = "RISK_ASSESSED">
    APPROVED = "APPROVED"
    ALLOCATED = "ALLOCATED"
    COMPLETED = "COMPLETED"
    REJECTED = "REJECTED"


class LoanRequest(BaseModel):
    id: str
    applicant_name: str
    amount: float
    status: LoanRequestStatus = LoanRequestStatus.SUBMITTED
```

This `LoanRequest` object is the **heart** of the state machine. Its `status` property must be changed **only** through valid transitions.

---

### 🔁 Valid State Transitions

Here’s the complete FSM logic encoded as a Python dictionary:

```python
status_mapping: dict = {
    LoanRequestStatus.SUBMITTED: [
        LoanRequestStatus.BACKGROUND_CHECK,
        LoanRequestStatus.REJECTED,
    ],
    LoanRequestStatus.BACKGROUND_CHECK: [
        LoanRequestStatus.CERTIFICATED,
        LoanRequestStatus.REJECTED,
    ],
    LoanRequestStatus.CERTIFICATED: [
        LoanRequestStatus.RISK_ASSESSED,
        LoanRequestStatus.REJECTED,
    ],
    LoanRequestStatus.RISK_ASSESSED: [
        LoanRequestStatus.APPROVED,
        LoanRequestStatus.REJECTED,
    ],
    LoanRequestStatus.APPROVED: [
        LoanRequestStatus.ALLOCATED,
    ],
    LoanRequestStatus.ALLOCATED: [
        LoanRequestStatus.COMPLETED,
    ],
    LoanRequestStatus.COMPLETED: [],
    LoanRequestStatus.REJECTED: [],
}
```

This simple structure allows your processor to **enforce transitions**, reducing errors and maintaining state integrity.

---

### 🧠 Business-Aware Transition Logic: `LoanRequestProcessor`

Let’s define a processor that takes a loan and processes a status update using FSM rules.

```python
class LoanRequestProcessSchema(BaseModel):
    target_status: LoanRequestStatus
    reviewed_by: str
    notes: str | None = None
```

```python
class LoanRequestProcessor:
    def __init__(self, loan_request: LoanRequest):
        self.loan_request = loan_request

    def process(self, payload: LoanRequestProcessSchema) -> LoanRequest:
        current = self.loan_request.status
        next_status = payload.target_status
        valid_next = status_mapping.get(current, [])

        if next_status not in valid_next:
            raise ValueError(f"❌ Invalid transition: {current} → {next_status}")

        # Optional business rule: high-value loans require manual board approval
        if (
            next_status == LoanRequestStatus.RISK_ASSESSED
            and self.loan_request.amount > 100_000
        ):
            raise ValueError("❌ High-risk loans above $100k require manual board approval.")

        # All clear — transition allowed
        self.loan_request.status = next_status
        return self.loan_request
```

---

### 🚦 Why This FSM Matters

This FSM pattern provides several tangible benefits:

* ✅ **Safety** — Invalid transitions are blocked programmatically.
* 🧩 **Extensibility** — Adding new steps (e.g., fraud checks) is straightforward.
* 🔍 **Auditability** — You can easily track state changes.
* 🔐 **Encapsulation** — Business logic stays near the data it affects.

---

## 🧰 Benefits of FSMs in This Context

| Benefit                | Why It Matters                                             |
| ---------------------- | ---------------------------------------------------------- |
| ✅ Predictable Workflow | All transitions are explicit and well-defined.             |
| ✅ Auditability         | Logs can easily trace each transition in order.            |
| ✅ Maintainability      | Adding a new state like `LEGAL_REVIEW` is straightforward. |
| ✅ Error Prevention     | Impossible transitions raise meaningful errors.            |

---

## ✍️ Conclusion

While Finite State Machines may sound academic, they are incredibly **practical tools** for structuring business workflows—especially in backend systems dealing with stages, approvals, and validations.

In our loan request case study, FSMs gave us:

* A clean map of what’s allowed and what’s not
* Clear enforcement of transitions
* Simulated risk and verification logic embedded in process flow

> If your application models processes, approvals, or evolving states — **consider adopting an FSM-based design.** You’ll save yourself a ton of bugs, confusion, and inconsistent states.