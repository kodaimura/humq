# HUMQ Overview

HUMQ is an architecture that keeps important complexity visible by expressing business flows as explicit compositions of simple, predictable parts.

Its four responsibilities—Handler, Usecase, Module, and Query—and their mechanical boundaries fix where code and complexity belong. HUMQ's value is not reducing the number of abstractions. It is reducing the indirection needed to trace a process and the design decisions required to place code.

## The Problem HUMQ Solves

In common architectures, teams often run into questions like these:

- How much logic is acceptable in a Controller?
- Where is the boundary between Service and Repository?
- Should business exceptions live in the Domain?
- Where should reads that span multiple tables be placed?
- Who should manage transactions?

HUMQ reduces this ambiguity by redefining these responsibilities as practical rules.

## Central Idea

> **Essential complexity should be visible.**

When the business itself is complex, that complexity may appear in code. HUMQ does not aim to push business procedures behind Service or Repository abstractions merely to make the caller look short.

That does not mean exposing everything in Usecase.

| May be hidden | Must remain visible from Usecase |
| --- | --- |
| ORM and SQL construction | Operations on multiple tables |
| Database access mechanics | Business-significant order and branching |
| Local data conversion | State transitions and consistency decisions |
| External client transport details | The occurrence and failure handling of external I/O |
| Implementation details of one-table operations | Transaction boundaries |

Implementation details may be hidden. Business flows should not be hidden. This is HUMQ's position on encapsulation.

## Four-Rule Overview

| Layer | Worldview | File naming | Owns transaction boundary | Writes |
| --- | --- | --- | --- | --- |
| Handler | External order | Plural nouns | No | Not directly |
| Usecase | Intent, business unit | Verb, one flow per file | Decides the required boundaries | Through Module |
| Module | Local table order | Singular table names | No | Single-table reads and writes through ORM or similar tools |
| Query | Observation, structural chaos | Business context names | No | No |

## Dependency Model

In HUMQ, processing usually flows like this:

```text
Handler
   │
   ▼
Usecase
   ├── Module          -> Single-table Read / Write
   ├── Module          -> Single-table Read / Write
   ├── Query           -> Cross-table Read
   └── External Client -> Side Effect
```

Handler calls Usecase. Usecase explicitly composes the Modules, Query code, and external clients it needs. Module uses persistence tools such as an ORM and provides reads and writes closed around exactly one table.

For cross-table screens, reports, and search endpoints, Usecase calls Query. Query handles reads and aggregations that span multiple tables, but it does not write and does not own transaction boundaries.

The important rule is that Modules do not directly depend on each other and that business-significant composition remains visible from Usecase. Usecase owns the coordination of multiple Modules, external I/O, and state transitions.

## What HUMQ Protects

HUMQ protects explainable and traceable order, not automatic safety for every condition.

Instead of hiding all consistency inside a Domain or Aggregate, HUMQ exposes it in Usecase. The code shows which Modules are combined, in what order, and which range is treated as a transaction.

In exchange, designers and implementers must take responsibility for Usecase. A Usecase may be complex, but unrelated intents, deep nesting, and untraceable state changes are not acceptable. Judge it by whether one business intent can be explained from top to bottom, not by length alone.

## Target Systems and Design Assumptions

HUMQ primarily targets applications that use a relational database as their main persistence model and handle multi-table state changes and cross-table reads.

HUMQ intentionally couples Module boundaries to database table structure. This favors predictable operation targets and fewer boundary decisions over a rich Domain Model that is independent of persistence.

[Adoption and Tradeoffs](05-comparison.md#adoption-and-tradeoffs) explains HUMQ's benefits and drawbacks,<br>
where it fits, and when another design may be more appropriate.

> Keep the parts simple. Keep the important chaos visible.

---

Previous: [README](../README.md) | Next: [Layer Rules](02-layer-rules.md)
