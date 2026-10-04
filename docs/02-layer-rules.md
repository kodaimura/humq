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
It absorbs real-world branches and special cases by combining the required Modules,<br>
Query code, and external clients, directly or through named internal business processing.

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

The Usecase called directly by Handler owns the transaction boundary<br>
and is responsible for any required `begin`, `commit`, and `rollback`.

### One Usecase = One Primary Flow

The purpose, primary operation order, branches based on results, transaction boundaries,<br>
external I/O, and post-failure policy must be readable from the Usecase file.<br>
The validation, locking, and changes performed by named internal processing must be traceable in its referenced file.

Judge Usecase by whether one business operation remains traceable from top to bottom, not by its length.<br>
See [Design Principles](03-design-principles.md) for readability and decomposition principles.

Business processing may remain in Usecase or be extracted into a clearly named internal file.<br>
Extracting a policy or other business processing into a separate file is optional, not a HUMQ requirement.<br>
Extraction does not create another HUMQ layer or a required abstraction.

## Internal Business Processing

An internal file can hold a pure decision or calculation, a decision based on database information,<br>
or consistency processing that combines multiple Modules. It remains part of the Usecase responsibility.<br>
Whether it uses the database affects its implementation and tests, not a mandatory category or naming rule.

### If You Extract

Even when only one Usecase uses it, a team may extract processing if its business meaning makes it worth<br>
explaining, verifying, and changing independently. Examples include pricing, cancellation eligibility,<br>
returnable quantity, approval routing, authorization, and inventory reservation.<br>
Such processing and small local decisions may also remain in Usecase.<br>
Independent meaning, line count, duplication, and purity do not by themselves require a separate file.<br>
After extraction, Usecase still shows the call, its purpose in the flow, and the primary branch based on its result.

### Placement and Naming

If processing is extracted, a useful starting point is a file directly in the Usecase directory of the business domain that owns it.<br>
For extracted business rules, `usecases/<domain>/_policies.py` is the recommended starting point.<br>
This is a filename convention, not a required Policy category or a rule that the processing must be pure.<br>
When a more specific name makes the purpose easier to find, files such as `_pricing.py` and `_reservation.py` are also valid.

| Processing | File |
| --- | --- |
| Order cancellation flow | `usecases/orders/cancel.py` |
| Cancellation eligibility | `usecases/orders/_policies.py` |
| Pricing | `usecases/orders/_pricing.py` |
| Authorization | `usecases/organizations/_authorization.py` |
| Inventory reservation | `usecases/inventory/_reservation.py` |

The leading `_` marks an internal implementation. Handler does not call it directly,<br>
and it is not re-exported as a public Usecase through `__init__.py`.<br>
Even if orders and shipping both use inventory reservation, inventory may be a clear owner.<br>
For cross-domain processing, a team may instead choose a cohesive top-level `policy/` or another shared package.<br>
HUMQ does not prescribe that choice: use business ownership, discoverability, and dependency direction<br>
to decide where the processing belongs. A `policy/` package does not add a HUMQ layer<br>
or make Policy a required category of internal processing.

The Usecase responsibility and the transaction and data access rules below apply regardless of the package location.

Both a folder per Usecase and a shared folder for internal processing are project choices.<br>
Avoid a catch-all of unrelated rules and nested calls that obscure the primary business flow.<br>
If a domain becomes difficult to scan, a folder within that domain may help.<br>
HUMQ requires no particular class, class name, or method name for this processing.<br>
See [Adoption Limits and Evolution](07-adoption-limits-and-evolution.md) for signs that a domain needs review.

### Transaction and Data Access Rules

Database-using internal processing participates in the calling Usecase's transaction with the same Session.<br>
It does not own a transaction boundary: it never calls `begin`, `commit`, or `rollback`,<br>
and it does not create its own Session or independent transaction.<br>
A pure decision or calculation does not need a Session.

- Read through Module or Query; write through Module.
- Do not retrieve or persist data directly with an ORM or SQL in internal processing.
- Validation, locking, and changes made within the processing must be inspectable in its referenced file.
- Keep transaction boundaries, external-system communication and email, and post-failure policy in Usecase.

Pure processing can be unit tested from its inputs and outputs.<br>
For database-using processing, test consistency and failures with the database,<br>
including that the calling Usecase rolls back the combined changes when required.

## Module

By default, Module reads and writes one table.

As a result, a normalized table structure may appear as multiple Module calls<br>
in Usecase or its named internal business processing.<br>
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

As a rule, writes to multiple tables are expressed by Usecase calling multiple Modules,<br>
directly or through named internal business processing that participates in its transaction.<br>
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
