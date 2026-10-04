# Design Principles

Design principles define what HUMQ prioritizes when abstraction, reuse, and decomposition require judgment.

## Principle 1: Keep Essential Complexity Visible

The primary business flow remains visible from Usecase: its purpose, major operation order,<br>
branches based on results, transaction boundaries, external I/O, and post-failure policy.<br>
Detailed decisions and Module operations may live in clearly named internal processing,<br>
where their validation, locking, and change targets can be inspected.

ORM operations, SQL construction, communication mechanics, and local data conversion<br>
may be hidden as implementation details inside Modules or external clients.<br>
HUMQ does not avoid abstraction itself. It avoids abstraction that hides important steps merely to make a business flow look short.

## Principle 2: Prefer Local Complexity to Distributed Simplicity

Optimize not for the length of one file, but for the number of places required to understand one business operation.

Splitting logic across several Services or helpers to shorten Usecase<br>
can make each file simple while making the overall business flow harder to follow.<br>
A long Usecase is acceptable when one business operation can be explained by reading it from top to bottom.

Allowing Usecase to grow does not mean giving up readability.<br>
Organize operations into meaningful business stages and use clear names, early returns, and shallow nesting<br>
so that the primary flow remains readable from top to bottom.<br>
Independently meaningful business processing may be extracted, including processing that combines Modules,<br>
as long as Usecase still shows why it is called and what happens with its result.

This does not allow unrelated business operations, shared mutable state,<br>
or persistence implementation details that belong in Module to accumulate in Usecase.<br>
Split Usecase by business intent, not by line count.<br>
Being long, having many branches, or calling many Modules is not sufficient reason by itself.

Consider a separate Usecase when the operation:

- Can run independently and has its own user action or trigger.
- Has an independent authorization decision.
- Can be described as an independent success, failure, or transaction boundary.
- Forms an independent retry or compensation unit.
- Is more naturally described as “and then, as a separate operation” in the original flow.

Independent business operations, internal decisions or consistency processing, and input-format conversion<br>
may be separated when the primary flow remains traceable. Avoiding decomposition is not the goal;<br>
keeping the primary flow visible is. HUMQ allows traceable local complexity,<br>
not an unstructured giant function.

## Principle 3: Prefer Business Meaning and Traceability to Reuse

An independently meaningful business decision or process may be extracted even when only one Usecase calls it.<br>
Pure calculations, database-informed decisions, and multi-Module consistency processing follow this same criterion.<br>
Extraction is optional even when the processing has independent business meaning or multiple callers.<br>
When extracted, the call and primary branch based on its result remain in Usecase.

Two occurrences alone do not require sharing, and line count alone does not require extraction.<br>
HUMQ favors a clear business reason to change and a traceable flow over reuse for its own sake.

## Principle 4: Use Clear Boundaries to Reduce Placement Decisions

Decisions such as "these things are related," "this is used only by this screen," or "this belongs near the Domain"<br>
may all be reasonable, but their conclusions change with the person and situation.

HUMQ fixes input and output for the caller in Handler, business flows in Usecase,<br>
one-table reads and writes in Module by default, and cross-table reads in Query.<br>
These rules reduce placement decisions and make code easier to find; they do not eliminate judgment.<br>
Usecase granularity, the Module–Query boundary, and whether to extract internal processing still require design decisions.<br>
Teams also choose where cross-domain internal processing lives; an owning domain is a useful default,<br>
but a cohesive shared package can serve the same Usecase responsibility.

## Principle 5: Bugs Can Be Fixed. Distortion Eventually Becomes Unmanageable.

Here, "breakage" means a defect causing the system to stop behaving as expected.<br>
"Distortion" means responsibility boundaries change with the person or situation,<br>
until code placement can no longer be determined from the structure.

HUMQ does not tolerate or leave defects unfixed.<br>
It treats breaking responsibility boundaries while fixing a defect or adding a special case—<br>
and thereby increasing the places that future changes must follow—as a larger design failure.

Handler accessing the database, Module calling another Module, Module updating multiple tables,<br>
Query writing, or a business flow disappearing into a helper Service are structural distortions.<br>
Even when Usecase grows, the chaos that belongs there does not spread into other layers.

> Keep the parts simple. Keep the important chaos visible.

---

Previous: [Layer Rules](02-layer-rules.md) | Next: [Handling Consistency](04-consistency-and-transactions.md)
