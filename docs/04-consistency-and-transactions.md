# Handling Consistency

In HUMQ, consistency across multiple tables and database transaction boundaries are Usecase responsibilities.<br>
This chapter explains the risks that this design accepts<br>
and how Usecase, Module, Database, and tests protect consistency.

## HUMQ's Tradeoff

HUMQ does not structurally guarantee consistency across multiple tables.<br>
Even when Handler, Usecase, Module, and Query follow their responsibility boundaries correctly,<br>
an omitted business step or consistency rule can allow inconsistent data to be committed.

This is a HUMQ constraint and an intentional tradeoff for lightweight, explicit placement rules.<br>
Database constraints and Usecase tests reduce the risk, but they cannot express every business invariant<br>
or make implementation omissions impossible by construction.<br>
Where that risk is unacceptable, choose a design such as aggregate-centered DDD<br>
that confines invariants within a Domain Model.

## Responsibilities

- **Usecase**: Owns cross-table consistency, operation order, failure conditions, and transaction boundaries.
- **Optional internal business processing**: When extracted, handles decisions, calculations, or consistency processing; it may use Module and Query but does not own a transaction boundary.
- **Module**: By default, reads and writes one table and does not call `commit` or `rollback`.
- **Query**: Is read-only and does not own transaction boundaries.
- **Database**: Enforces database-expressible constraints and provides concurrency-control mechanisms.
- **Test**: Unit tests pure decisions when useful; verifies consistency, failures, and calling-Usecase `rollback` for database-using processing.

## Transaction Boundaries

A transaction boundary is determined by which state changes must be established together as a business operation,<br>
not by the convenience of a table or Module.

For example, if confirming an order, reserving inventory, and registering a delivery request<br>
would leave invalid state when any one is missing, they belong in the same transaction.
The following sketch omits Module setup and exception definitions to focus on the transaction flow.

```python
# usecases/orders/confirm_order.py
from usecases.inventory._reservation import reserve_inventory

def confirm_order(session, order_id: int) -> None:
    with session.begin():
        order = order_module.get_for_update(session, order_id)
        items = order_item_module.list_by_order(session, order_id)
        unavailable_product_id = reserve_inventory(session, items)
        if unavailable_product_id is not None:
            raise InsufficientInventory(unavailable_product_id)
        order_module.mark_confirmed(session, order.id)
        outbox_module.enqueue_order_confirmed(session, order.id)
```

Inventory reservation has its own business meaning. This example chooses to keep its detail in a separate file<br>
in the inventory domain; it could also remain inside the Usecase:

```python
# usecases/inventory/_reservation.py
def reserve_inventory(session, items) -> int | None:
    for item in items:
        updated = inventory_module.decrease_if_available(
            session,
            product_id=item.product_id,
            quantity=item.quantity,
        )
        if not updated:
            return item.product_id
    return None
```

The internal processing uses the calling Usecase's Session and does not own a transaction boundary.<br>
When inventory is insufficient, the Usecase raises inside its transaction,<br>
rolling back preceding inventory updates before confirming the order or enqueuing the delivery request.<br>
Usecase shows the primary order and transaction; the referenced file shows the reservation's checks and changes.

HUMQ does not require `1 Usecase = 1 Transaction`.<br>
A read-only Usecase may not need an explicit transaction.<br>
A business flow spanning multiple requests or external systems may use multiple transactions.<br>
Even then, Usecase keeps each confirmed state and post-failure policy traceable.

## Consistency Enforced by the Database

Keeping the primary flow in Usecase does not prevent omissions or inconsistencies caused by concurrent updates.

Rules expressible with `UNIQUE`, `NOT NULL`, `CHECK`, or foreign keys are enforced as database constraints.<br>
Operations such as decrementing inventory, where multiple requests can update the same data,<br>
use conditional updates, row locks, optimistic locking, or other required concurrency control inside Module operations.

Usecase makes explicit the business condition under which each operation is called and how its failure is handled.

## Consistency with External Systems

Sending email or calling a payment API cannot be handled in the same transaction as the database.<br>
The database may `rollback` after the external operation succeeds, or the external operation may fail after the database `commit` succeeds.

- Run notifications whose failure is acceptable after the database `commit`.
- When a delivery request must not be lost, record it in an outbox table in the same transaction.
- When external state changes, as with payments, design idempotency, retries, and compensation.

With Outbox, Usecase does not guarantee completion of the external operation.<br>
It guarantees that the database state change and the delivery request are recorded together.

## Internal Processing and Verification

Independently meaningful processing may be extracted even when only one Usecase calls it;<br>
it may also remain in Usecase. Separate policy or other internal files are optional.<br>
Pure decisions and calculations need no Session and can be unit tested from their inputs and outputs.<br>
Database-using decisions or consistency processing use the same Session as the calling Usecase,<br>
read through Module or Query, and write through Module. They do not issue direct ORM or SQL<br>
data access, create an independent Session, or call `begin`, `commit`, or `rollback`.

Test database-using processing against consistency requirements and failure cases.<br>
Also test that the calling Usecase rolls back all changes when a later step fails.<br>
Extraction makes the implementation easier to inspect and test, but does not by itself<br>
prevent an omitted call or provide a structural guarantee of cross-table consistency.

---

Previous: [Design Principles](03-design-principles.md) | Next: [Architecture and Design Pattern Comparison](05-comparison.md)
