# HUMQ - An Architecture for Designing Chaos

<img align="right" src="assets/logo.png" alt="HUMQ logo" width="190">

**Language:** [日本語](README.ja.md) | English

For applications centered on a relational database, HUMQ assigns<br>
caller input and output to Handler, business flows and transactions to Usecase,<br>
reads and writes of exactly one table to Module, and cross-table reads to Query.<br>
Instead of splitting complexity into abstractions that hide it,<br>
HUMQ keeps that complexity traceable from Usecase.

> **Bugs can be fixed. Distortion eventually becomes unmanageable.**

HUMQ is an application architecture that does not tolerate structural distortion.

Here, "distortion" means that responsibility boundaries become ambiguous,<br>
so the structure can no longer determine where an operation belongs.

In real operations, branches and special cases emerge that do not fit<br>
the original design. When such chaos has no defined place,<br>
the structure begins to distort.<br>
HUMQ does not try to eliminate such chaos and impose order on everything.<br>
It allows necessary chaos within order and uses responsibility boundaries<br>
to limit where that chaos belongs and how far its effects may spread.

> **The structure—not the individual developer—decides where code belongs.**

## How a Structure Becomes Distorted

Order confirmation may begin as a simple flow that saves the order, decreases inventory, and sends a notification.<br>
Once the system enters real operation, requirements appear that do not fit the original design.

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
The code may still work, but if its correct placement cannot be explained, the structure is already distorted.

## Why Structures Become Distorted in Existing Designs

MVC + Service, aggregate-centered DDD, and Clean Architecture<br>
are all widely used application design approaches.<br>
However, each leaves some responsibility boundaries to human design judgment.

Distortion arises not only from wrong decisions, but when multiple reasonable decisions coexist.

### MVC + Service

MVC + Service separates responsibilities but does not define one exclusive home for business logic.

For example, even the single requirement "Cancel a confirmed order from the administration screen"<br>
can lead to different decisions depending on which aspect the developer emphasizes.

- Put it in Controller because it exists only on the administration screen.
- Put it in Model because it is a rule that changes order state.
- Put it in Service because it also coordinates inventory restoration and notification.

Each decision has a rationale.<br>
MVC + Service alone does not determine which aspect should take priority as the placement criterion.<br>
When developers choose different criteria, the same business flow becomes distributed across multiple layers.

### Aggregate-Centered DDD

Aggregate-centered DDD makes invalid states harder to create by concentrating invariants and behavior in the Domain.<br>
Because Aggregate and Domain Service boundaries are designed from business concepts,<br>
the team must continue to maintain the same model and placement decisions.

### Clean Architecture

Clean Architecture defines dependency direction and separates business rules from frameworks and databases.<br>
It still leaves decisions such as whether a rule belongs in Entity or Usecase, and how large a Usecase should be.<br>
Dependencies can remain correct while differing decisions make code placement vary between developers.

HUMQ does not reject these designs.<br>
For relational-database-centered applications, it replaces some human placement decisions with mechanical responsibility boundaries<br>
so code placement remains stable as people change.

## HUMQ's Solution

HUMQ is a lightweight architecture for RDB-centered applications<br>
that keeps code placement from depending on individual judgment.

It fixes the responsibility boundaries of Handler, Usecase, Module, and Query,<br>
so where code belongs and where a change should be traced remain stable as the business grows more complex.<br>
It limits where chaos is absorbed and keeps the rest of the structure simple and predictable.

It provides clearer placement rules than MVC + Service<br>
and requires fewer design concepts than DDD centered on Aggregates and a rich Domain Model.

## Target Applications

Its primary target is applications that use a relational database as their main persistence model<br>
and handle multi-table state changes and cross-table reads.<br>
[Adoption and Tradeoffs](docs/05-comparison.md#adoption-and-tradeoffs) explains HUMQ's benefits and drawbacks,<br>
where it fits, and when another design may be more appropriate.

## Documentation

- [Overview](docs/01-overview.md)
- [Layer rules](docs/02-layer-rules.md)
- [Design principles](docs/03-design-principles.md)
- [Handling consistency](docs/04-consistency-and-transactions.md)
- [Architecture and design pattern comparison](docs/05-comparison.md)
- [Adoption and tradeoffs](docs/05-comparison.md#adoption-and-tradeoffs)
- [FastAPI example](docs/06-fastapi-example.md)
- [Adoption limits and evolution](docs/07-adoption-limits-and-evolution.md)

## License

[MIT](LICENSE)
