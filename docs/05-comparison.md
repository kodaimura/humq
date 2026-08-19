# Comparison with Existing Architectures and Design Patterns

The difference between HUMQ and existing designs is not a matter of superiority, but of<br>
what each design protects, which decisions remain human judgments, and where complexity is absorbed.

MVC, layered architecture, Clean Architecture, DDD, Transaction Script, Table Module, and CQRS<br>
are not concepts at the same level.<br>
Some can be used together, while other combinations have conflicting design priorities.<br>
This document compares their typical defaults with the tradeoffs HUMQ chooses.

## Comparison Axes

The comparison focuses on four questions:

- What does the architecture protect?
- Where do business flow and complexity belong?
- How much of the responsibility boundary remains a human design decision?
- How strongly is the design coupled to relational table structure?

HUMQ prioritizes stable code placement across developers and traceable business flows.<br>
It does not automatically guarantee business correctness or data consistency.<br>
Instead, it fixes where those concerns must be implemented and verified.

## Positioning HUMQ

HUMQ provides clearer code placement than MVC + Service<br>
and uses fewer design concepts than aggregate-centered DDD, making it a lightweight application architecture.<br>
In exchange for that lighter structure, each Usecase explicitly protects consistency across multiple tables.

This choice assumes that protecting every domain with an Aggregate is often overengineering,<br>
and that only a limited part of a system needs invariants enforced so callers cannot omit them.<br>
HUMQ is neither a midpoint between MVC + Service and DDD nor a higher or lower stage of architectural maturity.<br>
It chooses a different default for what the structure itself must protect.

| Design | Primary optimization target | Basis of boundaries | Main tradeoff |
| --- | --- | --- | --- |
| MVC + Service | Separation of concerns and structural flexibility | Roles such as Controller, Model, and Service defined by the team | The same behavior can have several reasonable locations |
| HUMQ | Predictable placement and traceable business flows | Usecase flow, a one-table Module by default, and cross-table Query | Intentional RDB coupling and explicit invariant handling in Usecase |
| Domain-centered DDD | Protection of invariants and business concepts through a Domain Model | Bounded Contexts, Aggregates, Entities, and Domain Models | Requires continuous modeling and shared design judgment about boundaries |

Moving from HUMQ to DDD is not an architectural upgrade.<br>
It is a change in optimization target when protecting shared invariants in a Domain Model matters more than placement and traceability.

HUMQ does not require Aggregate, Value Object, Domain Service, Repository,<br>
or Bounded Context. It permits coupling to ORM models and relational tables<br>
and prioritizes predictable placement and traceable business flows over domain modeling.

## MVC + Service / Layered Architecture

MVC + Service and layered architecture separate input/output, business processing, and data access.<br>
They are widely understood, flexible designs whose layer granularity can be adjusted to a system's scale and characteristics.

However, the team chooses Service granularity, Model behavior, Repository responsibilities,<br>
and transaction ownership.<br>
Several reasonable answers may exist for whether the same behavior belongs in Controller, Model, or Service.

HUMQ removes the generic Service role and fixes caller input/output in Handler, business flow in Usecase,<br>
one-table reads and writes in Module by default, and cross-table reads in Query.<br>
It gives up some flexibility to keep placement stable across developers.

HUMQ's value is not inventing new concepts.<br>
It selects the minimum needed from existing implementation patterns and reduces placement decisions.

## Clean Architecture

Clean Architecture controls dependency direction and keeps business rules independent<br>
of external details such as frameworks and databases.<br>
Its strength is protecting important policy while making technical mechanisms replaceable.

It still leaves decisions such as whether an individual rule belongs in Entity or Usecase<br>
and how large a Usecase or Port should be.<br>
Correct dependencies do not make code placement unique.

HUMQ additionally fixes operation targets and code placement.<br>
Clean Architecture values keeping Domain and Usecase independent of ORMs, Web frameworks,<br>
databases, and other external details. HUMQ does not require that independence<br>
and allows Usecase to handle Sessions and ORM models.

