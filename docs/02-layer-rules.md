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
- Direct calls to Module, Query, or external clients
- Database operations or transaction management
- Joins or aggregations

## Usecase

Usecase represents one explainable business operation and its primary flow.<br>
It absorbs real-world branches and special cases by explicitly combining the required Operations,<br>
Modules, Query code, and external clients.

### Belongs Here

- Business-significant operation order and branches
- Combining Operations and Modules, and reading through Query
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

### Helpers, Shared Rules, and Operation

Local calculations and data conversions may be extracted into functions when doing so does not hide the business flow.<br>
Business decisions used by multiple Usecases may be shared as side-effect-free Policies or functions.<br>
The Policy call and any branch based on its result remain visible in Usecase.

Avoid calls from one Usecase to another. When an internal business operation is reused by multiple Usecases,<br>
it may be extracted as an Operation within the Usecase responsibility.

Operation is not a new HUMQ layer. It may call Module, Query, and external clients,<br>
but it participates in the caller's Session and transaction and never calls `begin`, `commit`, or `rollback`.
Consider extracting an Operation only when all of the following are true:

- It is genuinely reused by multiple Usecases.
- It has an independent business meaning.
- It does not hide the primary flow of the calling Usecase.

Do not share unrelated operations merely because some code looks similar.<br>
When multi-Module operations must be shared, express them as an Operation with a business-specific name,<br>
not as an ambiguously named helper or Service.

## Module

As a rule, Module reads and writes exactly one table<br>
and groups that table's standard data operations. Each table written by HUMQ has<br>
exactly one Module that owns its operations.

As a result, a normalized table structure may appear in Usecase as multiple Module calls.<br>
This is an intentional consequence of prioritizing a mechanical answer to where a table is changed<br>
over abstract Domain boundaries.

As an exception, when writing its owned table requires information from another table,<br>
Module may read it through SELECT, joins, or subqueries. It must not change the other table<br>
or use this exception to broaden its normal read scope.

### Belongs Here

- Creating, updating, and deleting rows in the one owned table
- Standard retrieval such as lookup by primary key, existence checks, and standard lists
- Basic searches for the owned table that recur across multiple Usecases
- Constraints and state changes decidable from that table's values alone
- Concurrency control for one table, such as conditional updates or locks
- As an exception, reads from other tables required to write the owned table
- Persistence implementation details using an ORM or SQL

### Does Not Belong Here

- Writes to any table it does not own
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

Query may read a table owned by Module, but another Module must not expose<br>
a second write API for the same table. A writable join table also has its own Module.

Repository is not a HUMQ layer and is not required.<br>
Use it only as an internal Module implementation when persistence code must be separated.<br>
Usecase does not call Repository directly, and extracting one does not change the one-table boundary.

## Query

Query reads across multiple tables to build models required by screens, searches, reports,<br>
and similar purposes. As a rule, choose between Module and Query by asking whether the operation<br>
is a standard operation on one table or a read that spans multiple tables.

Standard operations for a Module's owned table, such as lookup by primary key,<br>
existence checks, and standard lists, belong in Module. As an exception, a complex query specific to a screen<br>
or display DTO, a window function, specialized SQL statement, or other read that does not fit Module's<br>
standard operations may belong in Query even when it reads only one table.

Even when joins, aggregations, and search conditions make Query long,<br>
do not split it by line count while it still represents one observation purpose or read model.

### Belongs Here

- Reads across multiple tables for screens, searches, reports, CSV exports, dashboards, and analysis
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
- **Operation**: Place internal operations shared by multiple Usecases directly under `usecases/` and give them business-specific names.
- **Module**: Use a singular noun representing the corresponding table.
- **Query**: Name it after what it observes or its business context, not after a table.
- **External client**: Name it after the external service or communication capability it provides.

```text
handlers/accounts.py
handlers/events/order_created.py
handlers/cli/rebuild_index.py
usecases/accounts/signup.py
usecases/accounts/list_accounts.py
usecases/register_order.py
modules/account/module.py
modules/account_role/module.py
queries/account_orders.py
queries/sales_report.py
clients/payment_gateway.py
clients/shipping_api.py
```

`usecases/register_order.py` is an example of an Operation shared by multiple Usecases.<br>
Because Operation is not a new layer, do not create an `operations/` directory.
If too many Operations accumulate at the root, reconsider whether too much has been extracted<br>
before adding another directory.

Avoid names such as `common.py` or `shared.py`; name each file after the responsibility it represents.

---

Previous: [Overview](01-overview.md) | Next: [Design Principles](03-design-principles.md)
