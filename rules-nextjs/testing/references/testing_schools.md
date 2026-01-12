# Rule 3.5: Chicago School vs London School - Deep Dive into Philosophy

### Beyond Tools: It's About System Design

The distinction between Chicago and London schools is not just about "Mock vs Real Object". It's a fundamental difference in **how you view the system** and **what you consider the unit of verification**.

### Chicago School (Classical TDD): "Objects are State Islands"

**Core Philosophy:** **Verification of State & Invariants**
- **View of System:** A collection of objects that maintain their own internal consistency.
- **Design Direction:** **Inside-Out**. Start with the domain model (the heart of the system) and build outwards.
- **Focus:** "I don't care *who* you talk to. I care about *how you change* and *what you return*."
- **Unit of Isolation:** The **Unit Test** is isolated, not the **Object**. It's okay for an object to talk to real neighbor objects as long as they are fast and deterministic.

**Why use it?**
- To ensure **Encapsulation**: You verify the contract (input/output), not the implementation details.
- To maintain **Invariants**: You ensure the object is always in a valid state.
- **Quote:** *"If the result is correct, I don't ask about the process."*

**Example:**
Testing a `User` entity. You don't mock its `Wallet`. You give it a real `Wallet`, deposit money, and check the final balance.

### London School (Mockist TDD): "Objects are a Network of Messages"

**Core Philosophy:** **Verification of Behavior & Collaboration**
- **View of System:** A network of collaborating objects sending messages to each other.
- **Design Direction:** **Outside-In**. Start at the system boundary (API) and discover the collaborators needed to fulfill the request.
- **Focus:** "I don't care about your internal state. I care about *who you delegate responsibility to*."
- **Unit of Isolation:** The **Object** is isolated. All neighbors are replaced with Mocks to verify the **Protocol** (Interactions).

**Why use it?**
- To design **Interfaces**: You discover what collaborators you need and what methods they should have *while* writing the test.
- To verify **Responsibilities**: You ensure the object delegates the right work to the right collaborator.
- **Quote:** *"I cannot do this alone. My identity is defined by who I work with."*

**Example:**
Testing a `TransferService`. You don't care about the balance. You Mock the `SourceAccount` and `DestinationAccount` to verify that `withdraw` was called on source and `deposit` was called on destination.

### Pragmatic Decision Guide: Verify Intent, Not Just Tools

Don't choose based on "Is it an external service?". Choose based on **"What am I trying to verify?"**

**Decision Tree:**

```
What is the primary intent of this test?
│
├── 1. Verifying Logic & Data Transformation? (Most Common)
│   │
│   └── Strategy: Chicago School (State Verification)
│       │
│   	├── Is the dependency fast/in-memory? (e.g., Entities, Value Objects)
│       │   └── Use Real Objects. (Verify final state)
│       │
│       └── Is the dependency slow/external? (e.g., DB, API)
│           └── Use Fake Object. (e.g., FakePaymentGateway, FakeRepository)
│               └── Implements real interface, stores state in-memory
│               └── Verify final state of the system or the Fake
│
└── 2. Verifying Orchestration & Side Effects?
    │
    └── Strategy: London School (Behavior Verification)
        │
        └── Use Mocks to verify the Protocol.
            (Did I call the logger? Did I send the email? Did I commit the transaction?)
```

**Key Insight:**
- **Fake Objects > Mocks as Stubs:** Fakes implement the real interface and behave like the real thing (but faster). They are preferred over "Mock as Stub" because they ensure contract compliance and enable state verification.
- **Prefer Chicago Style (State Verification)** because it decouples tests from implementation details.
- **Reserve London Style (Behavior Verification)** for when the *collaboration itself* is the business logic.
