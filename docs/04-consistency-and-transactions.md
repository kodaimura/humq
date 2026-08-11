# Consistency and Transactions

In HUMQ, consistency and transactions are Usecase responsibilities.

## Basic Rules

- Start and end transactions in Usecase.
- Module does not call `commit` or `rollback`.
- Even if Repository is extracted, it does not call `commit` or `rollback`.
- Make consistency across multiple Modules explicit in Usecase.
- Query is read-only and does not own transaction boundaries.

## Why Transactions Belong in Usecase

A transaction is an operation that temporarily binds multiple forms of order.

Module is an order closed around exactly one table. If Module manages transactions, it becomes difficult for the surrounding Usecase to control consistency across the whole operation.

When Usecase owns transactions, the following become explicit:

- Which operations form one business unit.
- Which Modules are combined.
- Which range succeeds or fails as one unit.
- Which consistency rules are intentionally protected.

## Difference from DDD

In DDD, the Aggregate Root protects consistency. In HUMQ, Usecase makes consistency explicit.

| Viewpoint | DDD | HUMQ |
| --- | --- | --- |
| Consistency responsibility | Aggregate | Usecase |
| Transaction boundary | Aggregate unit | Usecase unit |
| Flexibility | Strongly depends on boundaries | Easy to coordinate table-sized Modules |
| Visibility | Hidden inside | Visible in code |
| Reusability | Tends to stay inside Aggregate | Easier at Module level |

HUMQ prioritizes explicit order over automatic consistency.

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

This Usecase binds the order of Account, Role, AccountRole, and AuditLog into one business intent.

## External Side Effects

Do not perform external side effects, such as sending email or calling a payment API, in the middle of a database transaction.

```text
database transaction
↓
commit
↓
external side effect
```

This avoids a simple inconsistency where the email succeeds but the database commit fails and rolls back. If stronger consistency is required, use a pattern such as Outbox. HUMQ itself does not require Outbox; it only requires that transaction boundaries and side effects are visible in Usecase.

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

This is a choice to trade implicit consistency for explicit order. It does not ignore consistency; it clarifies where designers must handle it responsibly.

## Query and Transactions

Query is read-only. Query does not decide transaction boundaries and does not call `commit` or `rollback`.

This does not mean the underlying ORM or database connection never uses a transaction internally. It only means Query does not own that boundary as a HUMQ responsibility.

---

Previous: [Design Principles](03-design-principles.md) | Next: [Comparison with Existing Architectures](05-comparison.md)
