# Design Principles

Design principles define what HUMQ prioritizes when abstraction, reuse, and decomposition require judgment.

## Principle 1: Keep Essential Complexity Visible

Business-significant operation order, branches, state transitions, multi-table writes,<br>
external I/O, and transaction boundaries remain visible from Usecase.

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
Local calculations may be extracted into helpers, but multi-Module operations and state changes must remain visible.

This does not allow unrelated business operations, shared mutable state,<br>
or one-table implementation details that belong in Module to accumulate in Usecase.<br>
Split Usecase when the result represents an independently explainable business operation, not because it exceeds a line count.

## Principle 3: Prefer Traceability to Reuse

A shared business decision may be reused as a pure Policy or function.<br>
Its invocation and the branch resulting from that decision remain visible in Usecase.

When exactly the same business flow appears in multiple Usecases<br>
and can be explained as an independent business operation, it may be extracted into a shared Usecase.<br>
The call and any branch based on its result remain visible in the caller.<br>
When called by another Usecase, the shared Usecase participates in the caller's transaction and does not `commit` itself.

Do not extract code merely because some lines look similar<br>
or hide multi-Module operations behind an ambiguously named shared operation.<br>
HUMQ does not avoid reuse itself. It avoids reuse that makes the location of a business flow unclear.

## Principle 4: Prefer Mechanical Boundaries to Judgment-Dependent Boundaries

Decisions such as "these things are related," "this is used only by this screen," or "this belongs near the Domain"<br>
may all be reasonable, but their conclusions change with the person and situation.

HUMQ fixes external input and output in Handler, business flows in Usecase,<br>
one-table operations in Module, and cross-table reads in Query.<br>
It prioritizes consistent placement across developers over making every individual boundary locally elegant.

## Principle 5: A Broken System Can Be Fixed. A Distorted Structure Cannot

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