HUMQ intentionally couples Module to relational tables. The major difference is that HUMQ prioritizes<br>
predictable placement and traceable business flows over replaceability of technical mechanisms.

## Aggregate-Centered DDD

Aggregate-centered DDD concentrates business concepts, invariants, and state transitions<br>
in a Domain Model, making invalid states harder to create.<br>
Its strength is allowing the model itself to protect complex consistency rules.

Aggregate, Entity, Value Object, and Domain Service boundaries are determined<br>
through continuing domain modeling.<br>
Cross-Aggregate operations, cross-cutting UI requirements, and temporary measures<br>
still require shared team judgment about placement.

HUMQ uses relational tables rather than Domain concepts as Module boundaries.<br>
Cross-table consistency remains explicit in Usecase, making business flow easier to trace.<br>
However, if a Usecase fails to call a required Module,<br>
a consistency rule may be omitted.<br>
HUMQ is not simplified DDD;<br>
it is a different choice that prioritizes predictable placement over protection through modeling.

Designing every domain around Aggregates from the beginning adds the continuing cost of modeling<br>
and maintaining boundaries even where invariants are simple. HUMQ starts with a lightweight structure.<br>
Only when multiple Usecases must preserve the same invariant may its implementation be centralized,<br>
as an exception, in an Operation. This keeps the implementation in one place.<br>
However, Operation does not structurally prevent callers from omitting it. When Operations continue to grow,<br>
or when an omitted call itself is unacceptable, aggregate-centered design is a better fit.

## Transaction Script

Transaction Script expresses one business operation as one procedure.<br>
Its top-to-bottom business flow makes it the pattern closest to HUMQ's Usecase.

Transaction Script alone does not standardize data-access boundaries, where cross-table reads belong,<br>
or how transaction boundaries are expressed.<br>
If each Script accesses the database directly or begins choosing different Gateways or Repositories,<br>
lower-level responsibilities and consistency handling can diverge.

HUMQ adds mechanical boundaries to Transaction Script: `1 Usecase = 1 Primary Flow`,<br>
`Module = one-table reads and writes by default`, and `Query = cross-table reads`,<br>
with transaction boundaries made explicit in Usecase.<br>
It keeps the procedural nature of Usecase visible while making lower-level parts simple and predictable.

## Table Module / Table Data Gateway

Table Module organizes business logic for one table or view in a single component,<br>
while Table Data Gateway centralizes data access for one table.<br>
Both resemble HUMQ's Module in treating a relational table as an explicit boundary.

Neither pattern by itself determines where multi-table business flows,<br>
cross-table reads, external I/O, and transactions are composed.

By default, a HUMQ Module reads and writes one table and provides that table's standard operations.<br>
Only when writing its target table requires it may Module read another table as an exception.<br>
It does not write other tables, own a business flow, or manage transaction boundaries.<br>
Beyond table-oriented parts, HUMQ requires their composition to remain in Usecase<br>
and separates cross-table reads into Query.

## CQRS

CQRS separates the model used to update information from the model used to read it.<br>
The two models may share a data store or use different stores and projections.

HUMQ Query resembles the Query or Read side of CQRS because it builds Read Models<br>
shaped for a purpose. It normally reads across multiple tables. As an exception, a complex<br>
or purpose-specific read may belong in Query even when it uses only one table.<br>
However, HUMQ does not require independent Command and Read Models for every operation.<br>
Separate data stores, asynchronous projection updates, and eventual consistency are not HUMQ prerequisites.

CQRS separates reads and writes but does not determine<br>
where business flow and data access belong on the Command side.<br>
HUMQ adds fixed placement rules for Handler, Usecase, Module, and Query.

## Adoption and Tradeoffs

HUMQ primarily targets applications that use a relational database as their main persistence model<br>
and handle multi-table state changes along with cross-table reads for screens, searches, and reports.

### Scale and Complexity Axes

HUMQ's applicability is not determined by table count or lines of code alone.

