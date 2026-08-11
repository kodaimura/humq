# FastAPI Example

This document shows a small FastAPI structure for applying HUMQ.

## Directory Structure

```text
app/
├── main.py
├── core/
│   ├── config.py
│   └── db.py
├── handlers/
│   └── accounts.py
├── usecases/
│   └── accounts/
│       └── signup.py
├── modules/
│   ├── account/
│   │   ├── model.py
│   │   └── module.py
│   └── account_role/
│       ├── model.py
│       └── module.py
├── queries/
│   └── account_overview.py
├── schemas/
│   └── account_dto.py
└── infrastructure/
    └── mailer.py
```

The important application flow is:

```text
Handler
  ↓
Usecase
  ├── AccountModule
  └── AccountRoleModule

Handler
  ↓
Usecase
  ↓
Query
```

## Handler Example

```python
# handlers/accounts.py

from fastapi import APIRouter, Depends

from app.core.db import get_session
from app.infrastructure.mailer import Mailer, get_mailer
from app.schemas.account_dto import AccountResponse, SignupRequest
from app.usecases.accounts.list_accounts import list_accounts
from app.usecases.accounts.signup import signup

router = APIRouter(prefix="/accounts", tags=["accounts"])


@router.post("/signup", response_model=AccountResponse)
def signup_account(
    request: SignupRequest,
    session = Depends(get_session),
    mailer: Mailer = Depends(get_mailer),
):
    account = signup(
        session=session,
        email=request.email,
        name=request.name,
        default_role="member",
        mailer=mailer,
    )
    return AccountResponse.model_validate(account)


@router.get("", response_model=list[AccountResponse])
def list_account_list(session = Depends(get_session)):
    return list_accounts(session=session)
```

Handler receives framework-specific request DTOs and converts them into plain Usecase arguments. The Usecase does not depend on FastAPI or Pydantic request objects.

## Usecase Example

```python
# usecases/accounts/signup.py

from app.modules.account import module as account_module
from app.modules.account_role import module as account_role_module


def signup(session, email: str, name: str, default_role: str, mailer):
    with session.begin():
        account = account_module.create(
            session=session,
            email=email,
            name=name,
        )

        account_role_module.create(
            session=session,
            account_id=account.id,
            role=default_role,
        )

    mailer.send_welcome(email=account.email, name=account.name)
    return account
```

Usecase owns the business flow and transaction boundary. It coordinates table-sized Modules.

The external side effect happens after the database transaction commits. For stronger consistency, use a pattern such as Outbox, but HUMQ itself does not require it.

```python
# usecases/accounts/list_accounts.py

from app.queries.account_overview import list_account_overview


def list_accounts(session):
    return list_account_overview(session)
```

Read Usecases can be thin. The important rule is that Handler still talks to Usecase, and Usecase decides which Query represents the read intent.

## Module Examples

```python
# modules/account/module.py

from app.modules.account.model import Account


def create(session, email: str, name: str):
    existing = session.query(Account).filter(Account.email == email).one_or_none()
    if existing:
        raise ValueError("account already exists")

    account = Account(email=email, name=name)
    session.add(account)
    session.flush()
    return account
```

```python
# modules/account_role/module.py

from app.modules.account_role.model import AccountRole


def create(session, account_id: int, role: str):
    account_role = AccountRole(account_id=account_id, role=role)
    session.add(account_role)
    session.flush()
    return account_role
```

Each Module is closed around exactly one table. `AccountModule` does not call `AccountRoleModule`, and `AccountRoleModule` does not call `AccountModule`.

Repository is not required. If ORM operations are readable enough inside Module, keep them there. Extract persistence code only as an internal helper inside Module when it grows too large.

## Query Example

```python
# queries/account_overview.py

from app.modules.account.model import Account
from app.modules.account_role.model import AccountRole


def list_account_overview(session):
    return (
        session.query(Account)
        .join(AccountRole, AccountRole.account_id == Account.id)
        .order_by(Account.id)
        .all()
    )
```

Query is read-only. It may read across multiple tables, but it does not write and does not own transaction boundaries.

## Placement Guide

When unsure during implementation, use these criteria:

| Concern | Place |
| --- | --- |
| API input and output | Handler |
| Business procedure | Usecase |
| Transaction boundary | Usecase |
| Single-table read or write | Module |
| Read intent | Usecase |
| Cross-table read | Query, called from Usecase |

---

Previous: [Comparison with Existing Architectures](05-comparison.md) | Next: [README](../README.md)
