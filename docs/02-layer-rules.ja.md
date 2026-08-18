# 層と責務のルール

HUMQは、コードの置き場所をHandler、Usecase、Module、Queryの4つの責務で決めます。<br>
外部クライアントは外部システムとの接続先であり、HUMQの層には含めません。

## Handler

Handlerは、HTTP、イベント、CLIなどの呼び出し元をアプリケーションへ接続します。<br>
呼び出し元からの入力をUsecaseへ渡し、その結果を呼び出し元の形式へ変換します。

### 置くもの

- ルーティング、イベント購読、CLIコマンドの入口
- リクエストやメッセージの受け取り
- 必須項目や型など、入力形式の検証
- 認証済みユーザーやリクエスト情報など、外界から得た文脈の受け渡し
- Usecaseの呼び出しと、レスポンスやステータスコードへの変換

### 置かないもの

- 業務上の条件分岐や権限判断
- Usecaseを介さないModule、Query、外部クライアントの呼び出し
- DB操作とトランザクション管理
- JOINや集計

## Usecase

Usecaseは、説明可能な1つの業務処理と、その主要なフローを表します。<br>
現実の分岐や特例を引き受け、必要なModule、Query、<br>
外部クライアントを明示的に組み合わせます。

### 置くもの

- 業務上意味のある処理順序と条件分岐
- Moduleの組み合わせとQueryによる読み取り
- 状態遷移と整合性の判断
- トランザクション境界
- 外部クライアントの呼び出しと失敗時の方針
- 例外的な業務要件

### 置かないもの

- HTTPリクエストやレスポンスなど、外界の形式への依存
- ORMのセッションやクエリAPIを使った直接的なDB操作
- ORMモデルを直接変更して永続化する処理
- ORMや生SQLによる直接的な読み取り
- 通信方法やレスポンス解析など、外部クライアントの実装詳細
- 無関係な複数の業務処理
- 主要な業務フローを隠すServiceや補助処理

UsecaseはSQLAlchemyの`Session`を受け取り、ModuleやQueryとの間で、<br>
ORMモデルを含む処理結果を受け渡せます。永続化方式から独立したDomain Entityや、<br>
Input、Output、DTOへの変換は必須ではありません。

ただし、UsecaseがORMのクエリAPIを使ってデータを直接取得したり、<br>
ORMモデルを変更して永続化したりすることはできません。永続化操作はModule、<br>
複数テーブルを横断する読み取りはQueryを通し、<br>
Usecaseは業務フローとトランザクションを所有します。

Handlerから直接呼ばれるUsecaseがトランザクション境界を確定し、<br>
必要な`begin`、`commit`、`rollback`に責任を持ちます。

### 1 Usecase = 1つの主要フロー

主要な処理順序と分岐、利用するModuleとQuery、状態変更、<br>
トランザクション境界、外部I/Oと失敗時の方針は、Usecaseファイルから読み取れる必要があります。

Usecaseの長さではなく、1つの業務処理として上から下まで追えるかで判断します。<br>
可読性と分割の考え方は、[設計原則](03-design-principles.ja.md)で説明します。

複数のUsecaseで同じ処理を使う場合、DBを使わない判断や計算はPolicy、<br>
ModuleやQueryを使う処理はOperationに置けます。どちらもUsecaseから呼び出し、<br>
Handlerからは直接呼びません。HUMQの新しい層ではありません。

## Policy

Policyは、渡された値だけで判断や計算を行う処理です。DBにはアクセスしません。<br>
関数で十分ならクラスを作る必要もありません。

Policyは次の条件をすべて満たします。

- DB、Session、Module、Query、外部クライアントを使わない。
- `begin`、`commit`、`rollback`、`flush`を行わない。
- 同じ入力に対して同じ結果を返す。
- 呼び出しと、その結果による主要な分岐をUsecaseから読み取れる。

### Policyの配置

Policyと純粋な補助関数は、次の優先順位で配置します。

1. 1つのUsecaseだけで使う判断や計算は、そのUsecaseファイル内に置く。
2. 同一ドメインの複数Usecaseで共有する判断は、`usecases/<domain>/_policies.py`に置く。
3. ドメインに依存せず全体で共有する純粋な計算だけを、`usecases/_policies.py`に置く。

```text
usecases/
├── _policies.py
├── procurement/
│   ├── create_order.py
│   ├── receive_goods.py
│   └── _policies.py
├── returns/
│   ├── request_return.py
│   ├── receive_return.py
│   └── _policies.py
└── billing/
    ├── generate_invoice.py
    ├── post_payment.py
    └── _policies.py
```

先頭の`_`は、Handlerから直接使わない内部ファイルであることを示します。<br>
`_policies.py`の関数や型は、ドメインの`__init__.py`から再exportしません。

### Policyに置かないもの

DBからのデータ取得、ModuleやQueryの呼び出し、データの更新、外部I/OはPolicyに置きません。<br>
Usecaseを短く見せるためだけに、処理をPolicyへ移すことも避けます。