| Scale or complexity axis | Fit with HUMQ |
| --- | --- |
| Many tables | Applicable |
| Many APIs, Usecases, screens, or reports | Applicable |
| Long individual flows with many branches and exceptions | Good fit because they remain traceable from Usecase |
| Many complex cross-table reads | Good fit because they can be separated into Query |
| Multiple Usecases share the same invariant and its implementation cannot diverge | May be centralized in an Operation as an exception |
| Operations hold primary flows or many shared invariants | Evaluate the adoption limit |
| Interactions among complex Domain Models are central | Consider DDD or another domain-centered design |

Handling many tables or a large codebase is a different capability<br>
from safely reusing complex shared invariants.

### Benefits of HUMQ

- Fewer team decisions and agreements about where code belongs.
- Easier tracing of business flows, state transitions, and transaction boundaries from Usecase during a change.
- More predictable impact and code scope for one-table operations.
- Prevention of complex lists and reports from becoming mixed into write processing.
- Business flows remain local to Usecase, bounding the context needed for one change<br>
  and helping both people and AI agents identify what to inspect.

### Drawbacks of HUMQ

- Usecase tends to grow because it concentrates business flow and consistency decisions.
- Because Module is coupled to table structure, schema changes affect boundaries and naming.
- A normalized table structure appears in Usecase as multiple Module calls.
- Cross-table consistency depends on Usecase implementation, so an omitted rule can produce inconsistent data.

Schema complexity becoming visible in Usecase is an intentional HUMQ tradeoff.<br>
HUMQ prioritizes a mechanical answer to “where is the code that changes this table?”<br>
over hiding the table structure behind abstract Domain boundaries.

### Where HUMQ Provides the Most Value

- Business state is managed in stable relational tables, and each state transition can be expressed as a Usecase.
- Multi-table operations keep accumulating branches, special cases, and temporary measures.
- Lists, searches, reports, and administration views frequently read across multiple tables.
- Multiple people maintain the system over time and want fewer decisions about code placement.
- Most business rules are tied to individual Usecases.
- The system does not need the cost of a rich Domain Model or persistence abstraction.

Typical examples include order processing, inventory, contracts, billing, and approval systems<br>
whose relational business flows change continuously.

### Where HUMQ Adds Little Value

- Most operations are simple CRUD against one table.
- Multi-table business flows and reads are rare.
- A small or short-lived project has no recurring problem with code placement.

In these cases, HUMQ adds little predictability,<br>
and a simpler MVC or layered architecture may be sufficient.

### Where Another Design Should Lead

- One domain state spans many tables that must be updated as a single unit.
- A missed update directly causes inconsistency in financial, inventory, or contract state,<br>
  so consistency cannot depend only on each Usecase implementation.
- The same business logic must be reused across systems with different database schemas or persistence mechanisms.
- Processing spans multiple external systems over long periods, with retry and compensation state as central concerns.

### Adoption Limits and Evolution

Sharing the same database-backed processing across multiple Usecases is not itself a HUMQ adoption limit.<br>
Reconsider the affected domain when shared processing grows beyond a supporting role for Usecase.<br>
See [Adoption Limits and Evolution](07-adoption-limits-and-evolution.md)<br>
for detailed criteria and migration options.

### What HUMQ Requires

Mechanical boundaries do not automatically guarantee business correctness. HUMQ requires implementers to:

- Keep each Usecase to one business purpose, such as “confirm an order,” that can be explained from top to bottom.
- Make consistency, state transitions, external I/O, and transaction boundaries explicit in Usecase.
- By default, keep each Module closed around reads and writes for one table.
- Keep Query read-only.
- Limit Handler to input and output for the caller.
- Verify the consistency rules owned by Usecase through tests.

HUMQ does not make the business simple.<br>
It is a choice to keep real-world complexity where the team can continue to trace it.

---

Previous: [Handling Consistency](04-consistency-and-transactions.md) | Next: [FastAPI Example](06-fastapi-example.md)
