# HUMQ - Designing Chaos in Architecture

<img align="right" src="assets/logo.png" alt="HUMQ logo" width="190">

> 日本語版はこちら: [README.ja.md](README.ja.md)

HUMQ is a software architecture principle for organizing application code into four layers:<br>
Handler, Usecase, Module, and Query.<br>
It does not try to eliminate the exceptions and changes that inevitably exist in real business systems.<br>
Instead, it defines where that complexity should live.

Handler owns the boundary with the outside world.<br>
Usecase carries business intent, orchestration, consistency, and transaction boundaries.<br>
Module protects local internal order around a single subject.<br>
Query observes the system through joins, aggregations, and cross-subject reads.

HUMQ is not about making systems impossible to break.<br>
It is about knowing where to clean up when they do break,<br>
while keeping responsibility boundaries from distorting.

> Do not force everything into order. Allow the necessary chaos inside the order.

## Documentation

- [Overview](docs/01-overview.md)
- [Layer rules](docs/02-layer-rules.md)
- [Design principles](docs/03-design-principles.md)
- [Consistency and transactions](docs/04-consistency-and-transactions.md)
- [Comparison with existing architectures](docs/05-comparison.md)
- [FastAPI example](docs/06-fastapi-example.md)

## One-Sentence Definition

HUMQ separates the layers that protect order from the layers that absorb chaos, creating a structure that is easy to clean up and hard to distort.

## Four Layers

| Layer | Role | Main responsibility |
| --- | --- | --- |
| Handler | Boundary | Receives HTTP requests or events and transforms input/output |
| Usecase | Intent and reality | Handles business flows, consistency, and transactions |
| Module | Internal order | Provides operations closed around one table or one subject |
| Query | Observation | Handles joins, aggregations, and cross-subject reads |

## Basic Principles

- Keep Handler thin.
- Let Usecase absorb real-world complexity.
- Keep Module closed around one subject, unaware of other Modules.
- Keep Query read-only, with no writes or transactions.
- Put transaction boundaries in Usecase.
- Allow breakage, but do not allow responsibility boundaries to distort.
