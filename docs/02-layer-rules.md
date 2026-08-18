# Layer Rules

HUMQ determines code placement through four responsibilities: Handler, Usecase, Module, and Query.<br>
External clients are destinations for connections to external systems and are not a HUMQ layer.

## Handler

Handler connects callers such as HTTP, events, and CLI to the application.<br>
It passes caller input to Usecase and converts the result into the caller's format.

### Belongs Here

- Entry points for routes, event subscriptions, and CLI commands
- Receiving requests and messages
- Input-format validation, such as required fields and types
- Passing external context, such as an authenticated user or request metadata
- Calling Usecase and converting results into responses or status codes

### Does Not Belong Here

- Business conditions or authorization decisions
- Calls to Module, Query, or external clients that bypass Usecase
- Database operations or transaction management
- Joins or aggregations

## Usecase

Usecase represents one explainable business operation and its primary flow.<br>
It absorbs real-world branches and special cases by explicitly combining the required Modules,<br>
Query code, and external clients.

### Belongs Here

- Business-significant operation order and branches
- Combining Modules and reading through Query
- State transitions and consistency decisions
- Transaction boundaries
- External client calls and post-failure policy
- Special-case business requirements

### Does Not Belong Here

- Dependencies on external formats such as HTTP requests or responses
- Direct database access through ORM sessions or query APIs
- Modifying and persisting ORM models directly
- Direct reads written through an ORM or raw SQL
- External-client implementation details such as communication mechanics or response parsing
- Multiple unrelated business operations
- Services or helpers that hide the primary business flow

Usecase may receive a SQLAlchemy `Session` and pass operation results, including ORM models,<br>
to and from Module and Query. Persistence-independent Domain Entities and conversion into<br>
Input, Output, or DTO types are not required.

Usecase must not use ORM query APIs to retrieve data directly<br>
or modify ORM models and persist them. Persistence operations go through Module,<br>
reads spanning multiple tables go through Query,<br>
and Usecase owns the business flow and transaction.

The Usecase called directly by Handler finalizes the transaction boundary<br>
and is responsible for any required `begin`, `commit`, and `rollback`.

### One Usecase = One Primary Flow

The primary operation order and branches, Modules and Query code used, state changes,<br>
transaction boundaries, external I/O, and post-failure policy must be readable from the Usecase file.

Judge Usecase by whether one business operation remains traceable from top to bottom, not by its length.<br>
See [Design Principles](03-design-principles.md) for readability and decomposition principles.

When multiple Usecases need the same processing, place decisions and calculations that do not use the database in Policy,<br>
and processing that uses Module or Query in Operation. Both are called by Usecase,<br>
never directly by Handler. They are not new HUMQ layers.

## Policy

Policy makes a decision or calculation only from the values it receives. It does not access the database.<br>
When a function is sufficient, no class is required.

Policy satisfies all of the following conditions:

- It does not use the database, Session, Module, Query, or external clients.
- It never calls `begin`, `commit`, `rollback`, or `flush`.
- It returns the same result for the same input.
- Its call and the primary branch based on its result remain visible in Usecase.

### Policy Placement

Place Policies and pure helpers in this order of preference:

1. A decision or calculation used by one Usecase stays in that Usecase file.
2. A decision shared by Usecases in one domain goes in `usecases/<domain>/_policies.py`.
3. Only domain-independent pure calculations shared across the application go in `usecases/_policies.py`.

```text
usecases/
├── _policies.py
├── procurement/
│   ├── create_order.py
│   ├── receive_goods.py
│   └── _policies.py
├── returns/
│   ├── request_return.py
│   ├── receive_return.py
│   └── _policies.py
└── billing/
    ├── generate_invoice.py
    ├── post_payment.py
    └── _policies.py
```

The leading `_` marks an internal file that Handler does not use directly.<br>
Do not re-export functions or types from `_policies.py` through the domain's `__init__.py`.

### Does Not Belong in Policy

Database reads, Module or Query calls, data updates, and external I/O do not belong in Policy.<br>
Do not move processing into Policy merely to make Usecase look shorter.

