# 層と責務のルール

このドキュメントでは、HUMQの各層に置くもの、置かないもの、命名規則を定義します。

## Handler

Handlerは外界の入口です。HTTPリクエスト、イベント、CLI入力など、アプリケーション外部から来た入力を受け取り、Usecaseへ渡します。

### 置くもの

- ルーティング
- リクエストの受け取り
- レスポンスの整形
- 入出力スキーマとの接続
- 認証済みユーザーなど、外界から得た文脈の受け渡し

### 置かないもの

- 業務ロジック
- DB操作
- トランザクション管理
- 複数Moduleの組み合わせ
- 集計やJOINの組み立て

### 命名

Handlerのファイル名はURLリソースに合わせ、複数形の名詞にします。

```text
handlers/
├── accounts.py
├── projects.py
├── users.py
└── health.py
```

## Usecase

Usecaseは業務上の意志を表す層です。複数のModuleを組み合わせ、現実の例外、分岐、整合性、トランザクションを引き受けます。

### 置くもの

- 業務フロー
- 条件分岐
- 複数Moduleの組み合わせ
- Queryの利用
- トランザクション境界
- 例外的な業務要件

### 置かないもの

- HTTPリクエストやレスポンスへの直接依存
- 生SQLによる横断的な読み取り
- ORMモデルへの直接依存
- `commit` / `rollback` をModuleへ委譲する設計

### 命名

UsecaseのディレクトリはHandlerと同じく複数形のリソース名にします。各Usecaseファイルは動詞または動詞句にします。

```text
usecases/
├── accounts/
│   ├── signup.py
│   ├── login.py
│   ├── update_profile.py
│   └── delete.py
└── projects/
    ├── create.py
    └── archive.py
```

`usecases/accounts/signup.py` は、ディレクトリが文脈を語るため十分です。`register_user.py` のように対象を重ねる命名は避けます。

## Module

Moduleは内部秩序を担当します。原則として、1テーブルまたは1対象に閉じた操作を提供します。

### 置くもの

- 対象に閉じた取得、作成、更新、削除
- 対象固有の不変条件
- ORMなどを使った意味のある操作
- 外部連携を1対象として扱う薄い操作

### 置かないもの

- 複数Moduleをまたぐ業務フロー
- Usecaseの都合による一時的な条件分岐
- トランザクション管理
- 他Moduleへの直接依存
- JOINや集計を中心にした読み取り

### 命名

Module名は単数形の名詞にします。

```text
modules/
├── account/
│   ├── model.py
│   └── module.py
├── project/
│   ├── model.py
│   └── module.py
└── mail/
    └── module.py
```

### Repositoryについて

RepositoryはHUMQの必須層ではありません。ModuleがORMを直接扱ってよいです。

永続化処理が増えすぎる場合や、ORM以外のストレージを隠したい場合だけ、Module内部の補助ファイルとして`repository.py`を切り出します。その場合でもRepositoryは外部公開する層ではなく、Usecaseから直接呼びません。

## Query

Queryは読み取り専用の観測層です。複数テーブルを横断するJOIN、集計、レポート、一覧表示のための読み取りを担当します。

### 置くもの

- JOIN
- 集計
- レポート取得
- 検索画面用の読み取り
- 複数Moduleを横断する読み取り

### 置かないもの

- 書き込み
- トランザクション管理
- 業務状態の変更
- Moduleの代わりになるCRUD

### 命名

Queryはエンティティ名ではなく、観測対象や業務文脈で命名します。

| 観測対象 | ファイル名 |
| --- | --- |
| アカウントの注文履歴 | `account_orders.py` |
| プロジェクト進行状況 | `project_progress.py` |
| 売上集計 | `sales_report.py` |
| ユーザー行動全体 | `activity_overview.py` |

Queryは「どのテーブルか」ではなく「何を観測しているか」を語る層です。