例えば、返品可能数の算出式はPolicyに置けます。出荷数と過去の返品数の取得は、<br>
ModuleまたはQueryに置き、Policyを呼んで返品を登録する流れはUsecaseに残します。

## Operation

Operationは、複数のUsecaseで共有するDB依存の処理です。ModuleやQueryを利用できますが、<br>
`begin`、`commit`、`rollback`は行いません。Handlerから直接呼ばず、Usecaseから利用します。

Usecaseから別のUsecaseを呼ぶ代わりに、共有する内部処理をOperationとして切り出します。

認可、採番、重複確認、共通の事前条件、限定された複数テーブル更新などを、<br>
複数Usecaseから同じ意味と規則で利用する場合にOperationを検討します。<br>
DBを使わない処理はOperationではなくPolicyにします。

### Operationの配置と命名

Operationは、まず所有ドメインの`usecases/<domain>/_operations.py`に置きます。<br>
Operationが少ない間は、1つのファイルへまとめます。<br>
クラス名は`*Operation`、メソッド名は`run()`を基本とし、公開Usecaseの`*Usecase`と`execute()`から区別します。

```text
usecases/
└── procurement/
    ├── create_order.py
    ├── receive_goods.py
    ├── _policies.py
    └── _operations.py
```

先頭の`_`は内部モジュールであることを示します。`_operations.py`は`__init__.py`から再exportしません。<br>
Operationが増えて1ファイルでは読みづらくなった場合の分割ルールは、<br>
[適用限界と発展](07-adoption-limits-and-evolution.ja.md)で説明します。

### Operationのルール

- 呼び出し元Usecaseと同じSessionを使用する。
- ModuleとQueryを呼び出してよい。
- 1つまたは複数のテーブルを読み書きしてよいが、書き込みは各Moduleを通す。
- 必要な検証、ロック、`flush`を行ってよい。
- `begin`、`commit`、`rollback`を行わない。
- Handlerから直接呼び出さない。
- 別のOperationを呼び出さない。
- 外部クライアントの呼び出しと、複数Operationの組み合わせはUsecaseに残す。
- 呼び出しと、結果による主要な分岐をUsecaseから読み取れるようにする。
- 成功、失敗、呼び出し元による`rollback`をテストする。

外部I/Oと失敗時の方針、トランザクションの確定は、呼び出し元Usecaseに残します。

### Operationへ抽出する基準

Operationへ抽出するのは、複数Usecaseから実際に使われ、同じ業務上の意味と変更理由を持ち、<br>
同じ検証、エラー、ロック、更新順序を共有する必要がある場合です。抽出後も、<br>
主要なフローはUsecaseから追跡できなければなりません。

## Module

Moduleは、原則として1テーブルの読み書きを扱います。

そのため、正規化されたテーブル構造が複数のModule呼び出しとして、<br>
Usecaseに現れることがあります。これは、抽象的なDomain境界よりも、<br>
テーブルを変更するコードの所在を機械的に判断できることを優先した結果です。

例外として、対象テーブルへの書き込みに別テーブルの情報が必要な場合は、<br>
SELECT、JOIN、サブクエリで参照できます。ただし、別テーブルの状態は変更せず、<br>
この例外によってModuleの通常の読み取り範囲を広げてはいけません。

### 置くもの

- 対象とする1テーブルの作成、更新、削除
- 主キー取得、存在確認、標準一覧など、そのテーブルの標準的な取得
- 複数のUsecaseで繰り返し使う、そのテーブルの基本的な検索
- そのテーブルの値だけで判断できる制約と状態変更
- 条件付き更新やロックなど、1テーブルに対する同時更新の制御
- 例外として、対象テーブルへの書き込みに必要な別テーブルの参照
- ORMやSQLを使った永続化の実装詳細

### 置かないもの

- 対象外のテーブルへの書き込み
- 他のModuleへの依存
- Usecase固有の条件分岐や業務フロー
- `commit`や`rollback`によるトランザクション境界の確定
- 外部システムとの通信
- ORMのcascade、hook、callbackによる別テーブルの暗黙的な書き込み

例えば、注文と注文明細を参照して請求書を作る処理は、<br>
`InvoiceModule`が次のようなSQLを実行できます。

```sql
INSERT INTO invoices (order_id, customer_id, amount)
SELECT orders.id, orders.customer_id, SUM(order_items.amount)
FROM orders
JOIN order_items ON order_items.order_id = orders.id
WHERE orders.id = :order_id
GROUP BY orders.id, orders.customer_id;
```

この処理は`orders`と`order_items`を参照しますが、<br>
状態を変更するのは`invoices`だけなので、`InvoiceModule`の責務に収まります。

複数テーブルへの書き込みは、原則としてUsecaseが複数のModuleを呼び出して表現します。<br>
単一SQLやストアドプロシージャによる複数テーブルへの書き込みを避けられない場合は、<br>
通常規則に対する例外として、対象テーブルと理由を明記し、ADRと結合テストを残します。<br>
この例外を汎用的なServiceへ拡張してはいけません。

