# FastAPI構成例

この章では、[scaf-fast](https://github.com/kodaimura/scaf-fast)の構成と実装方法を参考に、<br>
FastAPIへHUMQを適用する例を示します。説明に不要な設定や実装は省略しています。

HUMQを適用した、より大きな実装例は、<br>
[humq-sample](https://github.com/kodaimura/humq-sample)で確認できます。

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

`commit`や失敗時の`rollback`など、トランザクション境界を所有するのはUsecaseです。<br>
Moduleは`flush`によってSQLを実行できますが、トランザクションの成功を確定しません。

## Usecaseの責務に属する業務処理

この例では業務処理を別ファイルに分けますが、HUMQは分離を要求しません。<br>
使用箇所が1つでも、独立して説明・検証・変更する意味がある処理は、<br>
この例のように、所有する業務領域のUsecaseと同じディレクトリに分離できます。<br>
業務ルールを切り出す場合は`_policies.py`を初期案として推奨しますが、<br>
`_token_issuance.py`のような具体的な名前も使えます。<br>
横断的なルールなどのフォルダ構成は、利用側が選べます。<br>
小さな局所的判断はUsecase内に残せます。内部処理のクラス化は必須ではありません。<br>
この例のファイル名では、先頭の`_`で内部実装を示します。<br>
配置や命名によらず、Handlerから直接呼ばず、公開Usecaseとして再exportしません。<br>
`_policies.py`は純粋な処理だけを置く分類でも、HUMQの別の層でもありません。

次の発行可否判断は、渡された値だけを使う純粋な処理です。

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

一方、古いトークンの無効化と新しいトークンの発行は、DBを使う一つの業務処理として分離できます。<br>
呼び出し元Usecaseが同じSessionで作成したModuleを渡します。

```python
# usecases/auth/_token_issuance.py

from app.modules.password_reset_token.module import PasswordResetTokenModule


def issue_password_reset_token(tokens: PasswordResetTokenModule, account_id: int):
    tokens.invalidate_active_tokens(account_id)
    return tokens.create(account_id)
```

Usecaseは対象アカウントのロック、判断の結果による分岐、発行処理の呼び出し、<br>
トランザクション境界、メール送信の順序を示します。

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

内部処理はModule・Queryを通して読み取り、Moduleを通して書き込みます。<br>
ORMやSQLで直接データを取得・永続化せず、独自のSession、`begin`、`commit`、`rollback`も持ちません。<br>
純粋な発行可否判断にSessionを渡す必要はありません。トークン発行処理はUsecaseと同じSessionに参加し、<br>
変更対象と呼び出すModuleの操作は`_token_issuance.py`から確認できます。

この章では`PasswordResetTokenModule`の`count_created_today()`、<br>
`invalidate_active_tokens()`、`create()`の実装を省略しています。<br>
各Moduleは渡されたSessionを使い、対象テーブルの操作を提供し、`commit`しません。

同時発行時の件数上限と有効トークンの整合性は、対象DBで行ロックが機能し、<br>
すべての発行経路が同じアカウント行をロックすることを前提にしています。<br>
このロックを件数確認より前に取得し、Usecaseの`commit`または`rollback`まで保持します。<br>
行ロックを利用できないDBでは、Module内の条件付き更新やDB制約などで同等の同時実行制御が必要です。

この例では、DB処理が失敗したらUsecaseが`rollback`し、メールは`commit`後に送ります。<br>
送信が失敗すると例外が呼び出し元へ伝わりますが、確定済みのDB更新は戻りません。<br>
再送や配信保証が必要なら、Usecaseの失敗時の方針としてOutboxなどを導入します。

純粋な発行可否判断は入力と結果を単体テストできます。DBを使う発行処理は、<br>
同じSession内で古いトークンの無効化と新しいトークンの作成が整合すること、途中の失敗、<br>
呼び出し元Usecaseによる`rollback`をDBを使うテストで確認します。

## Module

ModuleはSessionを受け取り、原則として1テーブルの読み書きと標準操作を提供します。

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

AccountModuleは`account`テーブルだけを読み書きします。<br>
`get_by_email_for_update()`は、パスワードリセット処理で対象行のロックが必要な読み取りです。<br>
別のModuleを呼ばず、`commit`やResponse DTOへの変換も行いません。<br>
DB操作には、`Session.query()`ではなくSQLAlchemy 2.xの`select()`と`scalars()`を使います。

対象テーブルへの書き込みに必要な場合だけ、例外として別テーブルを参照できます。<br>
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

## 複数Moduleと外部I/O

パスワードリセットでは、Usecaseが2つのModuleとMailerを次の順序で扱います。

```text
ForgotPasswordUsecase
  AccountModule.get_by_email_for_update()
  PasswordResetTokenModule.count_created_today()
  can_issue_password_reset()
  issue_password_reset_token() → PasswordResetTokenModuleの無効化と発行
  db.commit()
  Mailer.send_password_reset()
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
