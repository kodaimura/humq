# Design Principles

HUMQ is not a design for keeping everything clean. It is a design for deciding where mess is allowed and protecting the places that must not distort.

## Principle 1: Break, but Do Not Distort

HUMQ distinguishes breaking from distorting.

| Viewpoint | Breaking | Distorting |
| --- | --- | --- |
| Cause | Implementation mistakes, requirement changes, temporary gaps | Responsibility leaks, mixed layers, conceptual collapse |
| Repairability | Fixable | Hard to fix |
| HUMQ treatment | Accepted as local chaos | Avoided as structural damage |
| Main repair area | Usecase | The whole structure |

Business requirements may become complex inside Usecase. That means Usecase is absorbing real-world chaos.

By contrast, Handler starting database operations, Module calling another Module, Module owning multiple tables, Query starting writes, or business flow leaking into Module are distortions. Once distortion enters the system, people begin placing code by personal preference, and order is lost.

## Principle 2: Design Freedom

If freedom is fully forbidden, deviation leaks somewhere else. HUMQ designs Usecase as the place for freedom.

Usecase absorbs:

- Business exceptions
- Temporary requirements
- Complex branching
- Combinations of multiple Modules
- Explicit control of consistency

Chaos is allowed, but not everything goes. A Usecase may be complex, but its complexity must remain explainable as one business intent.

Usecase freedom does not justify bundling unrelated business operations into one giant Usecase. When the same business procedure is duplicated across multiple Usecases, identify the responsibility and extract it appropriately. But do not push business logic into Module only to make Usecase look clean.

Because this freedom exists, Handler, Module, and Query can remain relatively pure. This is how HUMQ connects freedom to "Break, but do not distort": Usecase may change and be repaired, while the structural boundaries stay stable.

## Principle 3: Treat Module as the Laws of the World

Module is the stable local table order inside the application.

A Module is closed around exactly one table. It does not represent an Aggregate or a broad business subject. It does not know other Modules. It does not know business flows. It provides table-specific operations as a part called from Usecase.

If exceptional business concerns are placed in Module, Module becomes harder to reuse. It also starts behaving unnaturally from the perspective of Usecases that do not need that exception.

If a business operation spans three tables, HUMQ prefers three Modules coordinated by one Usecase over one large Module.

## Principle 4: Keep Query as Observation

Single-table reads belong in Module. Reads that span multiple tables should not be forced into Module. Module order is table-based, while cross-table reads have a different nature.

Query is the layer that accepts this cross-cutting nature. But it does not write. It only observes.

## Principle 5: Do Not Hide Consistency

HUMQ makes consistency explicit in Usecase.

Designs like DDD Aggregate, where consistency is enclosed internally, are theoretically powerful. In practice, however, business operations often live at the Usecase level, and enclosing consistency too deeply can reduce reusability.

In HUMQ, Usecase shows which Modules are combined to maintain consistency. This trades automatic safety for structural visibility and changeability.

## Principle 6: Prioritize Responsibility Boundaries

The most important thing in HUMQ is that each layer's responsibility stays stable.

Crossing a boundary because it is convenient may be fast in the short term, but it eventually destroys the criteria for where code belongs.

When unsure, decide by asking these questions:

| Question | Place |
| --- | --- |
| Is it about input/output from the outside world? | Handler |
| Is it about business flow or consistency? | Usecase |
| Is it an operation closed around exactly one table? | Module |
| Is it a read that spans multiple tables? | Query |

---

Previous: [Layer Rules](02-layer-rules.md) | Next: [Consistency and Transactions](04-consistency-and-transactions.md)
