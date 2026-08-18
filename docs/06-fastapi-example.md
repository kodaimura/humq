# FastAPI Example

Using the structure and implementation style of [scaf-fast](https://github.com/kodaimura/scaf-fast) as a reference,<br>
this chapter shows how to apply HUMQ to FastAPI. Configuration and implementation details unnecessary to the explanation are omitted.

HUMQ does not require class-based Usecases or Modules.<br>
This example follows scaf-fast by injecting the database Session and dependencies through constructors.

## Directory Structure

```text
app/
├── main.py
├── router.py
├── database.py
├── config.py
├── crypto.py
├── error.py
├── mailer.py
├── response.py
├── handlers/
│   ├── accounts.py
│   ├── auth.py
│   └── dto/
│       └── accounts.py
├── usecases/
│   ├── accounts/
│   │   ├── create.py
│   │   └── list.py
│   ├── auth/
│   │   ├── forgot_password.py
│   │   └── _policies.py
│   ├── organizations/
│   │   └── _operations.py
│   ├── orders/
│   │   └── confirm.py
├── modules/
│   ├── account/
│   │   ├── model.py
│   │   └── module.py
│   ├── organization/
│   │   ├── model.py
│   │   └── module.py
│   ├── organization_member/
│   │   ├── model.py
│   │   └── module.py
│   ├── order/
│   │   ├── model.py
│   │   └── module.py
│   └── password_reset_token/
│       ├── model.py
│       └── module.py
└── queries/
    └── account_security.py
```

Whether top-level directories use singular or plural names, such as `handler` or `handlers`,<br>
is not a HUMQ responsibility boundary. This example follows the notation used in [Layer Rules](02-layer-rules.md).

Placement of database connections, configuration, email, and similar code is outside HUMQ's scope.<br>
A project may instead group these under `core/`, `infrastructure/`, `clients/`, or another structure.

Handler-called Usecases correspond to the Handler's resource structure.<br>

## Database Session

A FastAPI dependency creates one Session per request and passes it from Handler to Usecase.

```python
# database.py

from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, sessionmaker

from app.config import config


class Base(DeclarativeBase):
    pass


engine = create_engine(config.DATABASE_URL, pool_pre_ping=True)
SessionLocal = sessionmaker(
    bind=engine,
    autoflush=False,
    expire_on_commit=False,
)


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

Receiving Session does not itself mean that Handler performs database operations.<br>
Handler only passes Session to Usecase; it does not query, update, or `commit`.

Because Handler converts an ORM model returned by Usecase after `commit` into a DTO,<br>
this example sets `expire_on_commit=False`. This prevents DTO conversion<br>
from triggering an implicit refresh query in Handler.

Session and ORM models may be passed among Handler, Usecase, and Module.<br>
HUMQ does not require conversion into persistence-independent Entities.

## Handler

Request and response DTOs live under Handler, keeping Pydantic and FastAPI dependencies there.

```python
# handlers/dto/accounts.py

from pydantic import BaseModel, EmailStr


class PostAccountRequest(BaseModel):
    login_id: str | None = None
    email: EmailStr | None = None
    password: str
    first_name: str
    last_name: str


class AccountResponse(BaseModel):
    id: int
    login_id: str
    email: EmailStr | None
    first_name: str
    last_name: str

    model_config = {"from_attributes": True}
```

```python
# handlers/accounts.py

from fastapi import APIRouter, Depends, Response
from sqlalchemy.orm import Session

from app.database import get_db
from app.response import ApiResponse
from app.handlers.dto.accounts import (
    AccountResponse,
    PostAccountRequest,
)
from app.usecases.accounts.create import (
    CreateAccountInput,
    CreateAccountUsecase,
)

router = APIRouter()


@router.post("/accounts")
def post_account(
    request: PostAccountRequest,
    response: Response,
    db: Session = Depends(get_db),
):
    usecase = CreateAccountUsecase(db)
    account = usecase.execute(
        CreateAccountInput(
            login_id=request.login_id,
            email=request.email,
            password=request.password,
            first_name=request.first_name,
            last_name=request.last_name,
        )
    )

    data = AccountResponse.model_validate(account)
    return ApiResponse.created(data=data, response=response)
```

Handler converts a request DTO into Usecase Input and converts the Usecase result into a response DTO.<br>
Business duplicate checks, password hashing, and persistence do not belong in Handler.

The Input and response DTOs are choices that make types explicit in this FastAPI example,<br>
not required HUMQ elements. A project may pass dictionaries or ORM models directly when appropriate.

## Usecase

This example defines Usecase Input with framework-independent types.<br>
Usecase combines the required Modules and owns business branches and transaction boundaries.

```python
# usecases/accounts/create.py

from dataclasses import dataclass
from sqlalchemy.orm import Session

from app.crypto import hash_password
from app.error import AppError, ErrorCode
from app.modules.account.module import AccountModule


@dataclass(frozen=True)
class CreateAccountInput:
    login_id: str | None
    email: str | None
    password: str
    first_name: str
    last_name: str


class CreateAccountUsecase:
    def __init__(self, db: Session):
        self.db = db
        self.account_module = AccountModule(db)

    def execute(self, input: CreateAccountInput):
        login_id = input.login_id or input.email
        if login_id is None:
            raise AppError(code=ErrorCode.LOGIN_ID_REQUIRED)

        if self.account_module.get_by_login_id(login_id):
            raise AppError(code=ErrorCode.LOGIN_ID_ALREADY_EXISTS)

        account = self.account_module.create(
            login_id=login_id,
            email=input.email,
            password_hash=hash_password(input.password),
            first_name=input.first_name,
            last_name=input.last_name,
        )

        self.db.commit()
        return account
```

Only Usecase calls `commit`. Module may execute SQL through `flush`,<br>
but it does not finalize the successful transaction.

## Policy

Policy does not retrieve required values from the database.<br>
It makes a business decision or calculation only from values supplied by Usecase.

```python
# usecases/auth/_policies.py

MAX_PASSWORD_RESET_REQUESTS_PER_DAY = 3


def can_issue_password_reset(
    *,
    account_is_active: bool,
    requests_today: int,
) -> bool:
    return (
        account_is_active
        and requests_today < MAX_PASSWORD_RESET_REQUESTS_PER_DAY
    )
```

Usecase retrieves values through Modules and changes state based on the Policy result.

```python
# usecases/auth/forgot_password.py

class ForgotPasswordUsecase:
    def execute(self, email: str):
        account = self.accounts.get_by_email(email)
        requests_today = self.tokens.count_created_today(account.id)

        if not can_issue_password_reset(
            account_is_active=account.is_active,
            requests_today=requests_today,
        ):
            raise PasswordResetNotAllowed()

        self.tokens.invalidate_active_tokens(account.id)
        token = self.tokens.create(account.id)
        self.db.commit()
        self.mailer.send_password_reset(account.email, token.value)
```

Database access, Module calls, `commit`, and email delivery do not move into Policy.<br>
The Policy call and resulting branch remain visible in Usecase.

## Operation

Place database-backed business processing shared by multiple Usecases in the owning domain's `_operations.py` first.<br>
See [Adoption Limits and Evolution](07-adoption-limits-and-evolution.md)<br>
for splitting rules when it becomes difficult to read.<br>
The following Operation retrieves an organization and member through Modules and verifies a shared authorization condition.

```python
# usecases/organizations/_operations.py

class RequireOrganizationRoleOperation:
    def __init__(self, db: Session):
        self.organizations = OrganizationModule(db)
        self.members = OrganizationMemberModule(db)

    def run(
        self,
        *,
        organization_id: int,
        account_id: int,
        allowed_roles: set[str],
    ) -> None:
        organization = self.organizations.get_by_id(organization_id)
        if not organization:
            raise AppError(code=ErrorCode.ORGANIZATION_NOT_FOUND)

        member = self.members.get(
            organization_id=organization_id,
            account_id=account_id,
        )
        if not member or member.role not in allowed_roles:
            raise AppError(code=ErrorCode.ACCESS_DENIED)
```

Usecase calls Operation explicitly within the primary flow and finalizes the transaction.

```python
# usecases/orders/confirm.py

from app.usecases.organizations._operations import (
    RequireOrganizationRoleOperation,
)


class ConfirmOrderUsecase:
    def __init__(self, db: Session):
        self.db = db
        self.orders = OrderModule(db)
        self.require_role = RequireOrganizationRoleOperation(db)

    def execute(self, input):
        order = self.orders.get_for_update(input.order_id)
        self.require_role.run(
            organization_id=order.seller_organization_id,
            account_id=input.account_id,
            allowed_roles={"ADMIN", "SALES"},
        )
        self.orders.confirm(order)
        self.db.commit()
        return order
```

Operation uses the same Session as its caller and routes every write through a Module.<br>
Transaction finalization remains in Usecase.

## Module

Module receives Session and provides reads, writes, and standard operations for exactly one table.

```python
# modules/account/module.py

from sqlalchemy import select
from sqlalchemy.orm import Session

from app.modules.account.model import Account


class AccountModule:
    def __init__(self, db: Session):
        self.db = db

    def create(
        self,
        *,
        login_id: str,
        email: str | None,
        password_hash: str,
        first_name: str,
        last_name: str,
    ) -> Account:
        account = Account(
            login_id=login_id,
            email=email,
            password_hash=password_hash,
            first_name=first_name,
            last_name=last_name,
        )
        self.db.add(account)
        self.db.flush()
        self.db.refresh(account)
        return account

    def get_by_login_id(self, login_id: str) -> Account | None:
        stmt = select(Account).where(Account.login_id == login_id)
        return self.db.scalars(stmt).first()

    def get_all(self) -> list[Account]:
        stmt = select(Account).order_by(Account.id)
        return list(self.db.scalars(stmt).all())
```

AccountModule reads and writes only the `account` table.<br>
It neither calls another Module nor converts results into response DTOs or calls `commit`.<br>
Database operations use SQLAlchemy 2.x `select()` and `scalars()` instead of `Session.query()`.

Only when writing its target table requires it may Module read another table as an exception.<br>
Even then, it does not change the other table's state.

## Build Cross-Table Read Models in Query

Standard table operations such as lookup by primary key, existence checks, and standard lists belong in Module.<br>
Reads that shape data from multiple tables for a screen, search, report, or other purpose belong in Query.<br>
Only as an exception, a complex search, specialized SQL statement, or other read that does not fit<br>
Module's standard operations may belong in Query when it reads only one table.

The following Query reads across `account` and `password_reset_token`<br>
and returns a read model of account security status rather than an ORM model.

```python
# queries/account_security.py

from dataclasses import dataclass
from datetime import datetime

from sqlalchemy import func, select
from sqlalchemy.orm import Session

from app.modules.account.model import Account
from app.modules.password_reset_token.model import PasswordResetToken


@dataclass(frozen=True)
class AccountSecurityOverview:
    account_id: int
    login_id: str
    email: str | None
    reset_request_count: int
    last_reset_requested_at: datetime | None


class AccountSecurityQuery:
    def __init__(self, db: Session):
        self.db = db

    def list_overviews(self) -> list[AccountSecurityOverview]:
        stmt = (
            select(
                Account.id.label("account_id"),
                Account.login_id,
                Account.email,
                func.count(PasswordResetToken.id).label("reset_request_count"),
                func.max(PasswordResetToken.created_at).label(
                    "last_reset_requested_at"
                ),
            )
            .outerjoin(
                PasswordResetToken,
                PasswordResetToken.account_id == Account.id,
            )
            .group_by(Account.id, Account.login_id, Account.email)
            .order_by(Account.id)
        )
        return [
            AccountSecurityOverview(**row._mapping)
            for row in self.db.execute(stmt)
        ]
```

Handler does not call Query directly, even for a read-only operation.

```python
# usecases/accounts/list.py

from sqlalchemy.orm import Session

from app.queries.account_security import AccountSecurityQuery


class ListAccountSecurityUsecase:
    def __init__(self, db: Session):
        self.query = AccountSecurityQuery(db)

    def execute(self):
        return self.query.list_overviews()
```

Query represents how data is read; Usecase represents the operation the application provides.<br>
The thinness of this Usecase is not a problem.

## Multiple Modules and External Operations

In a password-reset flow, Usecase handles two Modules and a mailer in this order:

```text
ForgotPasswordUsecase
  AccountModule.get_by_email()
  PasswordResetTokenModule.invalidate_active_tokens()
  PasswordResetTokenModule.create()
  db.commit()
  Mailer.send()
```

Because email is sent after the database `commit`, a delivery failure does not roll back the database change.<br>
Add Outbox or another pattern when retries or delivery guarantees are required.

## Errors and Responses

Usecase reports failures through application ErrorCode values rather than HTTP status codes.<br>
A FastAPI exception handler converts AppError into an HTTP response.

```python
# usecases/accounts/create.py
raise AppError(code=ErrorCode.LOGIN_ID_ALREADY_EXISTS)


# main.py
@app.exception_handler(AppError)
async def handle_app_error(request: Request, exc: AppError):
    return ApiResponse.error(
        data={"code": exc.code, "details": exc.details},
        status_code=exc.status_code,
    )
```

This keeps Usecase independent of FastAPI's `HTTPException` and response format.

For a FastAPI implementation reference including application startup, migrations, authentication, and tests,<br>
see [scaf-fast](https://github.com/kodaimura/scaf-fast).

---

Previous: [Architecture and Design Pattern Comparison](05-comparison.md) | Next: [Adoption Limits and Evolution](07-adoption-limits-and-evolution.md)
