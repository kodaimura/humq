# FastAPI構成例

このドキュメントでは、FastAPIでHUMQを適用する場合の構成例を示します。

## ディレクトリ構成

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

## Handler例

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

Handlerはリクエストを受け、Usecaseを呼び、レスポンスへ変換するだけです。

## Usecase例

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

Usecaseは業務フローとトランザクション境界を持ちます。複数Moduleを組み合わせる責務もUsecaseにあります。

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

ModuleはAccountという対象に閉じた操作を提供します。他のModuleは呼びません。トランザクションも管理しません。

Repositoryは必須ではありません。ORM操作がModule内で十分に読めるなら、そのままModuleに書きます。永続化処理が大きくなった場合だけ、Module内部の補助として切り出します。

## Query例

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

Queryは読み取り専用です。JOINや集計のような横断的な観測を担当します。

## 判断基準

実装中に迷ったら、次の基準で置き場所を決めます。

| 処理 | 置き場所 |
| --- | --- |
| APIの入力と出力 | Handler |
| 業務手続き | Usecase |
| トランザクション境界 | Usecase |
| 1テーブルに閉じた操作 | Module |
| DBへの単純アクセス | Module |
| 複数テーブルをまたぐ読み取り | Query |

---

前へ: [既存アーキテクチャとの比較](05-comparison.ja.md) | 次へ: [README](../README.ja.md)
