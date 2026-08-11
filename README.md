# HUMQ - Designing Chaos in Architecture

<img align="right" src="assets/logo.png" alt="HUMQ logo" width="190">

> 日本語版はこちら: [README.ja.md](README.ja.md)

HUMQ is a software architecture principle that redefines responsibilities often blurred in existing architectures into four practical rules:<br>
Handler, Usecase, Module, and Query.<br>
It does not try to make every part of a business system clean.<br>
Instead, it designs constraints for where real-world complexity is allowed to live.

Handler owns the boundary with the outside world.<br>
Usecase carries one explainable business intent, including orchestration, branching, consistency, and transaction boundaries.<br>
Module protects local internal order around exactly one table, including single-table reads and writes.<br>
Query observes the system through read-only, cross-table views.

HUMQ is not about making systems impossible to break.<br>
It is about knowing where to clean up when they do break,<br>
without letting responsibility boundaries distort.

> Do not force everything into order. Allow the necessary chaos inside the order.
>
> Break, but do not distort.

## Documentation

- [Overview](docs/01-overview.md)
- [Layer rules](docs/02-layer-rules.md)
- [Design principles](docs/03-design-principles.md)
- [Consistency and transactions](docs/04-consistency-and-transactions.md)
- [Comparison with existing architectures](docs/05-comparison.md)
- [FastAPI example](docs/06-fastapi-example.md)

## One-Sentence Definition

HUMQ redefines ambiguous application responsibilities into four practical rules that separate where order is protected from where business chaos is absorbed.

## Four Responsibility Rules

| Layer | Role | Main responsibility |
| --- | --- | --- |
| Handler | Boundary | Receives HTTP requests or events and transforms input/output |
| Usecase | Intent and reality | Handles business flows, consistency, and transactions |
| Module | Local order | Provides reads and writes closed around exactly one table |
| Query | Observation | Handles read-only joins, aggregations, and cross-table reads |

## Basic Principles

- Keep Handler thin.
- Let Usecase absorb real-world complexity, but keep each Usecase tied to one explainable business intent.
- Keep Module closed around exactly one table, unaware of other Modules.
- Keep Query read-only. Query does not own transaction boundaries.
- Put transaction boundaries in Usecase.
- Allow breakage, but do not allow responsibility boundaries to distort.

## License

[MIT](LICENSE)
