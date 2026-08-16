# Design Principles

HUMQ does not try to eliminate essential business complexity. It keeps lower-level parts simple so that complexity remains traceable through their composition.

## Principle 1: Keep Essential Complexity Visible

Business-significant order, branches, state transitions, multi-table writes, external I/O, and transaction boundaries remain visible from Usecase.

ORM mechanics, SQL construction, transport protocols, and local data conversion may be hidden as implementation details. HUMQ does not reject encapsulation. It rejects encapsulation used merely to make a business flow look short.

```text
Implementation details may be hidden.
Business flows should not be hidden.
```

## Principle 2: Prefer Local Complexity to Hidden Complexity

The important measure is not the number of lines. It is the number of places someone must follow to understand one business operation.

A 300-line Usecase understandable from top to bottom may be preferable to 100 lines scattered across several Services. This does not approve giant functions unconditionally.

**Acceptable Usecase**

- Represents one explainable business intent
- Makes important steps, branches, I/O, and transaction boundaries traceable
- Gives each Module, Query, and external client a clear role

**Usecase to reconsider**

- Mixes unrelated business intents
- Obscures order through deep nesting or shared mutable state
- Contains local one-table work that belongs in Module
- Hides the business flow inside extracted helpers

## Principle 3: Give Module a Mechanical Boundary

A Module is a small, predictable operation unit that owns exactly one table. It is not an Aggregate or broad Domain concept and knows neither other Modules nor business flows.

If "related tables" define the boundary, the team must repeatedly decide how far relatedness extends and where exceptions belong. HUMQ removes that design decision by choosing one table as a mechanical, unambiguous boundary.

```text
UserModule       -> users
OrderModule      -> orders
OrderItemModule  -> order_items
```

One business concept may span multiple tables; Usecase expresses that business whole. Avoid ORM cascades or hooks that silently modify another table because they break this predictability.

## Principle 4: Compose Business Flows Explicitly

Operations spanning multiple tables appear in Usecase as calls to their Modules.

```text
CreateOrderUsecase
  OrderModule.create()
  OrderItemModule.createMany()
  InventoryModule.decrease()
  AuditLogModule.create()
```

A Usecase shows one explainable business intent and its primary flow in one file. Local calculations and pure Policies may be extracted, but coordination of multiple Modules, external I/O, and state transitions must not disappear behind a helper Service.

When several Usecases share a business decision, the decision may be reused while its invocation and the resulting branch remain visible in each Usecase. Reuse and visibility can coexist.

## Principle 5: Keep Query as Cross-Table Observation

Single-table reads belong in Module. Reads that span multiple tables belong in read-only Query code.

Joins and aggregations shaped by screens, searches, or reports do not need to be forced into write-oriented Module boundaries. Query still does not write or own transaction boundaries.

## Principle 6: Do Not Hide Consistency or Side Effects

Usecase shows which Modules are combined, in what order, and which range succeeds or fails as a unit. It also reveals where external side effects occur and what happens when they fail.

DDD Aggregates are a strong way to protect invariants within their boundaries. HUMQ does not reject them; it chooses table-sized parts and explicit Usecase-level consistency as its default. This deliberately favors traceability over automatic protection.

## Principle 7: A Broken System Can Be Fixed. A Distorted Structure Cannot

Here, "breakage" means ordinary bugs and implementation mistakes that cause the system to stop behaving as expected. It does not mean tolerating bugs or leaving defects unfixed. Correcting the implementation that caused the problem restores the expected behavior.

Here, "distortion" means that structural responsibility boundaries change with the person or situation. Individual features may still work, but neither the correct placement nor the repair scope remains clear. HUMQ does not guarantee the absence of bugs; it prioritizes preventing bug fixes and new special cases from distorting the entire structure.

Handler performing database operations, Module calling another Module, Module modifying multiple tables, Query writing, or business flow disappearing into a helper Service are structural distortions. Once distortion enters, code placement varies by developer and change impact becomes difficult to trace.

When unsure, decide by asking these questions:

| Question | Place |
| --- | --- |
| Is it about input/output from the outside world? | Handler |
| Is it about business flow, consistency, state transitions, or external I/O? | Usecase |
| Is it an operation closed around exactly one table? | Module |
| Is it a read that spans multiple tables? | Query |

> Keep the parts simple. Keep the important chaos visible.

---

Previous: [Layer Rules](02-layer-rules.md) | Next: [Consistency and Transactions](04-consistency-and-transactions.md)
