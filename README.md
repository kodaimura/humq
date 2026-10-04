# HUMQ - An Architecture for Designing Chaos

<img align="right" src="assets/logo.png" alt="HUMQ logo" width="190">

**Language:** [日本語](README.ja.md) | English

For applications centered on a relational database, HUMQ is a lightweight application architecture<br>
that makes it easier to predict where to write code and where to trace a change as business branches and exceptions grow.

## Four Responsibilities

HUMQ assigns caller input and output to Handler, business flows and transactions to Usecase,<br>
one-table reads and writes to Module by default, and cross-table reads to Query.<br>
Usecase keeps the primary business flow visible. When details are extracted,<br>
named processing within its responsibility keeps business decisions and state changes traceable from Usecase.

## How a Structure Becomes Distorted

Order confirmation may begin as a simple flow that saves the order, decreases inventory, and sends a notification.<br>
Once the system enters real operation, branches and special cases appear that do not fit the original design.<br>
HUMQ calls this business complexity "chaos."

- “Customers on legacy contracts must continue using the previous price.”
- “Support staff must be able to cancel a confirmed order from an administration screen.”
- “A different approval path is required during the month-end busy period.”
- “Orders created by data migration must not send notifications.”

Every one of these requirements may be necessary, but without a defined place for the implementation,<br>
the developer chooses a locally reasonable criterion.

- “It applies only to the administration screen, so it belongs in Controller.”
- “It is a rule about order state, so it belongs in Model.”
- “It coordinates several operations, so it belongs in Service.”

None of these decisions is wrong at the time.<br>
But one special-case placement becomes the precedent for the next, distributing the same business flow across multiple layers.<br>
When chaos has no defined place, responsibility boundaries become ambiguous and the structure begins to distort.<br>
HUMQ calls this state, in which the placement of complexity has broken down, "distortion."<br>
The code may still work, but if its correct placement cannot be explained, the structure is already distorted.

> **Bugs can be fixed. Distortion eventually becomes unmanageable.**

## HUMQ's Solution

HUMQ fixes the responsibility boundaries of Handler, Usecase, Module, and Query,<br>
so where code belongs and where a change should be traced remain predictable as the business grows more complex.<br>
The order processing above follows these boundaries.<br>
Usecase retains the purpose, main order, result branches, transaction boundaries,<br>
external I/O, and how failures are handled.

HUMQ does not eliminate chaos. It allows necessary chaos within order<br>
and uses responsibility boundaries to limit where it belongs and how far its effects may spread.<br>
This reduces structural distortion and keeps the rest of the structure simple and predictable.

> **The structure narrows placement decisions and makes code easier to find.**

## Target Applications and Tradeoffs

Its primary target is applications that use a relational database as their main persistence model<br>
and handle multi-table state changes and cross-table reads.

In exchange for that lighter structure, each Usecase explicitly protects consistency across multiple tables.<br>
Rather than protecting every domain with an Aggregate,<br>
HUMQ treats domains that require structural protection as the exception.

[Adoption and Tradeoffs](docs/05-comparison.md#adoption-and-tradeoffs) explains HUMQ's benefits and drawbacks,<br>
where it fits, and when another design may be more appropriate.

## Extracting and Placing Internal Processing

Business processing worth explaining, verifying, or changing independently may be extracted into a clearly named internal file,<br>
even if only one Usecase uses it. The owning domain's Usecase directory is a useful starting point.<br>
Extraction is optional; business decisions may also remain in the Usecase file.

For an extracted business rule, `usecases/<domain>/_policies.py` is the recommended starting point,<br>
not a required Policy category; a more specific filename may suit the processing.<br>
Teams choose where cross-domain processing is easiest to own and find.<br>
Database-using internal processing joins the calling Usecase's Session and does not own a transaction boundary.

## Comparison with Existing Architectures

MVC + Service, aggregate-centered DDD, and Clean Architecture<br>
are all widely used application design approaches.<br>
However, each leaves some responsibility boundaries to human design judgment.

Distortion arises not only from wrong decisions, but when multiple reasonable decisions coexist.

### MVC + Service

MVC + Service separates responsibilities. Teams can also establish clear conventions for<br>
business logic, dependencies, and transactions within that structure, making placement and tracing similarly predictable.

For example, even the single requirement "Cancel a confirmed order from the administration screen"<br>
can lead to different decisions depending on which aspect the developer emphasizes.

- Put it in Controller because it exists only on the administration screen.
- Put it in Model because it is a rule that changes order state.
- Put it in Service because it also coordinates inventory restoration and notification.

Each decision has a rationale.<br>
Without those conventions, MVC + Service alone does not determine which aspect should take priority<br>
as the placement criterion. When developers choose different criteria, the same business flow<br>
can become distributed across multiple layers.

### Aggregate-Centered DDD

Aggregate-centered DDD makes invalid states harder to create by concentrating invariants and behavior in the Domain.<br>
Because Aggregate and Domain Service boundaries are designed from business concepts,<br>
the team must continue to maintain the same model and placement decisions.

### Clean Architecture

Clean Architecture defines dependency direction and separates business rules from frameworks and databases.<br>
It still leaves decisions such as whether a rule belongs in Entity or Usecase, and how large a Usecase should be.<br>
Dependencies can remain correct while differing decisions make code placement vary between developers.

HUMQ does not reject these designs.<br>
For relational-database-centered applications, it uses explicit responsibility boundaries<br>
to reduce placement decisions and make the location of processing easier to predict.<br>
It fixes more placement boundaries than MVC + Service does by itself<br>
and requires fewer design concepts than DDD centered on Aggregates and a rich Domain Model.<br>
Usecase size, the Module and Query boundary, and when to extract internal processing still require judgment.

## Documentation

- [Overview](docs/01-overview.md)
- [Layer rules](docs/02-layer-rules.md)
- [Design principles](docs/03-design-principles.md)
- [Handling consistency](docs/04-consistency-and-transactions.md)
- [Architecture and design pattern comparison](docs/05-comparison.md)
- [Adoption and tradeoffs](docs/05-comparison.md#adoption-and-tradeoffs)
- [FastAPI example](docs/06-fastapi-example.md)
- [Implementation example (humq-sample)](https://github.com/kodaimura/humq-sample)
- [Adoption limits and evolution](docs/07-adoption-limits-and-evolution.md)

## License

[MIT](LICENSE)
