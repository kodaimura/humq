# HUMQ Overview

HUMQ is an architecture that redefines responsibilities often blurred in existing architectures into four practical rules: Handler, Usecase, Module, and Query.

It decomposes application code by responsibility to avoid the ambiguous boundaries that often appear in MVC, layered architecture, clean architecture, and DDD. HUMQ's value is the design of constraints that make placement decisions harder to distort.

## The Problem HUMQ Solves

In common architectures, teams often run into questions like these:

- How much logic is acceptable in a Controller?
- Where is the boundary between Service and Repository?
- Should business exceptions live in the Domain?
- Where should reads that span multiple tables be placed?
- Who should manage transactions?

HUMQ reduces this ambiguity by redefining these responsibilities as practical rules.

## Core Idea

At the center of HUMQ is the separation of order and chaos.

- Decide where order is protected.
- Decide where chaos is accepted.
- Isolate cross-table observation as read-only Query code.
- Do not hide consistency; make it explicit in Usecase.

HUMQ is not a design that never breaks. Real requirements, changes, and exceptions can still break parts of the system. The point is that breakage tends to gather in Usecase, so the cleanup area stays limited.

## Four-Rule Overview

| Layer | Worldview | File naming | Owns transaction boundary | Writes |
| --- | --- | --- | --- | --- |
| Handler | External order | Plural nouns | No | Not directly |
| Usecase | Intent, business unit | Verbs | Yes | Through Module |
| Module | Local table order | Singular table names | No | Single-table reads and writes through ORM or similar tools |
| Query | Observation, structural chaos | Business context names | No | No |

## Dependency Model

In HUMQ, processing usually flows like this:

```text
Single-table operation:
Handler -> Usecase -> Module

Cross-table read:
Handler -> Usecase -> Query
```

Handler calls Usecase. Usecase combines the Modules or Query it needs. Module uses persistence tools such as an ORM and provides reads and writes closed around exactly one table.

For cross-table screens, reports, and search endpoints, Usecase calls Query. Query handles reads and aggregations that span multiple tables, but it does not write and does not own transaction boundaries.

The important rule is that Modules should not directly depend on each other. When multiple Modules need to be combined, that responsibility belongs to Usecase.

## What HUMQ Protects

HUMQ protects explainable order, not perfect automatic consistency.

Instead of hiding all consistency inside a Domain or Aggregate, HUMQ exposes it in Usecase. The code shows which Modules are combined, in what order, and which range is treated as a transaction.

In exchange, designers and implementers must take responsibility for Usecase. Usecase is a flexible layer, but it is not a layer where anything goes. A Usecase may be complex, but its complexity must remain explainable as one business intent.

---

Previous: [README](../README.md) | Next: [Layer Rules](02-layer-rules.md)
