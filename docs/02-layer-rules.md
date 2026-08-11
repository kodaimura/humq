# Layer Rules

> Japanese: [02-layer-rules.ja.md](02-layer-rules.ja.md)

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

Usecase represents business intent. It combines multiple Modules and absorbs real-world exceptions, branches, consistency, and transaction boundaries.

### Belongs Here

- Business flows
- Conditional branches
- Combining multiple Modules
- Using Query
- Transaction boundaries
- Exceptional business requirements

### Does Not Belong Here

- Direct dependency on HTTP requests or responses
- Cross-cutting reads written as raw SQL
- Direct dependency on ORM models
- Designs that delegate `commit` / `rollback` to Module

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

Module owns internal order. As a rule, it provides operations closed around one table or one subject.

### Belongs Here

- Reads, creates, updates, and deletes closed around one subject
- Subject-specific invariants
- Meaningful operations implemented with an ORM or similar tool
- Thin operations that treat an external integration as one subject

### Does Not Belong Here

- Business flows spanning multiple Modules
- Temporary branches driven by a specific Usecase
- Transaction management
- Direct dependency on other Modules
- Reads centered on joins or aggregations

### Naming

Module names should be singular nouns.

```text
modules/
├── account/
│   ├── model.py
│   └── module.py
├── project/
│   ├── model.py
│   └── module.py
└── mail/
    └── module.py
```

### About Repository

Repository is not a required HUMQ layer. Module may use an ORM directly.

Only extract `repository.py` as an internal helper inside Module when persistence code grows too large or when you need to hide storage other than an ORM. Even then, Repository is not a public layer and should not be called directly from Usecase.

## Query

Query is a read-only observation layer. It handles joins, aggregations, reports, and list reads that span multiple tables.

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
