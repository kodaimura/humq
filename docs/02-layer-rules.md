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

### Belongs Here

- Business flows
- Conditional branches
- Combining multiple Modules
- Using Query
- Calling mailers or external clients when a business flow needs them
- Transaction boundaries
- Exceptional business requirements

### Does Not Belong Here

- Direct dependency on HTTP requests or responses
- Cross-cutting reads written as raw SQL
- Direct dependency on ORM models
- Designs that delegate `commit` / `rollback` to Module
- Unrelated business operations bundled together only because they share some code

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

`usecases/accounts/signup.py` is enough because the directory already provides the context. Avoid repeating the target in names like `register_user.py`.

## Module

A Module is closed around exactly one table.

Module is not a Business Usecase, Aggregate, or broad domain subject. It is a small, stable unit of local table order. If a business operation spans three tables, HUMQ prefers three Modules coordinated by one Usecase over one large Module.

This constraint keeps Modules stable and local. Modules do not call other Modules. Business flows that span multiple tables belong in Usecase.

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

Tableless concerns should not be placed under `modules/`.

### About Repository

Repository is not required by HUMQ. Module may use an ORM directly.

Only extract `repository.py` as an internal helper inside Module when persistence code grows too large or when you need to hide storage other than an ORM. Even then, Repository is not a public layer and should not be called directly from Usecase.

## Query

Query is a read-only observation rule for cross-table reads. It handles joins, aggregations, reports, and list reads that span multiple tables.

Single-table reads belong in Module. Query exists for reads whose meaning comes from observing multiple tables or a reporting/search context.

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
