# FastAPI Example

This document shows an example structure for applying HUMQ with FastAPI.

## Directory Structure

```text
app/
├── main.py
├── core/
│   ├── config.py
│   ├── db.py
│   ├── exceptions.py
│   ├── jwt.py
│   ├── logger.py
│   └── mail.py
├── handlers/
│   ├── accounts.py
│   ├── projects.py
│   ├── users.py
│   └── health.py
├── usecases/
│   ├── accounts/
│   │   ├── signup.py
│   │   ├── login.py
│   │   ├── update_profile.py
│   │   └── delete.py
│   ├── projects/
│   │   ├── create.py
│   │   ├── update.py
│   │   └── archive.py
│   └── users/
│       └── deactivate.py
├── modules/
│   ├── account/
│   │   ├── model.py
│   │   └── module.py
│   ├── project/
│   │   ├── model.py
│   │   └── module.py
│   └── mail/
│       └── module.py
├── queries/
│   ├── account_orders.py
│   ├── project_progress.py
│   ├── sales_report.py
│   └── activity_overview.py
└── schemas/
    ├── account_dto.py
    ├── project_dto.py
    └── __init__.py
```

## Handler Example

```python
# handlers/accounts.py

from fastapi import APIRouter, Depends

from app.core.db import get_session
from app.schemas.account_dto import SignupRequest, AccountResponse
from app.usecases.accounts.signup import signup

router = APIRouter(prefix="/accounts", tags=["accounts"])


@router.post("/signup", response_model=AccountResponse)
def signup_account(
    request: SignupRequest,
    session = Depends(get_session),
):
    account = signup(session=session, request=request)
    return AccountResponse.model_validate(account)
```

Handler only receives the request, calls Usecase, and transforms the result into a response.

## Usecase Example

```python
# usecases/accounts/signup.py

from app.modules.account import module as account_module
from app.modules.mail import module as mail_module


def signup(session, request):
    with session.begin():
        account = account_module.create(
            session=session,
            email=request.email,
            name=request.name,
        )

        mail_module.send_welcome(
            email=account.email,
            name=account.name,
        )

        return account
```

Usecase owns the business flow and transaction boundary. It is also responsible for combining multiple Modules.

## Module Example

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

Module provides operations closed around the Account subject. It does not call other Modules and does not manage transactions.

Repository is not required. If ORM operations are readable enough inside Module, keep them there. Extract persistence code only as an internal helper inside Module when it grows too large.

## Query Example

```python
# queries/account_orders.py

def list_account_orders(session, account_id: int):
    return (
        session.query(...)
        .join(...)
        .filter(...)
        .order_by(...)
        .all()
    )
```

Query is read-only. It handles cross-cutting observation such as joins and aggregations.

## Placement Guide

When unsure during implementation, use these criteria:

| Concern | Place |
| --- | --- |
| API input and output | Handler |
| Business procedure | Usecase |
| Transaction boundary | Usecase |
| Operation closed around one table | Module |
| Simple database access | Module |
| Read spanning multiple tables | Query |

---

Previous: [Comparison with Existing Architectures](05-comparison.md) | Next: [README](../README.md)