For example, the formula for returnable quantity may belong in Policy. Retrieving shipped quantity<br>
and prior returns belongs in Module or Query, while the flow that calls Policy and registers the return remains in Usecase.

## Operation

Operation is database-backed processing shared by multiple Usecases. It may use Module or Query,<br>
but never calls `begin`, `commit`, or `rollback`. Usecase calls it; Handler does not.

Extract shared internal processing as Operation instead of calling one Usecase from another.

Consider Operation for authorization, numbering, duplicate checks, shared preconditions,<br>
or limited multi-table updates used with the same meaning and rules by multiple Usecases.<br>
Processing that does not use the database belongs in Policy, not Operation.

### Operation Placement and Naming

Place Operation in the owning domain's `usecases/<domain>/_operations.py` by default.<br>
Keep Operations together in that file while there are only a few.<br>
Use a `*Operation` class name and normally a `run()` method to distinguish it from public `*Usecase` and `execute()`.

```text
usecases/
└── procurement/
    ├── create_order.py
    ├── receive_goods.py
    ├── _policies.py
    └── _operations.py
```

The leading `_` marks an internal module. Do not re-export `_operations.py` through `__init__.py`.<br>
See [Adoption Limits and Evolution](07-adoption-limits-and-evolution.md)<br>
for splitting rules when Operation no longer fits readably in one file.

### Operation Rules

- Use the same Session as the calling Usecase.
- Module and Query may be called.
- One or more tables may be read or written, but every write goes through its Module.
- Required validation, locking, and `flush` are allowed.
- Never call `begin`, `commit`, or `rollback`.
- Handler must not call Operation directly.
- Do not call another Operation.
- Keep external-client calls and the composition of multiple Operations in Usecase.
- Keep the call and primary branch based on its result visible in Usecase.
- Test success, failure, and rollback by the caller.

External I/O, post-failure policy, and transaction finalization remain in the calling Usecase.

### When to Extract an Operation

Extract an Operation when multiple Usecases genuinely use it, it has the same business meaning and reason to change,<br>
and it must share the same validation, errors, locking, and update order. After extraction,<br>
the primary flow must remain traceable from Usecase.

## Module

By default, Module reads and writes one table.

As a result, a normalized table structure may appear in Usecase as multiple Module calls.<br>
This is an intentional consequence of prioritizing a mechanical answer to where a table is changed<br>
over abstract Domain boundaries.

As an exception, when writing its target table requires information from another table,<br>
Module may read it through SELECT, joins, or subqueries. It must not change the other table<br>
or use this exception to broaden its normal read scope.

### Belongs Here

- Creating, updating, and deleting rows in the one target table
- Standard retrieval such as lookup by primary key, existence checks, and standard lists
- Basic searches for the target table that recur across multiple Usecases
- Constraints and state changes decidable from that table's values alone
- Concurrency control for one table, such as conditional updates or locks
- As an exception, reads from other tables required to write the target table
- Persistence implementation details using an ORM or SQL

### Does Not Belong Here

- Writes to tables other than the target table
- Dependencies on other Modules
- Usecase-specific branches or business flows
- Finalizing transaction boundaries with `commit` or `rollback`
- Communication with external systems
- Hidden writes to another table through ORM cascades, hooks, or callbacks

For example, `InvoiceModule` may create an invoice by reading an order and its items:

```sql
INSERT INTO invoices (order_id, customer_id, amount)
SELECT orders.id, orders.customer_id, SUM(order_items.amount)
FROM orders
JOIN order_items ON order_items.order_id = orders.id
WHERE orders.id = :order_id
GROUP BY orders.id, orders.customer_id;
```

This statement reads `orders` and `order_items`, but only changes `invoices`,<br>
so it remains within `InvoiceModule`'s responsibility.

As a rule, writes to multiple tables are expressed by Usecase calling multiple Modules.<br>
If a multi-table write through one SQL statement or stored procedure cannot be avoided,<br>
document it as an exception to the standard rule: identify the affected tables and rationale,<br>
record an ADR, and protect it with an integration test. Do not expand this exception into a generic Service.

