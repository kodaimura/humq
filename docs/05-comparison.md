# Comparison with Existing Architectures

> Japanese: [05-comparison.ja.md](05-comparison.ja.md)

HUMQ does not reject MVC, layered architecture, clean architecture, or DDD. It is a design that treats responsibility boundaries more explicitly where those architectures often become ambiguous in practice.

## Common Problems in Existing Designs

### Service and Repository Granularity Drifts

Whether business logic belongs in Service, and how much intelligence Repository should have, often varies by project or developer.

As a result, Service grows too large, or Repository begins to hold business decisions.

### Controller Becomes Too Large

When lower-layer responsibilities are vague, logic tends to escape into Controller.

If validation, branching, database operations, and response shaping are mixed together, Controller stops being an entry point and becomes the body of the business process.

In HUMQ, this layer is called Handler, and its responsibility is limited to input and output.

### Domain Gets Polluted by Real-World Exceptions

In DDD, Domain and Aggregate hold strong order. But real business systems include exceptions, temporary measures, organizational constraints, and UI-driven needs.

If those are placed in Domain, it becomes difficult to preserve purity over time.

In HUMQ, real-world exceptions are absorbed by Usecase, while Module's order is protected.

## HUMQ Mapping

| Common layer | HUMQ treatment |
| --- | --- |
| Controller | Handler, limited to input/output |
| Service | Split into Usecase and Module |
| Domain | Treated as subject-specific order inside Module |
| Repository | Not required; used only as an internal helper inside Module when needed |
| Read Model / Report | Separated as read-only Query |

## Where HUMQ Works Well

HUMQ is especially effective in systems like these:

- Many business flows.
- Unavoidable exception handling and temporary measures.
- A growing number of tables.
- Many lists, searches, and aggregations spanning multiple tables.
- A need to keep code placement consistent as the team grows.
- A need to preserve responsibility boundaries during long-term operation.

## What HUMQ Requires

HUMQ gives freedom to Usecase. Therefore, the person writing Usecase must take responsibility.

- Make consistency explicit in Usecase.
- Do not pollute Module for convenience.
- Keep Query read-only.
- Keep Handler thin.
- Prioritize responsibility boundaries over convenience.

HUMQ's weakness is that consistency is not automatically protected. But that weakness is also its strength: consistency is not hidden, and it appears in code as business intent.

## Comparison Table

| Viewpoint | Common layered structure | DDD | HUMQ |
| --- | --- | --- | --- |
| Responsibility boundaries | Service often becomes ambiguous | Depends on Aggregate design | Fixed by four layers |
| Business exceptions | Easily scattered across Service or Domain | Can pollute Domain | Absorbed by Usecase |
| Cross-cutting reads | Easily mixed into Repository or Service | Requires separate Read Model design | Separated into Query |
| Transactions | Often placed in Service | Aggregate unit | Usecase unit |
| Reusability | Depends on Service granularity | Depends on Aggregate boundaries | Easier to preserve at Module level |
| Long-term operation | Often drifts by developer | Design difficulty is high | Reduces ambiguity about where code belongs |
