# HUMQ - カオスを設計するアーキテクチャ

<img align="right" src="assets/logo.png" alt="HUMQ logo" width="190">

> English: [README.md](README.md)

HUMQは、Handler / Usecase / Module / Query の4層で、アプリケーションを整理する<br>
ソフトウェアアーキテクチャの設計原則です。<br>
現実の業務に必ず存在する例外や変更を排除するのではなく、どこに置くべきかを明確にします。<br>

Handlerは外界との接点を受け持ち、<br>
Usecaseは業務上の意志と複雑さを引き受けます。<br>
Moduleは1つの対象に閉じた内部秩序を守り、<br>
Queryは複数の対象を横断する読み取りを観測として扱います。

HUMQが重視するのは、完全に壊れないことではありません。<br>
壊れたときにどこを片付ければよいかが分かり、責務境界が歪まないことです。<br>

> すべてを秩序化するのではなく、秩序の中に必要なカオスを許す。

## ドキュメント

- [HUMQの概要](docs/01-overview.ja.md)
- [層と責務のルール](docs/02-layer-rules.ja.md)
- [設計原則](docs/03-design-principles.ja.md)
- [整合性とトランザクション](docs/04-consistency-and-transactions.ja.md)
- [既存アーキテクチャとの比較](docs/05-comparison.ja.md)
- [FastAPI構成例](docs/06-fastapi-example.ja.md)

## HUMQの一文定義

HUMQは、秩序を守る層とカオスを引き受ける層を明確に分けることで、壊れても片付けやすく、歪みにくい構造を作るための設計です。

## 4層の役割

| 層 | 役割 | 主な責務 |
| --- | --- | --- |
| Handler | 外界の入口 | HTTPやイベントを受け、入出力を変換する |
| Usecase | 意志と現実 | 業務フロー、整合性、トランザクションを扱う |
| Module | 内部秩序 | ORMなどを使い、1テーブルまたは1対象に閉じた操作を提供する |
| Query | 観測 | JOIN、集計、横断的な読み取りを担当する |

## 基本方針

- Handlerは薄く保つ。
- Usecaseは現実の複雑さを引き受ける。
- Moduleは1つの対象に閉じ、他のModuleを知らない。
- Queryは読み取り専用にし、書き込みやトランザクションを持たない。
- トランザクション境界はUsecaseに置く。
- 壊れることは許すが、責務境界が歪むことは許さない。
