# 既存アーキテクチャとの比較

HUMQは、MVC、レイヤードアーキテクチャ、クリーンアーキテクチャ、DDDを否定するものではありません。それらが実務で曖昧になりやすい責務境界を、より固定的に扱う設計です。

## 既存設計で起きやすい問題

### ServiceとRepositoryの粒度が揺れる

Serviceに業務ロジックを書くのか、Repositoryにどこまで賢さを持たせるのかは、プロジェクトや開発者によって解釈が変わりやすい部分です。

結果として、Serviceが肥大化したり、Repositoryが業務判断を持ち始めたりします。

### Controllerが太る

下層の責務が曖昧だと、処理の逃げ場がControllerに向かいます。

Validation、条件分岐、DB操作、レスポンス整形が混在すると、Controllerは外界の入口ではなく、業務処理の本体になってしまいます。

HUMQでは、この層をHandlerと呼び、入出力に責務を限定します。

### Domainが現実の例外で汚れる

DDDではDomainやAggregateが強い秩序を持ちます。しかし現実の業務は、例外、暫定対応、組織都合、画面都合を含みます。

それらをDomainへ入れると、長期的に純度を保ちにくくなります。

HUMQでは、現実の例外はUsecaseで受け止め、Moduleのテーブル単位の秩序を守ります。

## HUMQの整理

| よくある層 | HUMQでの扱い |
| --- | --- |
| Controller | Handler。入出力に限定する |
| Service | UsecaseとModuleに分ける |
| Domain | 直接対応させない。安定したテーブル操作はModule、業務フローはUsecaseに置く |
| Repository | 必須層ではない。必要ならModule内部の補助として扱う |
| Read Model / Report | Queryとして読み取り専用に分離する |

## HUMQが強い場面

HUMQは、次のようなシステムで特に効果を発揮します。

- 業務フローが多い。
- 例外処理や暫定対応が避けられない。
- テーブル数が増えやすい。
- 複数テーブルをまたぐ一覧、検索、集計が多い。
- 開発者が増えても置き場所を揃えたい。
- 長期運用で責務境界を保ちたい。

## HUMQが要求するもの

HUMQは、Usecaseに自由を与えます。そのため、Usecaseを書く人には責任が必要です。

- 整合性をUsecaseで明示する。
- Moduleを都合よく汚さない。
- 大きな業務概念を表現するために複数テーブルを1つのModuleへまとめない。
- Queryを読み取り専用に保つ。
- Handlerを薄く保つ。
- 便利さより責務境界を優先する。

HUMQの弱点は、整合性が自動的に守られるわけではないことです。しかし、その弱点は同時に強みでもあります。整合性が隠れず、業務意志としてコード上に現れるためです。

## 比較表

| 観点 | 一般的なレイヤード構造 | DDD | HUMQ |
| --- | --- | --- | --- |
| 責務境界 | Serviceが曖昧になりやすい | Aggregate設計に依存する | 4つの実務ルールへ再定義する |
| 業務例外 | ServiceやDomainに散らばりやすい | Domainを汚しやすい | Usecaseが引き受ける |
| 読み取り横断 | RepositoryやServiceに混ざりやすい | Read Model設計が別途必要 | Queryに分離する |
| トランザクション | Serviceに置かれがち | Aggregate単位 | Usecase単位 |
| 再利用性 | Service粒度に依存する | Aggregate境界に依存する | 小さなテーブル単位のModuleで保ちやすい |
| 長期運用 | 人によって崩れやすい | 設計難度が高い | 置き場所の迷いを減らす |

---

前へ: [整合性とトランザクション](04-consistency-and-transactions.ja.md) | 次へ: [FastAPI構成例](06-fastapi-example.ja.md)
