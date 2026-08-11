# HUMQ Overview

> Japanese: [01-overview.ja.md](01-overview.ja.md)

HUMQ is an architecture made of four layers: Handler, Usecase, Module, and Query.

It decomposes application code into four responsibilities to avoid the ambiguous boundaries that often appear in MVC, layered architecture, clean architecture, and DDD.

## The Problem HUMQ Solves

In common architectures, teams often run into questions like these:

- How much logic is acceptable in a Controller?
- Where is the boundary between Service and Repository?
- Should business exceptions live in the Domain?
- Where should reads that span multiple tables be placed?
- Who should manage transactions?

HUMQ reduces this ambiguity by fixing the responsibility of each layer.

## Core Idea

At the center of HUMQ is the separation of order and chaos.

- Decide where order is protected.
- Decide where chaos is accepted.
- Isolate cross-cutting observation as read-only Query code.
- Do not hide consistency; make it explicit in Usecase.

HUMQ is not a design that never breaks. Real requirements, changes, and exceptions can still break parts of the system. The point is that breakage tends to gather in Usecase, so the cleanup area stays limited.

## Four-Layer Overview

| Layer | Worldview | File naming | Transactions | Writes |
| --- | --- | --- | --- | --- |
| Handler | External order | Plural nouns | No | Not directly |
| Usecase | Intent, business unit | Verbs | Yes | Through Module |
| Module | Internal laws | Singular nouns | No | Through ORM or similar tools |
| Query | Observation, structural chaos | Business context names | No | No |

## Dependency Model

In HUMQ, processing usually flows like this:

```text
Handler -> Usecase -> Module
                   -> Query
```

Handler calls Usecase. Usecase combines the Modules and Queries it needs. Module uses persistence tools such as an ORM and provides operations closed around one subject. Query handles reads and aggregations that span multiple tables.

The important rule is that Modules should not directly depend on each other. When multiple Modules need to be combined, that responsibility belongs to Usecase.

## What HUMQ Protects

HUMQ protects explainable order, not perfect automatic consistency.

Instead of hiding all consistency inside a Domain or Aggregate, HUMQ exposes it in Usecase. The code shows which Modules are combined, in what order, and which range is treated as a transaction.

In exchange, designers and implementers must take responsibility for Usecase. Usecase is a flexible layer, but it is not a layer where anything goes.
