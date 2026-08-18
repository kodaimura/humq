# FastAPI構成例

この章では、[scaf-fast](https://github.com/kodaimura/scaf-fast)の構成と実装方法を参考に、<br>
FastAPIへHUMQを適用する例を示します。説明に不要な設定や実装は省略しています。

UsecaseやModuleをクラスにすることはHUMQの必須規則ではありません。<br>
ここでは、DBセッションと依存関係をコンストラクタから渡すscaf-fastの形式を採用します。

## ディレクトリ構成

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

`handler`と`handlers`のような最上位ディレクトリの単数・複数は、HUMQの責務境界ではありません。<br>
この例では[層と責務のルール](02-layer-rules.ja.md)の表記に合わせています。<br>
DB接続、設定、メールなどの配置はHUMQの規定外であり、<br>
`core/`、`infrastructure/`、`clients/`などへ分けることもできます。

Handlerから呼ばれるUsecaseは、Handlerのリソース構成に対応させます。<br>

## DBセッション

FastAPIのDependencyでリクエストごとのSessionを作り、HandlerからUsecaseへ渡します。

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

Sessionを受け取ること自体はHandlerによるDB操作ではありません。<br>
HandlerはSessionをUsecaseへ渡すだけで、検索、更新、`commit`は行いません。

この例では、Usecaseが`commit`後に返したORMモデルをHandlerがDTOへ変換するため、<br>
`expire_on_commit=False`を指定します。これにより、DTOへの変換が、<br>
Handlerからの暗黙的な再検索になることを防ぎます。

SessionやORMモデルをHandler、Usecase、Moduleの間で受け渡すことは許容されます。<br>
HUMQは、永続化方式から独立したEntityへの変換を必須としません。

## Handler

リクエストとレスポンスのDTOはHandler配下に置き、PydanticとFastAPIへの依存をここへ閉じます。

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

HandlerはリクエストDTOをUsecaseのInputへ変換し、Usecaseの結果をレスポンスDTOへ変換します。<br>
業務上の重複確認、パスワードのハッシュ化、DBへの保存はHandlerに置きません。

このInputとResponse DTOはFastAPIで型を明確にするための選択であり、<br>
HUMQの必須要素ではありません。不要であれば辞書やORMモデルを直接受け渡せます。

## Usecase

この例では、UsecaseのInputをフレームワークに依存しない型として定義します。<br>
Usecaseは必要なModuleを組み合わせ、業務上の分岐とトランザクション境界を持ちます。

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

`commit`するのはUsecaseです。Moduleは`flush`によってSQLを実行できますが、<br>
トランザクションの成功を確定しません。

## Policy

Policyは、DBから必要な値を取得する処理ではなく、<br>
Usecaseから渡された値だけで業務上の判断または計算を行います。

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

UsecaseがModuleから値を取得し、Policyの結果に基づいて状態を変更します。

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

DBアクセス、Module呼び出し、`commit`、メール送信はPolicyへ移しません。<br>
Policyの呼び出しと、その結果による分岐はUsecaseから確認できます。

## Operation

複数Usecaseで共有するDB依存の業務処理は、所有ドメインの`_operations.py`へ置きます。<br>
次のOperationは、組織とメンバーをModuleから取得し、共通の認可条件を検証します。

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

UsecaseはOperationを主要なフローの中から明示的に呼び、トランザクションを確定します。

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

Operationは呼び出し元と同じSessionを使い、書き込みは各Moduleを通します。<br>
トランザクションの確定はUsecaseに残します。

## Module

ModuleはSessionを受け取り、正確に1テーブルの読み書きと標準操作を提供します。

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

AccountModuleは`account`テーブルだけを読み書きします。<br>
別のModuleを呼ばず、`commit`やResponse DTOへの変換も行いません。<br>
DB操作には、`Session.query()`ではなくSQLAlchemy 2.xの`select()`と`scalars()`を使います。

所有するテーブルへの書き込みに必要な場合だけ、例外として別テーブルを参照できます。<br>
その場合も、別テーブルの状態は変更しません。

## Queryで横断的な読み取りモデルを作る

主キー取得、存在確認、標準一覧など、テーブルの標準操作はModuleに置きます。<br>
複数テーブルから画面、検索、帳票などに必要な形を作る読み取りはQueryに置きます。<br>
複雑な検索や特殊なSQLなど、Moduleの標準操作に収まらない読み取りに限り、<br>
1テーブルだけを参照する場合でも例外としてQueryに置けます。

次のQueryは、`account`と`password_reset_token`を横断し、<br>
ORMモデルではなくアカウントのセキュリティ状況を表す読み取りモデルを返します。

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

読み取り専用でも、HandlerからQueryを直接呼びません。

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

Queryはデータの読み方、Usecaseはアプリケーションが提供する操作を表します。<br>
このUsecaseが薄いことは問題ではありません。

## 複数Moduleと外部処理

パスワードリセットでは、Usecaseが2つのModuleとMailerを次の順序で扱います。

```text
ForgotPasswordUsecase
  AccountModule.get_by_email()
  PasswordResetTokenModule.invalidate_active_tokens()
  PasswordResetTokenModule.create()
  db.commit()
  Mailer.send()
```

DB更新を`commit`した後でメールを送るため、メール送信が失敗してもDB更新は戻りません。<br>
再送や配信保証が必要な場合は、Outboxなどを追加します。

## エラーとレスポンス

UsecaseはHTTPステータスコードではなく、アプリケーションのErrorCodeを使って失敗を表します。<br>
FastAPIのexception handlerがAppErrorをHTTPレスポンスへ変換します。

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

これにより、UsecaseはFastAPIの`HTTPException`やレスポンス形式に依存しません。

起動設定、マイグレーション、認証、テストを含むFastAPI実装の参考として、<br>
[scaf-fast](https://github.com/kodaimura/scaf-fast)を参照してください。

---

前へ: [既存アーキテクチャ・設計パターンとの比較](05-comparison.ja.md) | 次へ: [適用限界と発展](07-adoption-limits-and-evolution.ja.md)
