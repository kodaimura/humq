# FastAPI構成例

このドキュメントでは、FastAPIでHUMQを適用する場合の小さな構成例を示します。

## ディレクトリ構成

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

重要なアプリケーションの流れは次の通りです。

```text
Handler
  ↓
Usecase
  ├── AccountModule
  ├── AccountRoleModule
  ├── Query（複数テーブルを横断する読み取りが必要な場合）
  └── Mailer（外部Side Effectが必要な場合）
```

Module、Query、外部clientはすべてUsecaseから呼びます。HandlerがQueryを直接呼んだり、外部clientをModuleとして扱ったりはしません。

## Handler例

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

HandlerはFramework固有のRequest DTOを受け取り、Usecaseには通常の引数として渡します。UsecaseはFastAPIやPydanticのRequest objectに依存しません。

## Usecase例

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

Usecaseは業務フローとトランザクション境界を持ちます。テーブル単位のModuleを組み合わせる責務もUsecaseにあります。

このファイルから、「Account作成 → AccountRole作成 → commit → welcome mail」という主要なフローを上から下まで追えます。この順序や外部I/Oを別のServiceへ移してUsecaseを短く見せることはしません。

外部Side EffectはDBトランザクションのcommit後に実行します。この例では、メール送信が失敗してもDB更新はcommit済みです。実システムでは、失敗を呼び出し元へ返す、記録して再送する、Outbox Patternを使うなど、必要な保証を明示的に選びます。

```python
# usecases/accounts/list_accounts.py

from app.queries.account_overview import list_account_overview


def list_accounts(session):
    return list_account_overview(session)
```

読み取りUsecaseは薄くて構いません。重要なのは、HandlerはUsecaseを呼び、どのQueryで読み取り意図を表現するかはUsecaseが決めることです。

## Module例

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

各Moduleは正確に1テーブルに閉じ、そのテーブルの書き込みを所有します。`AccountModule` は `AccountRoleModule` を呼びません。`AccountRoleModule` も `AccountModule` を呼びません。ORM cascadeやhookによって相手のテーブルを暗黙に更新することも避けます。

Repositoryは必須ではありません。ORM操作がModule内で十分に読めるなら、そのままModuleに書きます。永続化処理が大きくなった場合だけ、Module内部の補助として切り出します。

## Query例

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

Queryは読み取り専用です。複数テーブルを横断して読んで構いませんが、書き込みはせず、トランザクション境界も所有しません。1テーブルだけに閉じた検索や集計は、複雑でもModuleに置きます。

## 判断基準

実装中に迷ったら、次の基準で置き場所を決めます。

| 処理 | 置き場所 |
| --- | --- |
| APIの入力と出力 | Handler |
| 業務手続き | Usecase |
| 1つの業務処理の主要なフロー | 1つのUsecaseファイル |
| トランザクション境界 | Usecase |
| 1テーブルに閉じた読み書き | Module |
| 読み取り意図 | Usecase |
| 複数テーブルをまたぐ読み取り | Usecaseから呼ばれるQuery |
| 複数Usecaseで共有する純粋な判定 | Policyまたは補助関数。呼び出しと分岐はUsecaseに残す |
| Mail、決済、外部API | Moduleではない外部client。Usecaseから明示的に呼ぶ |

---

前へ: [既存アーキテクチャとの比較](05-comparison.ja.md) | 次へ: [README](../README.ja.md)
