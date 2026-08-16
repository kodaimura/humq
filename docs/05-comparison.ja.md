# 既存アーキテクチャとの比較

HUMQは、MVC、レイヤードアーキテクチャ、クリーンアーキテクチャ、DDDを否定するものではありません。これらの設計でも、Usecase相当の場所に業務フローを明示することは可能です。

違いは、複雑さをどこに置くか、下位部品の境界をどのように決めるか、間接参照をどこまで許容するかについて、HUMQが機械的な既定値を持つことです。

## 比較するときの前提

アーキテクチャ名だけで実装の形は決まりません。例えばClean Architectureが必ず`Usecase -> Service -> Domain Service -> Repository`という多段構造になるわけではなく、DDDが必ず業務フローをAggregate内部へ隠すわけでもありません。

HUMQが対象にするのは、既存設計を運用する中で次のような状態になった場合です。

```text
Usecase
   ↓
Service
   ↓
Domain Service
   ↓
Repository
   ↓
Persistence
```

この構造自体が悪いのではありません。業務上重要な手順、分岐、複数データ更新、外部I/O、トランザクション境界が各層へ分散し、全体を理解するために何箇所も追う状態を問題にします。

HUMQは階層を減らすこと自体を目的にせず、処理を追跡するための不要な間接参照を減らします。

## 複雑さを置く場所

```text
HUMQ

Complex business flow
          ↓
       Usecase
   ┌──────┼────────┐
Module  Query  External Client
  ↓       ↓          ↓
1 table  Cross-table Side Effect
```

下位部品は狭く予測可能に保ち、それらをいつ、どの順序で、どの条件で使うかをUsecaseに表現します。

## 各設計との関係

### レイヤードアーキテクチャ

レイヤードアーキテクチャでは、Controller、Service、Repositoryの粒度と責務をチームが決めます。この自由度は有用ですが、Serviceが業務フロー、共通処理、データアクセスの調整を同時に抱えたり、Repositoryが業務判断を持ったりする場合があります。

HUMQはServiceという汎用的な置き場所を使わず、業務フローをUsecase、1テーブル操作をModule、横断的な読み取りをQueryへ固定します。

### クリーンアーキテクチャ

クリーンアーキテクチャは依存関係の方向と、ビジネスルールを外部詳細から独立させることを重視します。HUMQは永続化からの独立よりも、テーブル単位の操作対象と業務フローの追跡可能性を優先します。

両者は一部を組み合わせられますが、`Module = exactly one table`はデータベース構造への意図的な結合なので、永続化非依存を最優先する設計とは緊張関係があります。

### DDD

DDDはDomain ModelとAggregate境界を通して業務概念や不変条件を表現します。HUMQはDomain概念の境界をModuleの境界には使いません。Module境界を1 tableへ固定し、複数テーブルにまたがる業務上のまとまりと整合性をUsecaseに表現します。

HUMQはAggregateを簡略化したものではなく、境界を決める判断コストと、横断操作の見えにくさに対する別のトレードオフです。

## HUMQでの対応

| よくある層・概念 | HUMQでの扱い |
| --- | --- |
| Controller | Handler。入出力に限定する |
| Application Service / Interactor | 主要な業務フローはUsecaseに対応する |
| 汎用Service | 対応させない。フローはUsecase、局所操作はModuleへ分ける |
| Domain / Aggregate | 直接対応させない。共有する純粋な判定はPolicyとして抽出できる |
| Repository | 必須層ではない。必要ならModule内部の実装詳細として扱う |
| Read Model / Report | 読み取り専用のQueryとして扱う |
| Mailer / Payment Gateway / API Client | Moduleではなく外部clientとしてUsecaseから呼ぶ |

## 比較表

