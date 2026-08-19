# Handling Consistency

HUMQ does not automatically protect consistency across multiple tables through its structure.<br>
If a required Module call or validation is omitted from Usecase, data inconsistency becomes more likely.<br>
This is a major tradeoff that HUMQ accepts.

That makes it essential to define where consistency is protected.<br>
In HUMQ, business consistency across multiple tables<br>
and database transaction boundaries are Usecase responsibilities.

## Responsibilities

- **Usecase**: Owns cross-table consistency, operation order, failure conditions, and transaction boundaries.
- **Module**: By default, reads and writes one table and does not call `commit` or `rollback`.
- **Query**: Is read-only and does not own transaction boundaries.
- **Database**: Enforces database-expressible constraints and provides concurrency-control mechanisms.
- **Test**: Verifies business branches, failures, and `rollback` behavior.

## Transaction Boundaries

A transaction boundary is determined by which state changes must be established together as a business operation,<br>
not by the convenience of a table or Module.

For example, if confirming an order, reserving inventory, and registering a delivery request<br>
would leave invalid state when any one is missing, they belong in the same transaction.

```python
# usecases/orders/confirm_order.py

def confirm_order(session, order_id: int) -> None:
    with session.begin():
        order = order_module.get_for_update(session, order_id)
        items = order_item_module.list_by_order(session, order_id)

        for item in items:
            updated = inventory_module.decrease_if_available(
                session,
                product_id=item.product_id,
                quantity=item.quantity,
            )
            if not updated:
                raise InsufficientInventory(item.product_id)

        order_module.mark_confirmed(session, order.id)
        outbox_module.enqueue_order_confirmed(session, order.id)
```

When inventory is insufficient, an exception causes all preceding inventory updates,<br>
the order confirmation, and the delivery request to `rollback` together.<br>
Usecase shows which Modules are combined and which writes fail together.

HUMQ does not require `1 Usecase = 1 Transaction`.<br>
A read-only Usecase may not need an explicit transaction.<br>
A business flow spanning multiple requests or external systems may use multiple transactions.<br>
Even then, Usecase keeps each confirmed state and post-failure policy traceable.

## Consistency Enforced by the Database

Placing an operation in Usecase does not prevent omissions or inconsistencies caused by concurrent updates.

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

## When the Same Invariant Repeats

Write business flows and consistency decisions directly in each Usecase by default.<br>
When multiple Usecases must preserve the same invariant and allowing the implementation to diverge<br>
would cause an inconsistency such as double-booking, negative stock, or a duplicate charge,<br>
the processing may be shared as an Operation as an exception.

Operation is not a component for shortening Usecases or a general-purpose code-reuse mechanism.<br>
If you cannot name the invariant and the concrete consequence of violating it, keep the processing in Usecase.

Operation participates in the same Session as the calling Usecase and delegates every write to a Module.<br>
It never calls `begin`, `commit`, or `rollback`; the calling Usecase still owns the transaction boundary.

## HUMQ's Tradeoff

HUMQ does not automatically prevent a missing Module call or consistency rule.<br>
An omission in Usecase can still produce inconsistent state.<br>
Sharing through Operation does not structurally prevent a caller from omitting it.

Database constraints and Usecase tests are therefore used together.<br>
HUMQ provides no automatic guarantee of consistency.<br>
It provides a structure in which responsibility for consistency and the scope to inspect during change do not spread.

---

Previous: [Design Principles](03-design-principles.md) | Next: [Architecture and Design Pattern Comparison](05-comparison.md)
