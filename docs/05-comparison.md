# Comparison with Existing Architectures

HUMQ does not reject MVC, layered architecture, clean architecture, or DDD. Each of them can make business flows explicit in a place equivalent to Usecase.

The difference is that HUMQ provides mechanical defaults for where complexity belongs, how lower-level boundaries are chosen, and how much indirection is acceptable along an important flow.

## Assumptions Behind the Comparison

An architecture name alone does not determine an implementation. Clean architecture does not necessarily produce a `Usecase -> Service -> Domain Service -> Repository` chain, and DDD does not necessarily hide a business flow inside an Aggregate.

HUMQ addresses cases where operating another design has led to a structure like this:

```text
Usecase
   ↓
Service
   ↓
Domain Service
   ↓
Repository
   ↓
Persistence
```

The structure itself is not wrong. The problem arises when business-significant steps, branches, multi-record changes, external I/O, and transaction boundaries are distributed across these layers, forcing a reader to follow many places to understand the whole operation.

HUMQ does not aim to reduce layers for their own sake. It reduces unnecessary indirection when tracing a process.

## Where Complexity Lives

```text
HUMQ

Complex business flow
          ↓
       Usecase
   ┌──────┼────────┐
Module  Query  External Client
  ↓       ↓          ↓
1 table  Cross-table Side Effect
```

Lower-level parts remain narrow and predictable. Usecase expresses when, in what order, and under which conditions they are used.

## Relationship to Other Designs

### Layered Architecture

In layered architecture, the team decides the granularity and responsibilities of Controller, Service, and Repository. That freedom is useful, but Service may come to hold business flow, shared helpers, and data-access coordination at once, while Repository may start making business decisions.

HUMQ removes Service as a generic placement category. It fixes business flow in Usecase, one-table operations in Module, and cross-table reads in Query.

### Clean Architecture

Clean architecture emphasizes dependency direction and independence of business rules from external details. HUMQ prioritizes traceable table-sized operation targets and business flows over persistence independence.

The two can be combined in part, but `Module = exactly one table` is an intentional coupling to database structure and therefore sits in tension with designs that make persistence independence the highest priority.

### DDD

DDD expresses business concepts and invariants through a Domain Model and Aggregate boundaries. HUMQ does not use Domain concept boundaries as Module boundaries. It fixes Module at one table and expresses business wholes and consistency spanning tables in Usecase.

HUMQ is not a simplified Aggregate. It makes a different tradeoff against boundary-design cost and the reduced visibility of cross-boundary operations.

## HUMQ Mapping

| Common layer or concept | HUMQ treatment |
| --- | --- |
| Controller | Handler, limited to input/output |
| Application Service / Interactor | A primary business flow generally maps to Usecase |
| Generic Service | No direct mapping; flow belongs in Usecase and local operations in Module |
| Domain / Aggregate | No direct mapping; reusable pure decisions may be extracted as Policies |
| Repository | Not required; an internal implementation detail of Module when needed |
| Read Model / Report | Read-only Query |
| Mailer / Payment Gateway / API Client | An external client called visibly from Usecase, not a Module |

## Comparison Table

| Viewpoint | Common layered structure | Clean architecture | DDD | HUMQ |
| --- | --- | --- | --- | --- |
| Business flow | Depends on Service design | May live in Usecase / Interactor | Expressed in application or Domain code | Explicit as one Usecase primary flow |
| Lower-level boundary | Chosen by the team | Chosen through dependency rules and abstractions | Chosen through Domain / Aggregate design | Mechanically fixed at exactly one table |
| Cross-table reads | Often placed in Repository or Service | Requires a Gateway, Read Model, or similar design | Requires Repository, Read Model, or similar design | Fixed in read-only Query code |
| Consistency | Often coordinated by Service | Depends on Usecase and Entity design | Protected inside Aggregates and coordinated by application code | Cross-table consistency is explicit in Usecase |
| Persistence coupling | Depends on implementation | Generally pushed to external details | May be isolated from Domain | Module intentionally couples to table structure |
| Indirection | Depends on layers and Service granularity | Ports and Adapters may add it | Model design may add it | Minimized along important flows |
| Main design cost | Service / Repository granularity | Boundaries, Ports, and dependency direction | Domain Model and Aggregates | Usecase readability and explicit consistency |

This table compares defaults and tradeoffs, not superiority. Any design can become explicit or opaque depending on its implementation and operation.

## Adoption and Tradeoffs

HUMQ primarily targets applications that use a relational database as their main persistence model and handle multi-table state changes and cross-table reads.

### What HUMQ Gains

- Predictable code placement that remains stable as people change
- Traceable business flows and transaction boundaries
- Modules confined to one table with explicit operation targets
- Queries that separate cross-table reads from writes

### What HUMQ Accepts

- Usecase grows when the business grows complex.
- Query grows when screens, searches, and reports grow complex.
- Module is intentionally coupled to table structure.
- Cross-table consistency is kept explicit in Usecase and tests rather than enclosed in Domain code.

HUMQ does not assume complexity can be eliminated.<br>
It chooses to absorb complexity where it remains traceable instead of distributing it across the structure.

### Where HUMQ Works Well

- Business flows, exceptions, and temporary measures are numerous.
- Multi-table write procedures need to remain explicit.
- Cross-table lists, searches, and reports are common.
- The team wants fewer recurring decisions about code placement.
- Long-term operation benefits from following fewer files during a change.
- Relational tables are stable, primary persistence boundaries.

### Where Another Design May Fit Better

HUMQ does not itself provide a persistence-independent Domain Model, Event Sourcing state management,<br>
or a runtime for long-running distributed workflows.

- Tables are not the primary persistence boundary, as in Event Sourcing.
- Complex invariants need to be strongly enclosed in a rich Domain Model or Aggregate.
- Strict Domain independence from persistence is the highest priority.
- Long-running workflows across external systems dominate and a Saga or workflow engine is itself the primary abstraction.

Using payments or external services does not by itself put a system outside HUMQ's scope.<br>
Ordinary integrations can complement HUMQ with Outbox, idempotency, Sagas, or compensating actions.

### What HUMQ Requires

Mechanical boundaries do not automatically guarantee business correctness. HUMQ requires implementers to:

- Make consistency, state transitions, and external I/O explicit in Usecase.
- Preserve traceability as one business intent instead of optimizing for short files.
- Avoid hiding business flow through shared helper extraction.
- Keep Module closed around exactly one table without hidden writes to another table.
- Keep Query read-only.
- Keep Handler thin.

HUMQ does not claim to be simpler than every alternative.

> Keep lower-level parts simple and place important complexity where it remains visible from Usecase.

HUMQ does not eliminate complexity. It puts complexity where it can be traced.

---

Previous: [Consistency and Transactions](04-consistency-and-transactions.md) | Next: [FastAPI Example](06-fastapi-example.md)
