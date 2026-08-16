# FastAPI Example

This chapter uses the structure and implementation style of<br>
[scaf-fast](https://github.com/kodaimura/scaf-fast) as a reference for applying HUMQ to FastAPI.<br>
Configuration and implementation details not needed for the explanation are omitted.

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
```

Whether top-level directories use singular or plural names, such as `handler` or `handlers`,<br>
is not a HUMQ responsibility boundary. This example follows the notation used in [Layer Rules](02-layer-rules.md).

## Database Session

A FastAPI dependency creates one Session per request and passes it from Handler to Usecase.

```python
# core/database.py

from sqlalchemy import create_engine
from sqlalchemy.orm import declarative_base, sessionmaker

from app.core.config import config

Base = declarative_base()
engine = create_engine(config.DATABASE_URL, pool_pre_ping=True, future=True)
SessionLocal = sessionmaker(
    bind=engine,
    autoflush=False,
    autocommit=False,
    future=True,
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
It neither calls another Module nor converts results into response DTOs or calls `commit`.

## Create Query Only When Needed

The account list in scaf-fast reads one table, so it uses AccountModule rather than Query.

```python
# usecases/accounts/list.py

from sqlalchemy.orm import Session

from app.modules.account.module import AccountModule


class ListAccountsUsecase:
    def __init__(self, db: Session):
        self.account_module = AccountModule(db)

    def execute(self):
        return self.account_module.get_all()
```

The reason it belongs in Module is that it reads one table, not merely that it has no join.<br>
Create a read-only Query when a list or report first needs to span multiple tables.<br>
Query does not call `commit` and is called from Usecase.

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

For a complete implementation including application startup, migrations, authentication, and tests,<br>
see [scaf-fast](https://github.com/kodaimura/scaf-fast).

---

Previous: [Architecture and Design Pattern Comparison](05-comparison.md) | Next: [README](../README.md)
