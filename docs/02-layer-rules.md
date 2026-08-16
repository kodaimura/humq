# Layer Rules

This document defines what belongs in each HUMQ layer, what does not, and how files should be named.

## Handler

Handler is the entry point from the outside world. It receives inputs such as HTTP requests, events, or CLI commands, then passes them to Usecase.

### Belongs Here

- Routing
- Receiving requests
- Shaping responses
- Connecting input/output schemas
- Passing context obtained from the outside world, such as the authenticated user

### Does Not Belong Here

- Business logic
- Database operations
- Transaction management
- Combining multiple Modules
- Building aggregations or joins

### Naming

Handler filenames should match URL resources and use plural nouns.

```text
handlers/
├── accounts.py
├── projects.py
├── users.py
└── health.py
```

## Usecase

Usecase represents one explainable business intent. It combines multiple Modules and absorbs real-world exceptions, branches, consistency, and transaction boundaries.

The goal of Usecase is not to make its caller look short. It is to make the important steps of one business operation traceable in one place.

### Belongs Here

- Business flows
- Conditional branches
- Combining multiple Modules
- Using Query
- Calling mailers or external clients when a business flow needs them
- Transaction boundaries
- State transitions and handling for external side-effect failures
- Exceptional business requirements

### Does Not Belong Here

- Direct dependency on HTTP requests or responses
- Cross-cutting reads written as raw SQL
- Direct dependency on ORM models
- Designs that delegate `commit` / `rollback` to Module
- Calling another Usecase in a way that hides the primary business flow
- Unrelated business operations bundled together only because they share some code

### One Usecase = One Primary Flow

One Usecase shows one explainable business intent and its primary flow in one file. At minimum, that file must reveal:

- The order of business-significant operations
- Important branches
- Which Modules and Query code are used
- Where external I/O occurs
- State changes and transaction boundaries

Length alone does not make a Usecase bad. A 300-line Usecase whose business flow is understandable from top to bottom may be preferable to 100 lines scattered across several Services.

Length does not excuse disorder. Reconsider the Usecase boundary or expression when unrelated intents are mixed, deep nesting obscures order, or shared mutable state keeps growing.

### Helpers and Shared Rules

Local calculations, data conversions, and formatting may be extracted into functions or helper files when doing so does not hide the business flow.

When multiple Usecases need the same business decision, it may be shared as a pure Policy or decision function. That helper must not call Module, Query, or external clients and must not own transactions. The Policy invocation and the branch based on its result remain explicit in Usecase.

A helper is not a new Service layer. Moving coordination of multiple Modules or external I/O into a helper would hide the business flow again.

### Naming

Usecase directories should use plural resource names, like Handler. Each Usecase file should be a verb or verb phrase.

```text
usecases/
├── accounts/
│   ├── signup.py
│   ├── login.py
│   ├── update_profile.py
│   └── delete.py
└── projects/
    ├── create.py
    └── archive.py
```

`usecases/accounts/signup.py` is enough because the directory already provides the context. Avoid repeating the target in names like `register_user.py`. Expose one Usecase entry point and keep its primary flow traceable from that file.

## Module

A Module is closed around exactly one table.

Module is not a Business Usecase, Aggregate, or broad domain subject. It is a small, stable unit of local table order. If a business operation spans three tables, HUMQ prefers three Modules coordinated by one Usecase over one large Module.

This constraint removes recurring judgments about relatedness or Aggregate boundaries and makes the operation target predictable from the Module name. Modules do not call other Modules. Business flows that span multiple tables belong in Usecase.

Each writable table managed by HUMQ has exactly one Module that owns its writes. Query may read that table, but another Module must not expose a second write API for it. A join table that is written also has its own Module.

### Belongs Here

- Reads, creates, updates, and deletes closed around one table
- Simple single-table lookups used by Usecase
- Table-specific invariants
- Meaningful operations implemented with an ORM or similar tool

### Does Not Belong Here

- Business flows spanning multiple Modules
- Operations spanning multiple tables
- Temporary branches driven by a specific Usecase
- Business logic extracted only to make Usecase look cleaner
- Transaction management
- Direct dependency on other Modules
- Reads centered on joins or aggregations
- External integrations that do not correspond to a table, such as mailers, payment gateways, or API clients
- Hidden reads from another table through ORM relationship lazy loading
- Hidden writes to another table through ORM cascades, event hooks, or callbacks

A Module operation must not silently modify another application table. If a database trigger or referential cascade is unavoidable, document the behavior so it remains traceable from Usecase and manage it as an architectural exception to the normal HUMQ rule.

### Naming

Module names should be singular nouns corresponding to one table.

```text
modules/
├── account/
│   ├── model.py
│   └── module.py
├── project/
│   ├── model.py
│   └── module.py
└── account_role/
    ├── model.py
    └── module.py
```

Tableless concerns should not be placed under `modules/`. Views and materialized views are treated as read-only Query code, not as write-owning Modules.

### About Repository

Repository is not required by HUMQ. Module may use an ORM directly.

Only extract `repository.py` as an internal helper inside Module when persistence code for the same table grows too large or when separating a non-ORM access implementation. Even then, Repository is not a public layer and should not be called directly from Usecase. Extracting Repository does not change the rule that Module operates on one table.

## Query

Query is a read-only observation rule for cross-table reads. It handles joins, aggregations, reports, and cross-table list reads.

Single-table reads belong in Module even when the search or aggregation is complex. Query exists for reads whose meaning comes from observing multiple tables or from a report/search context that spans tables.

Query does not own transaction boundaries. It does not call `commit` or `rollback`; whether the underlying ORM or database connection uses a transaction internally is a separate implementation detail.

### Belongs Here

- JOIN
- Aggregations
- Report retrieval
- Reads for search screens
- Reads that span multiple Modules

### Does Not Belong Here

- Writes
- Transaction management
- Changes to business state
- CRUD that replaces Module

### Naming

Query should be named by what it observes or by business context, not by entity name.

| Observation target | Filename |
| --- | --- |
| Account order history | `account_orders.py` |
| Project progress | `project_progress.py` |
| Sales summary | `sales_report.py` |
| Overall user activity | `activity_overview.py` |

Query describes what it observes, not which table it comes from.

---

Previous: [Overview](01-overview.md) | Next: [Design Principles](03-design-principles.md)