QueryはModuleの対象テーブルも読み取れます。書き込む中間テーブルには、<br>
独立したModuleを用意します。

RepositoryはHUMQの層ではなく、必須でもありません。<br>
永続化処理を分離する必要がある場合にだけ、Module内部の補助実装として置きます。<br>
Usecaseから直接呼ばず、Repositoryを切り出しても1テーブルの境界は変わりません。

## Query

Queryは読み取り専用で、複数テーブルを横断し、画面、検索、帳票、CSV、分析などに必要な、<br>
読み取りモデルを作ります。

主キー取得、存在確認、標準一覧など、Moduleが扱うテーブルの標準操作はModuleです。<br>
ただし、画面や表示DTOに特化した複雑な検索、ウィンドウ関数、特殊なSQLなど、<br>
Moduleの標準操作に収まらない用途固有の読み取りは、<br>
1テーブルだけを参照する場合でも、例外としてQueryに置けます。

JOIN、集計、検索条件が増えてQueryが長くなっても、<br>
1つの観測目的や読み取りモデルとして説明できる限り、行数だけを理由に分割しません。

### 置くもの

- 複数テーブルを使う画面、検索、帳票、CSV、ダッシュボード、分析のための読み取りモデル
- JOINや集計による横断的な読み取り
- 例外として、複雑な検索条件、ウィンドウ関数、JSON、用途固有のSQLを使う1テーブルの読み取り
- 画面や業務文脈に合わせた読み取りモデルへの変換

### 置かないもの

- `INSERT`、`UPDATE`、`DELETE`による書き込み
- ORMモデルの状態変更
- `commit`や`rollback`によるトランザクション管理
- 業務フローと状態遷移
- Moduleが提供する標準的なCRUDや基本取得の重複

利用するORMやDB接続の内部でトランザクションが使われていても、<br>
Queryはアプリケーション上のトランザクション境界を所有しません。

### 読み取り専用Usecase

Queryは「どのようにデータを読むか」を表し、<br>
Usecaseは「アプリケーションが呼び出し元にどの操作を提供するか」を表します。<br>
HandlerがQueryを直接呼ぶと、外部インターフェースがDBの読み取り構造へ直接結合します。

そのため、読み取り専用で処理をQueryへ委譲するだけの場合もUsecaseを通します。<br>
薄いUsecaseであること自体は問題ではありません。

## 外部クライアント

外部クライアントは、外部システムとの通信を隠すアダプターです。<br>
Usecaseから呼び出し、通信方法と外部システム固有のデータ形式をUsecaseから分離します。

### 置くもの

- HTTP、SDK、メッセージブローカーなどを使った通信
- 認証情報とリクエストの組み立て
- レスポンスの解析とアプリケーション用データへの変換
- タイムアウトや通信エラーの変換

### 置かないもの

- 業務フローと業務上の条件分岐
- ModuleやQueryの呼び出し
- DB操作とトランザクション管理
- 外部処理が失敗した場合の業務上の方針

外部処理を実行するか、失敗時に再試行や補償を行うかはUsecaseが決めます。

## HUMQが規定する範囲

HUMQが規定するのは、APIなどから始まる業務処理における、<br>
Handler、Usecase、Module、Queryの責務境界です。アプリケーション全体の、<br>
ディレクトリ構成やインフラストラクチャの分割方法は規定しません。

DB接続、設定、外部クライアント、メール、ストレージ、キャッシュ、ログ、<br>
テレメトリ、メトリクス、認証、マイグレーション、バッチ、フレームワーク固有のコードは、<br>
プロジェクトに合わせて配置できます。

`core/`、`clients/`、`infrastructure/`、`integrations/`、`adapters/`、`config/`などは、<br>
プロジェクト側の選択です。特に`core/`に、HUMQ固有の公式な意味はありません。

## 命名

- **Handler**: HTTPはリソース名で`handlers/`直下に置く。イベントやCLIは、<br>
  必要に応じて入力方式ごとのサブディレクトリへ分ける。
- **Usecase**: Handlerから呼ばれるUsecaseは、対応するリソースのディレクトリに置き、<br>
  動詞または動詞句で命名する。
- **Module**: 対応するテーブルを表す単数形の名詞で命名する。
- **Query**: テーブル名ではなく、観測対象や業務文脈で命名する。
- **外部クライアント**: 外部サービスや提供する通信機能で命名する。

```text
handlers/accounts.py
handlers/events/order_created.py
handlers/cli/rebuild_index.py
usecases/accounts/signup.py
usecases/accounts/list_accounts.py
modules/account/module.py
modules/account_role/module.py
queries/account_orders.py
queries/sales_report.py
clients/payment_gateway.py
clients/shipping_api.py
```

---

前へ: [HUMQの概要](01-overview.ja.md) | 次へ: [設計原則](03-design-principles.ja.md)
