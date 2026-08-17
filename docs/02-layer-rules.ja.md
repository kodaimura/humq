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
- Module、Query、外部クライアントの直接呼び出し
- DB操作とトランザクション管理
- JOINや集計

## Usecase

Usecaseは、説明可能な1つの業務処理と、その主要なフローを表します。<br>
現実の分岐や特例を引き受け、必要なOperation、Module、Query、<br>
外部クライアントを明示的に組み合わせます。

### 置くもの

- 業務上意味のある処理順序と条件分岐
- OperationとModuleの組み合わせ、Queryによる読み取り
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

### 補助処理、共有ルール、Operation

局所的な計算やデータ変換は、業務フローを隠さない範囲で関数へ抽出できます。<br>
複数Usecaseで使う業務判定は、副作用のないPolicyや関数として共有できます。<br>
Policyの呼び出しと、その結果による分岐はUsecaseに残します。

Usecaseから別のUsecaseを呼ぶことは避けます。複数のUsecaseで再利用する内部の業務処理は、<br>
必要であればOperationとしてUsecaseの責務内に切り出します。

OperationはHUMQの新しい層ではありません。Module、Query、外部クライアントを呼び出せますが、<br>
呼び出し元のSessionとトランザクションに参加し、`begin`、`commit`、`rollback`を行いません。
次の条件をすべて満たす場合にだけ、Operationへの切り出しを検討します。

- 複数のUsecaseから実際に再利用される。
- 独立した業務上の意味を持つ。
- 呼び出し元Usecaseの主要なフローを隠さない。

一部のコードが似ているだけで、無関係な処理を共通化してはいけません。<br>
複数Moduleの操作を共有する場合は、曖昧なhelperやServiceではなく、<br>
業務上の名前を持つOperationとして表現します。

## Module

Moduleは、原則として正確に1テーブルだけを読み書きし、<br>
そのテーブルの標準的なデータ操作をまとめます。HUMQが書き込む各テーブルには、<br>
操作を所有するModuleを1つだけ対応させます。

そのため、正規化されたテーブル構造が複数のModule呼び出しとして、<br>
Usecaseに現れることがあります。これは、抽象的なDomain境界よりも、<br>
テーブルを変更するコードの所在を機械的に判断できることを優先した結果です。

例外として、所有するテーブルへの書き込みに別テーブルの情報が必要な場合は、<br>
SELECT、JOIN、サブクエリで参照できます。ただし、別テーブルの状態は変更せず、<br>
この例外によってModuleの通常の読み取り範囲を広げてはいけません。

### 置くもの

- 所有する1テーブルの作成、更新、削除
- 主キー取得、存在確認、標準一覧など、そのテーブルの標準的な取得
- 複数のUsecaseで繰り返し使う、そのテーブルの基本的な検索
- そのテーブルの値だけで判断できる制約と状態変更
- 条件付き更新やロックなど、1テーブルに対する同時更新の制御
- 例外として、所有するテーブルへの書き込みに必要な別テーブルの参照
- ORMやSQLを使った永続化の実装詳細

### 置かないもの

- 所有していないテーブルへの書き込み
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

QueryはModuleが所有するテーブルを読み取れますが、別のModuleが同じテーブルの、<br>
書き込みAPIを持つことはできません。書き込み対象の中間テーブルにも、<br>
独立したModuleを用意します。

RepositoryはHUMQの層ではなく、必須でもありません。<br>
永続化処理を分離する必要がある場合にだけ、Module内部の補助実装として置きます。<br>
Usecaseから直接呼ばず、Repositoryを切り出しても1テーブルの境界は変わりません。

## Query

Queryは、複数テーブルを横断し、画面、検索、帳票などに必要な、<br>
読み取りモデルを作る責務です。ModuleとQueryは、原則として、<br>
1テーブルの標準操作か、複数テーブルを横断する読み取りかで使い分けます。

主キー取得、存在確認、標準一覧など、Moduleが所有するテーブルの標準操作はModuleです。<br>
ただし、画面や表示DTOに特化した複雑な検索、ウィンドウ関数、特殊なSQLなど、<br>
Moduleの標準操作に収まらない読み取りは、1テーブルだけを参照する場合でも、<br>
例外としてQueryに置けます。

JOIN、集計、検索条件が増えてQueryが長くなっても、<br>
1つの観測目的や読み取りモデルとして説明できる限り、行数だけを理由に分割しません。

### 置くもの

- 複数テーブルを使う画面、検索、帳票、CSV、ダッシュボード、分析のための読み取り
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
- **Operation**: 複数Usecaseで共有する内部処理は、業務上の名前で`usecases/`直下に置く。
- **Module**: 対応するテーブルを表す単数形の名詞で命名する。
- **Query**: テーブル名ではなく、観測対象や業務文脈で命名する。
- **外部クライアント**: 外部サービスや提供する通信機能で命名する。

```text
handlers/accounts.py
handlers/events/order_created.py
handlers/cli/rebuild_index.py
usecases/accounts/signup.py
usecases/accounts/list_accounts.py
usecases/register_order.py
modules/account/module.py
modules/account_role/module.py
queries/account_orders.py
queries/sales_report.py
clients/payment_gateway.py
clients/shipping_api.py
```

`usecases/register_order.py`は、複数のUsecaseから共有されるOperationの例です。<br>
Operationは新しい層ではないため、`operations/`ディレクトリは作りません。
ルートのOperationが増えすぎた場合は、ディレクトリを追加する前に、<br>
Operationを切り出しすぎていないかを見直します。

`common.py`や`shared.py`のような名前は避け、ファイルが表す責務で命名します。

---

前へ: [HUMQの概要](01-overview.ja.md) | 次へ: [設計原則](03-design-principles.ja.md)
