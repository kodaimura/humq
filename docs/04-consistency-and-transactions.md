# Consistency and Transactions

In HUMQ, consistency and transaction boundaries are Usecase responsibilities. The important question is not only where `commit` occurs, but what succeeds atomically and what is handled as a separate failure. That answer must remain visible from Usecase.

## Basic Rules

- Only Usecase may own application transaction boundaries.
- By default, group database writes that must be atomic into one transaction.
- Module does not call `commit` or `rollback`.
- Even if Repository is extracted, it does not call `commit` or `rollback`.
- Make consistency across multiple Modules explicit in Usecase.
- Query is read-only and does not own transaction boundaries.
- Do not perform external side effects in the middle of a database transaction.

## Why Transactions Belong in Usecase

A transaction is an operation that temporarily binds multiple forms of order.

Module is an order closed around exactly one table. If Module manages transactions, it becomes difficult for the surrounding Usecase to control consistency across the whole operation.

When Usecase owns transactions, the following become explicit:

- Which operations form one business unit.
- Which Modules are combined.
- Which range succeeds or fails as one unit.
- Which consistency rules are intentionally protected.

HUMQ does not force a one-to-one mapping of `Usecase = Transaction`. A read-only Usecase may not need an explicit transaction. A long-running business operation involving external systems may require several database transactions and compensating actions.

Even then, each boundary, side effect, and post-failure policy remains traceable from Usecase. When using a transaction decorator, its declaration must make clear which Usecase and which scope it wraps.

## Difference from Aggregate-Centered Designs

DDD Aggregates are a strong way to protect invariants within an Aggregate boundary. An Application Service may also coordinate multiple Aggregates, so DDD does not inherently hide business flows.

The difference is the default boundary.

| Viewpoint | Aggregate-centered design | HUMQ |
| --- | --- | --- |
| Local consistency | Expressed inside Aggregate | Expressed in a one-table Module |
| Cross-boundary consistency | Coordinated in an application layer or similar place | Made explicit in Usecase |
| Lower-level boundary | Chosen through Domain Model design | Mechanically fixed at one table |
| Main tradeoff | Strong invariants, harder boundary design | High traceability, less automatic protection |

HUMQ prioritizes traceable operation targets and consistency over automatic protection.

## Implementation Sketch

```python
# usecases/accounts/assign_role.py

def assign_role(session, account_id: int, role_id: int) -> None:
    with session.begin():
        account = account_module.get(session, account_id)
        role = role_module.get(session, role_id)

        account_role_module.register(
            session=session,
            account_id=account.id,
            role_id=role.id,
        )

        audit_log_module.record(
            session=session,
            action="assign_role",
            actor_id=account.id,
        )
```

This Usecase makes the operations on Account, Role, AccountRole, and AuditLog—and their atomic scope—visible in one place.

## External Side Effects

Do not perform external side effects, such as sending email or calling a payment API, in the middle of a database transaction.

```text
database transaction
↓
commit
↓
external side effect
```

This avoids an inconsistency where the email succeeds but the database commit fails and rolls back. However, the email can still fail after the commit. Reordering the operations cannot make a database and an external system atomic.

Usecase selects a failure policy according to the required guarantee.

| Situation | Policy |
| --- | --- |
| Notification failure is acceptable | Send after commit and record failures |
| Delivery must be retried | Design idempotent delivery and retries |
| A committed delivery request must not be lost | Use the Outbox Pattern |
| External state changes, as with payments | Design idempotency, a Saga, or compensation |

With Outbox, Usecase writes to an outbox table through a normal Module in the same transaction, and a separate worker delivers it. HUMQ does not require Outbox. It requires the transaction boundary, side effect, and failure guarantee to remain traceable from Usecase.

Mailers, payment gateways, and external API clients are not Modules, because they do not correspond to one table.

## Examples to Avoid

### Module Commits

```python
# modules/account/module.py

def create(session, name: str):
    account = Account(name=name)
    session.add(account)
    session.commit()
    return account
```

With this design, Usecase cannot combine multiple Modules into one transaction because Module closes the boundary on its own.

### Persistence Helper Owns Business Consistency

```python
# modules/account/repository.py

def create_account_and_role(session, name: str, role_id: int):
    ...
```

Even when Repository is extracted, it is an internal helper inside Module. Placing business combinations there breaks the responsibilities of Usecase and Module.

## HUMQ's Tradeoff on Consistency

HUMQ is not a design that automatically guarantees all consistency.

Instead, it moves consistency to a visible place. By reading the Usecase, you can see which forms of order are connected, in what order they run, and which range is allowed to fail together.

This trades automatic, implicit protection for explicit consistency and traceability. It does not ignore consistency; it clarifies where designers must handle it responsibly.

## Query and Transactions

Query is read-only. Query does not decide transaction boundaries and does not call `commit` or `rollback`.

This does not mean the underlying ORM or database connection never uses a transaction internally. It only means Query does not own that boundary as a HUMQ responsibility.

---

Previous: [Design Principles](03-design-principles.md) | Next: [Architecture and Design Pattern Comparison](05-comparison.md)
