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
├── core/
│   ├── database.py
│   ├── config.py
│   ├── crypto.py
│   ├── error.py
│   ├── mailer.py
│   └── response.py
├── handlers/
│   ├── accounts.py
│   └── dto/
│       └── accounts.py
├── usecases/
│   ├── accounts/
│   │   ├── create.py
│   │   └── list.py
│   └── auth/
│       └── forgot_password.py
├── modules/
│   ├── account/
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

## Database Session

A FastAPI dependency creates one Session per request and passes it from Handler to Usecase.

```python
# core/database.py

from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, sessionmaker

from app.core.config import config


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

from app.core.database import get_db
from app.core.response import ApiResponse
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

## Usecase

Usecase Input is defined with framework-independent types.<br>
Usecase combines the required Modules and owns business branches and transaction boundaries.

```python
# usecases/accounts/create.py

from dataclasses import dataclass
from sqlalchemy.orm import Session

from app.core.crypto import hash_password
from app.core.error import AppError, ErrorCode
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

## Module

Module receives Session and provides ORM operations closed around one table.

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

## Build Read Models in Query

A list that reads only one table belongs in Module.<br>
Use Query when multiple tables are read to produce data shaped for a screen or business context.

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

In scaf-fast's password-reset flow, Usecase handles two Modules and a mailer in this order:

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

Previous: [Architecture and Design Pattern Comparison](05-comparison.md) | Next: [README](../README.md)