| 観点 | 一般的なレイヤード構造 | クリーンアーキテクチャ | DDD | HUMQ |
| --- | --- | --- | --- | --- |
| 業務フロー | Serviceの設計次第 | Usecase / Interactorに置ける | Application層やDomainに表現 | 1 Usecaseのprimary flowとして明示 |
| 下位部品の境界 | チームが決める | 依存規則と抽象化で決める | Domain / Aggregateで決める | exactly one tableで機械的に決める |
| 横断的な読み取り | RepositoryやServiceに置かれやすい | Gateway / Read Modelなどを設計 | Repository / Read Modelなどを設計 | 読み取り専用Queryへ固定 |
| 整合性 | Serviceなどが調整 | UsecaseとEntityの設計次第 | Aggregate内とApplication層で守る | Usecaseに横断整合性を明示 |
| 永続化との結合 | 実装次第 | 原則として外部詳細へ追い出す | Domainから分離できる | Moduleをtable構造へ意図的に結合 |
| 間接参照 | 層とService粒度次第 | Port / Adapterにより増える場合がある | Model設計により増える場合がある | 重要なフロー上では最小化する |
| 主な設計コスト | Service / Repositoryの粒度 | 境界、Port、依存関係 | Domain Model、Aggregate | Usecaseの可読性と明示的な整合性 |

この表は優劣ではなく、既定値とトレードオフの比較です。どの設計でも、運用と実装次第で明示的にも不透明にもなります。

## 適用判断とトレードオフ

HUMQは、RDBを主要な永続化基盤とし、複数テーブルの状態変更と横断的な読み取りを扱うアプリケーションを主な対象とします。

### HUMQが得るもの

- 人が変わっても変わらない、予測可能なコードの置き場所
- 重要な業務フローとtransaction境界の追跡可能性
- 1テーブルに閉じた、操作対象が明確なModule
- 横断的な読み取りを書き込みから分離するQuery

### HUMQが引き受けるもの

- 業務が複雑になれば、Usecaseも肥大化する。
- 画面、検索、帳票が複雑になれば、Queryも肥大化する。
- Moduleをテーブル構造へ意図的に結合する。
- 複数テーブルの整合性をDomainへ閉じ込めず、Usecaseとテストで明示的に守る。

HUMQは、複雑さをなくせるとは考えません。<br>
複雑さを追跡可能な場所で引き受け、構造全体へ分散させないことを選びます。

### HUMQが向いている場面

- 業務フロー、例外処理、暫定対応が多い。
- 複数テーブルをまたぐ書き込み手順を明示したい。
- 一覧、検索、帳票など、複数テーブルを横断する読み取りが多い。
- チーム内でコードの置き場所に関する判断を減らしたい。
- 長期運用で、変更時に追うファイル数を抑えたい。
- RDBのテーブルが安定した主要な永続化境界である。

### 別の設計が適する可能性が高い場面

HUMQは、永続化非依存のDomain Model、Event Sourcingの状態管理、<br>
長時間の分散workflow実行基盤そのものを提供しません。

- Event Sourcingなど、テーブルが主要な永続化境界ではない。
- 複雑な不変条件を豊かなDomain ModelやAggregateへ強く閉じ込めたい。
- 永続化方式からDomainを厳密に独立させることが最優先である。
- 外部システムをまたぐ長時間のworkflowが中心で、Sagaやworkflow engine自体が主要な抽象化になる。

決済や外部サービスを利用すること自体は、HUMQの適用対象外になる理由ではありません。<br>
通常の外部連携では、Outbox、冪等性、Saga、補償処理などを補完的に組み合わせます。

### HUMQが要求するもの

機械的な境界は、業務上の正しさを自動保証しません。HUMQを使う実装者には次の責任があります。

- Usecaseに整合性、状態遷移、外部I/Oを明示する。
- 長さではなく、1つの業務意志として追跡できるかを守る。
- 共有処理の抽出によって業務フローを隠さない。
- Moduleを正確に1テーブルへ閉じ、暗黙の別テーブルへの書き込みを作らない。
- Queryを読み取り専用に保つ。
- Handlerを薄く保つ。

HUMQの主張は「HUMQの方が簡単」ではありません。

> 下位の部品を単純に保ち、重要な複雑さをUsecaseから見える場所に置く。

複雑さを消すのではなく、追跡可能な場所へ配置することがHUMQの選択です。

---

前へ: [整合性とトランザクション](04-consistency-and-transactions.ja.md) | 次へ: [FastAPI構成例](06-fastapi-example.ja.md)
