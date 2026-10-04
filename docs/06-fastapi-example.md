# FastAPI Example

Using the structure and implementation style of [scaf-fast](https://github.com/kodaimura/scaf-fast) as a reference,<br>
this chapter shows how to apply HUMQ to FastAPI. Configuration and implementation details unnecessary to the explanation are omitted.

See [humq-sample](https://github.com/kodaimura/humq-sample)<br>
for a larger implementation example of HUMQ.

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
│   │   ├── _policies.py
│   │   └── _token_issuance.py
│   ├── organizations/
│   │   └── _authorization.py
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
        try:
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
        except Exception:
            self.db.rollback()
            raise
```

Usecase owns transaction boundaries such as `commit` and `rollback` on failure.<br>
Module may execute SQL through `flush`, but it does not finalize the successful transaction.

## Business Processing Within the Usecase Responsibility

Even when used by just one Usecase, processing that is meaningful to explain, verify, and change independently<br>
may be placed beside the Usecases in its owning business domain. `_policies.py` is the recommended<br>
starting name for extracted business rules; a more specific name such as `_token_issuance.py` is also valid.<br>
These two internal files illustrate an optional choice; the decisions and Module calls may instead stay in the Usecase.<br>
This is an example location; HUMQ leaves the placement of cross-domain processing to each project.<br>
Small local decisions may remain in Usecase. Internal processing does not require a class.
The leading `_` marks internal implementation; Handler does not call it directly, and it is not re-exported as a public Usecase.
The `_policies.py` name does not require pure processing or create a separate HUMQ layer.

The issuance conditions can be extracted as a pure decision using only supplied values.

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

The details of invalidating old tokens and creating a new one can also be extracted as one database-backed business process.<br>
The calling Usecase supplies a Module created with the same Session.

```python
# usecases/auth/_token_issuance.py

from app.modules.password_reset_token.module import PasswordResetTokenModule


def issue_password_reset_token(tokens: PasswordResetTokenModule, account_id: int):
    tokens.invalidate_active_tokens(account_id)
    return tokens.create(account_id)
```

Usecase shows the account lock, what happens when issuance is denied, the token issuance call,<br>
and the order of database commit and email delivery.

```python
# usecases/auth/forgot_password.py

from sqlalchemy.orm import Session

from app.error import AppError, ErrorCode
from app.mailer import Mailer
from app.modules.account.module import AccountModule
from app.modules.password_reset_token.module import PasswordResetTokenModule
from app.usecases.auth._policies import can_issue_password_reset
from app.usecases.auth._token_issuance import issue_password_reset_token


class ForgotPasswordUsecase:
    def __init__(self, db: Session, mailer: Mailer):
        self.db = db
        self.accounts = AccountModule(db)
        self.tokens = PasswordResetTokenModule(db)
        self.mailer = mailer

    def execute(self, email: str):
        try:
            account = self.accounts.get_by_email_for_update(email)
            if account is None:
                self.db.commit()
                return

            requests_today = self.tokens.count_created_today(account.id)
            if not can_issue_password_reset(
                account_is_active=account.is_active,
                requests_today=requests_today,
            ):
                raise AppError(code=ErrorCode.PASSWORD_RESET_NOT_ALLOWED)

            token = issue_password_reset_token(self.tokens, account.id)
            self.db.commit()
        except Exception:
            self.db.rollback()
            raise

        self.mailer.send_password_reset(account.email, token.value)
```

Internal processing reads through Module or Query and writes through Module.<br>
It does not retrieve or persist data directly with ORM or SQL, create a separate Session,<br>
or call `begin`, `commit`, or `rollback`. The pure eligibility decision does not need Session.<br>
Token issuance participates in the calling Usecase's Session; its change targets and Module calls are visible in `_token_issuance.py`.

This chapter omits the implementations of `PasswordResetTokenModule.count_created_today()`,<br>
`invalidate_active_tokens()`, and `create()`.<br>
Each Module uses the supplied Session for operations on its target table and does not `commit`.

The request limit and active-token consistency during concurrent issuance assume a database that honors row locks<br>
and that every issuance path locks the same account row. The lock is acquired before counting requests<br>
and held until the Usecase calls `commit` or `rollback`.<br>
If the database does not support row locks, Module needs equivalent concurrency control such as<br>
conditional updates or database constraints.

Here, Usecase calls `rollback` if database processing fails and sends email after `commit`.<br>
A delivery failure propagates to the caller, but does not undo the committed database change.<br>
If retries or delivery guarantees are needed, the Usecase's failure policy can introduce Outbox or a similar pattern.

Test the pure eligibility decision with input and output unit tests. For database-backed issuance,<br>
use database tests to check that invalidating old tokens and creating the new one remain consistent in the same Session,<br>
including mid-process failure and `rollback` by the calling Usecase.

## Module

Module receives Session and provides reads, writes, and standard operations for one table by default.

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

    def get_by_email_for_update(self, email: str) -> Account | None:
        stmt = select(Account).where(Account.email == email).with_for_update()
        return self.db.scalars(stmt).first()

    def get_all(self) -> list[Account]:
        stmt = select(Account).order_by(Account.id)
        return list(self.db.scalars(stmt).all())
```

AccountModule reads and writes only the `account` table.<br>
`get_by_email_for_update()` reads with a row lock needed for the password reset flow.<br>
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

## Multiple Modules and External I/O

In a password-reset flow, Usecase handles two Modules and a mailer in this order:

```text
ForgotPasswordUsecase
  AccountModule.get_by_email_for_update()
  PasswordResetTokenModule.count_created_today()
  can_issue_password_reset()
  issue_password_reset_token() → PasswordResetTokenModule invalidation and creation
  db.commit()
  Mailer.send_password_reset()
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