Query may also read a Module's target table.<br>
An intermediate table written by HUMQ has its own Module.

Repository is not a HUMQ layer and is not required.<br>
Use it only as an internal Module implementation when persistence code must be separated.<br>
Usecase does not call Repository directly, and extracting one does not change the one-table boundary.

## Query

Query is read-only. It reads across multiple tables to build models for screens, searches, reports, CSV exports, and analysis.

Standard operations for a Module's target table, such as lookup by primary key,<br>
existence checks, and standard lists, belong in Module. As an exception, a complex query specific to a screen<br>
or display DTO, a window function, specialized SQL statement, or other purpose-specific read that does not fit Module's<br>
standard operations may belong in Query even when it reads only one table.

Even when joins, aggregations, and search conditions make Query long,<br>
do not split it by line count while it still represents one observation purpose or read model.

### Belongs Here

- Read models spanning multiple tables for screens, searches, reports, CSV exports, dashboards, and analysis
- Cross-table joins and aggregations
- As an exception, one-table reads using complex criteria, window functions, JSON, or purpose-specific SQL
- Conversion into read models shaped for a screen or business context

### Does Not Belong Here

- Writes through `INSERT`, `UPDATE`, or `DELETE`
- Changes to ORM model state
- Transaction management through `commit` or `rollback`
- Business flows or state transitions
- Duplication of standard CRUD or basic retrieval already provided by Module

Even when the underlying ORM or database connection uses a transaction,<br>
Query does not own the application-level transaction boundary.

### Read-only Usecases

Query represents how data is read.<br>
Usecase represents which operation the application makes available to a caller.<br>
If Handler calls Query directly, the external interface becomes coupled to the database read structure.

For this reason, a read-only operation still goes through Usecase even when it only delegates to Query.<br>
A thin Usecase is not a problem by itself.

## External Clients

An external client is an adapter that hides communication with an external system.<br>
Usecase calls it, keeping communication mechanics and external data formats out of Usecase.

### Belongs Here

- Communication through HTTP, an SDK, or a message broker
- Building authentication information and requests
- Parsing responses and converting them into application data
- Converting timeouts and communication errors

### Does Not Belong Here

- Business flows or business conditions
- Calls to Module or Query
- Database operations or transaction management
- Business policy for handling an external-operation failure

Usecase decides whether to perform an external operation and whether a failure is retried or compensated.

## Scope of HUMQ

HUMQ defines the Handler, Usecase, Module, and Query responsibility boundaries<br>
for business processing that begins at APIs and similar entry points. It does not prescribe<br>
the directory structure of the entire application or how infrastructure code is divided.

Database connections, configuration, external clients, email, storage, caches, logs,<br>
telemetry, metrics, authentication, migrations, batch jobs, and framework-specific code<br>
may be arranged to fit the project.

Directories such as `core/`, `clients/`, `infrastructure/`, `integrations/`, `adapters/`, and `config/`<br>
are project choices. In particular, `core/` has no official HUMQ-specific meaning.

## Naming

- **Handler**: Place HTTP files directly under `handlers/` and name them after resources.<br>
  Event and CLI files may be grouped into input-specific subdirectories when needed.
- **Usecase**: Place Handler-called Usecases in the corresponding resource directory<br>
  and name files with verbs or verb phrases.
- **Module**: Use a singular noun representing the corresponding table.
- **Query**: Name it after what it observes or its business context, not after a table.
- **External client**: Name it after the external service or communication capability it provides.

```text
handlers/accounts.py
handlers/events/order_created.py
handlers/cli/rebuild_index.py
usecases/accounts/signup.py
usecases/accounts/list_accounts.py
modules/account/module.py
modules/account_role/module.py
queries/account_orders.py
queries/sales_report.py
clients/payment_gateway.py
clients/shipping_api.py
```

---

Previous: [Overview](01-overview.md) | Next: [Design Principles](03-design-principles.md)
