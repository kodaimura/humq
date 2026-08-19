# Comparison with Existing Architectures and Design Patterns

HUMQ provides clearer code placement than MVC + Service<br>
and uses fewer design concepts than aggregate-centered DDD, making it a lightweight application architecture.<br>
In exchange, it does not structurally protect cross-table consistency inside a Domain Model;<br>
each Usecase is responsible for implementing it.

This page explains why HUMQ chooses that tradeoff,<br>
which applications fit it, and when another design should take the lead.

MVC, layered architecture, Clean Architecture, DDD, Transaction Script, Table Module, and CQRS<br>
are not concepts at the same level.<br>
Some can be used together, while other combinations have conflicting design priorities.<br>
This page compares their typical defaults for placement, traceability,<br>
consistency guarantees, and coupling to relational tables.

## Comparison Axes

Each design is compared from four perspectives:

- What does the architecture protect?
- Where do business flow and complexity belong?
- How much of the responsibility boundary remains a human design decision?
- How strongly is the design coupled to relational table structure?

The sections below focus on the perspectives where each architecture differs most.

## Positioning HUMQ

| Design | Primary optimization target | Basis of boundaries | Main tradeoff |
| --- | --- | --- | --- |
| MVC + Service | Separation of concerns and<br>structural flexibility | Roles such as Controller, Model, and Service<br>defined by the team | The same behavior can have<br>several reasonable locations |
| HUMQ | Predictable placement and<br>traceable business flows | Usecase flow, a one-table Module by default,<br>and cross-table Query | Cross-table consistency depends<br>on Usecase implementation |
| Aggregate-centered DDD | Protection of invariants and business<br>concepts through a Domain Model | Bounded Contexts, Aggregates,<br>Entities, Value Objects, and related concepts | Continuous modeling and shared<br>design judgment about boundaries |

HUMQ is neither a midpoint between MVC + Service and DDD<br>
nor a higher or lower stage of architectural maturity. It chooses a different default for what the structure protects.<br>
Moving from HUMQ to aggregate-centered DDD is not an architectural upgrade;<br>
it changes the optimization target from placement and traceability to structural protection of invariants.

HUMQ does not require Aggregate, Value Object, Domain Service, Repository,<br>
or Bounded Context. It permits coupling to ORM models and relational tables<br>
and prioritizes predictable placement and traceable business flows over domain modeling.

## The Same Business Flow in Each Design

Suppose confirming an order must finalize its line-item amounts, reflect their sum in the order total,<br>
and mark the order as confirmed.

- **MVC + Service** leaves the team to decide whether Controller, Model, or Service coordinates the processing.<br>
  This provides structural flexibility, but placement decisions may differ across developers.
- **HUMQ** makes `ConfirmOrderUsecase` call `OrderItemModule` and `OrderModule` explicitly.<br>
  The code to inspect is predictable,<br>
  but omitting a required Module call can omit a consistency rule.
- **Aggregate-centered DDD** places the invariants of the order and its line items in an Order Aggregate,<br>
  making invalid updates harder to create. In exchange, the Aggregate boundary must be designed,<br>
  and the model and implementation must be kept aligned over time.
- **Transaction Script** expresses order confirmation as one procedure that can be read from top to bottom.<br>
  It does not by itself determine where data access or cross-table reads belong.

The difference is not whether each design can implement the business operation.<br>
MVC + Service prioritizes structural flexibility, Transaction Script prioritizes procedural traceability,<br>
HUMQ combines traceability with consistent placement, and aggregate-centered DDD prioritizes structural protection of invariants.

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

Because HUMQ uses database tables as Module boundaries,<br>
schema changes also affect Module boundaries and naming. The major difference is that HUMQ prioritizes<br>
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
It delegates cross-table consistency to Usecase implementation and keeps it traceable from the business flow.<br>
HUMQ is not simplified DDD;<br>
it is a different choice that prioritizes predictable placement over protection through modeling.

Designing every domain around rich Aggregates from the beginning adds the continuing cost of modeling<br>
and maintaining boundaries even where invariants are simple. HUMQ starts with a lightweight structure.<br>
Only when multiple Usecases must preserve the same invariant should centralizing it<br>
in an Operation be considered as an exception.<br>
When complex shared invariants become central to the domain, aggregate-centered design is a better fit.

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

The comparisons above can now be applied to an actual adoption decision.

HUMQ primarily targets applications that use a relational database as their main persistence model<br>
and handle multi-table state changes along with cross-table reads for screens, searches, and reports.

### Where HUMQ Fits

- Business state is managed in stable relational tables.
- Multi-table operations keep accumulating branches, special cases, and temporary measures.
- Lists, searches, and reports frequently read across multiple tables.
- Multiple people maintain the system over time and want fewer decisions about code placement.
- Most business rules are tied to individual Usecases rather than sharing the same invariant across many operations.

With HUMQ, business flows, state transitions, and transaction boundaries are easier to trace from Usecase,<br>
while the impact of a one-table operation and the code to inspect during change become more predictable.<br>
As a secondary benefit, the context needed to understand one change stays bounded,<br>
which also helps AI agents identify what to inspect.

### Where HUMQ Adds Little Value

- Most operations are simple CRUD against one table.
- Multi-table business flows and reads are rare.
- A small or short-lived project has no recurring problem with code placement.

In these cases, HUMQ adds little predictability,<br>
and a simpler MVC or layered architecture may be sufficient.

### Tradeoffs HUMQ Accepts

- Usecase tends to grow because it concentrates business flow and consistency decisions.
- Because Module is coupled to table structure, schema changes affect boundaries and naming.
- A normalized table structure appears in Usecase as multiple Module calls.
- Cross-table consistency depends on Usecase implementation, so an omitted rule can produce inconsistent data.

HUMQ does not assume that consistency is unimportant.<br>
It assumes that, in most real systems, only a minority of domains<br>
require the structural protection provided by an Aggregate.<br>
Under that assumption, HUMQ deliberately accepts the risk of omitted consistency enforcement<br>
as the tradeoff for lightweight and explicit placement rules.

Database constraints and tests reduce that risk, but do not make consistency enforcement complete by construction.<br>
When that risk is unacceptable, choose another design that structurally protects the requirement.

Schema complexity becoming visible in Usecase is an intentional HUMQ tradeoff.<br>
HUMQ prioritizes a mechanical answer to “where is the code that changes this table?”<br>
over hiding the table structure behind abstract Domain boundaries.

### When to Choose Another Design

| Central requirement | Better-fit design |
| --- | --- |
| Multiple tables in one domain must always update as a unit,<br>and many operations share complex invariants | Aggregate-centered DDD |
| Domain must remain strictly independent of the relational database<br>or persistence mechanism | Clean Architecture or another design<br>that keeps Domain on the inside |
| Event history is the primary persistence model | Event Sourcing |
| Long-running processing spans external systems,<br>with retry and compensation management at the center | Workflow Engine or similar |

### What HUMQ Requires

- Keep each Usecase focused on one business purpose that can be explained from top to bottom.
- Make business flow, cross-table consistency, transaction boundaries, and external I/O explicit in Usecase.
- Keep Module within one-table reads and writes by default, Query read-only, and Handler limited to external input and output.
- Enforce database-expressible constraints in the database, and test the consistency and `rollback` behavior owned by Usecase.
- Use Operation only when the implementation of the same invariant cannot be allowed to diverge.

HUMQ's mechanical boundaries do not automatically guarantee business correctness.<br>
They fix where correctness is implemented and verified.

---

Previous: [Handling Consistency](04-consistency-and-transactions.md) | Next: [FastAPI Example](06-fastapi-example.md)
