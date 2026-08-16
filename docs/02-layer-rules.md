# Layer Rules

HUMQ determines code placement through four responsibilities: Handler, Usecase, Module, and Query.<br>
External clients are adapters that hide communication with external systems and are not a HUMQ layer.

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
It absorbs real-world branches and special cases by explicitly combining the required Modules, Query code, and external clients.

### Belongs Here

- Business-significant operation order and branches
- Combining Modules and reading through Query
- State transitions and consistency decisions
- Transaction boundaries
- External client calls and post-failure policy
- Exceptional business requirements

### Does Not Belong Here

- Dependencies on external formats such as HTTP requests or responses
- Direct database access through ORM sessions or query APIs
- Modifying and persisting ORM models directly
- Cross-table reads written as raw SQL
- External-client implementation details such as communication mechanics or response parsing
- Multiple unrelated business operations
- Services or helpers that hide the primary business flow

Usecase may use values returned by Module or Query within the business flow.<br>
It leaves one-table reads and writes to Module and cross-table reads to Query rather than operating the ORM directly.

### One Usecase = One Primary Flow

The primary operation order and branches, Modules and Query code used, state changes,<br>
transaction boundaries, external I/O, and post-failure policy must be readable from the Usecase file.

Judge Usecase by whether one business operation remains traceable from top to bottom, not by its length.<br>
See [Design Principles](03-design-principles.md) for readability and decomposition principles.

### Helpers, Shared Rules, and Shared Usecases

Local calculations and data conversions may be extracted into functions when doing so does not hide the business flow.<br>
Business decisions used by multiple Usecases may be shared as side-effect-free Policies or functions.<br>
The Policy call and any branch based on its result remain visible in Usecase.

When exactly the same business flow appears in multiple Usecases<br>
and can be explained as an independent business operation, it may be extracted into a shared Usecase.<br>
The call and any branch based on its result remain visible in the caller.

A Usecase called directly by Handler finalizes any required transaction boundary.<br>
A shared Usecase called by another Usecase participates in the caller's transaction<br>
and does not `commit` itself.

Do not share unrelated operations merely because some code looks similar.<br>
When multi-Module operations must be shared, express them as a shared Usecase with a business-specific name,<br>
not as an ambiguously named helper or Service.

## Module

Module reads and writes exactly one table.<br>
Each table written by HUMQ has exactly one Module that owns its writes.<br>
A business operation involving multiple tables is expressed by Usecase combining multiple Modules.

As a result, a normalized table structure may appear in Usecase as multiple Module calls.<br>
This is an intentional consequence of prioritizing a mechanical answer to where a table is changed<br>
over abstract Domain boundaries.

### Belongs Here

- Reads, creates, updates, and deletes closed around one table
- Searches and aggregations over one table
- Constraints and state changes decidable from that table's values alone
- Concurrency control for one table, such as conditional updates or locks
- Persistence implementation details using an ORM or SQL

### Does Not Belong Here

- Reads or writes spanning multiple tables
- Dependencies on other Modules
- Usecase-specific branches or business flows
- Finalizing transaction boundaries with `commit` or `rollback`
- Communication with external systems
- Hidden reads of another table through relationship lazy loading
- Hidden writes to another table through ORM cascades, hooks, or callbacks

Query may read a table owned by Module,<br>
but another Module must not expose a second write API for the same table.<br>
A writable join table also has its own Module.

Repository is not a HUMQ layer and is not required.<br>
Use it only as an internal Module implementation when persistence code must be separated.<br>
Usecase does not call Repository directly, and extracting one does not change the one-table boundary.

## Query

Query is responsible for read-only operations that span multiple tables.<br>
Placement depends on the number of tables read, not on the complexity of the read.<br>
A read over one table belongs in Module; a read spanning multiple tables belongs in Query.

Even when joins, aggregations, and search conditions make Query long,<br>
do not split it by line count while it still represents one observation purpose or read model.

### Belongs Here

- Joins and aggregations across multiple tables
- Cross-table reads for lists, searches, and reports
- Conversion into read models shaped for a screen or business context

### Does Not Belong Here

- Writes or changes to business state
- Transaction management through `commit` or `rollback`
- Business flows or state transitions
- Single-table CRUD, searches, or aggregations

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

## Naming

- **Handler**: Place HTTP files directly under `handlers/` and name them after resources. Event and CLI files may be grouped into input-specific subdirectories when needed.
- **Usecase**: Name directories after primary resources or business contexts, and files with verbs or verb phrases.
- **Module**: Use a singular noun representing the corresponding table.
- **Query**: Name it after what it observes or its business context, not after a table.
- **External client**: Name it after the external service or communication capability it provides.

```text
handlers/accounts.py
handlers/events/order_created.py
handlers/cli/rebuild_index.py
usecases/accounts/signup.py
usecases/projects/archive.py
modules/account/module.py
modules/account_role/module.py
queries/account_orders.py
queries/sales_report.py
clients/payment_gateway.py
clients/shipping_api.py
```

Avoid names such as `common.py` or `shared.py`; name each file after the responsibility it represents.

---

Previous: [Overview](01-overview.md) | Next: [Design Principles](03-design-principles.md)
