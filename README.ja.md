# HUMQ - カオスを設計するアーキテクチャ

<img align="right" src="assets/logo.png" alt="HUMQ logo" width="190">

> English: [README.md](README.md)

HUMQは、既存アーキテクチャで曖昧になりがちな責務を、<br>
Handler / Usecase / Module / Query という4つの実務ルールへ再定義する<br>
ソフトウェアアーキテクチャの設計原則です。<br>
業務システムのすべてを綺麗に保とうとはしません。<br>
現実の複雑さをどこに置くべきか、その制約を設計します。<br>

Handlerは外界との接点を受け持ち、<br>
Usecaseは説明可能な1つの業務意志、分岐、整合性、トランザクション境界を引き受けます。<br>
Moduleは正確に1テーブルに閉じた読み書きと局所的な秩序を守り、<br>
Queryは読み取り専用の横断的な観測を扱います。

HUMQが重視するのは、完全に壊れないことではありません。<br>
壊れたときにどこを片付ければよいかが分かり、<br>
責務境界が歪まないことです。<br>

> すべてを秩序化するのではなく、秩序の中に必要なカオスを許す。
>
> 壊れてもよい、歪むな。

## ドキュメント

- [HUMQの概要](docs/01-overview.ja.md)
- [層と責務のルール](docs/02-layer-rules.ja.md)
- [設計原則](docs/03-design-principles.ja.md)
- [整合性とトランザクション](docs/04-consistency-and-transactions.ja.md)
- [既存アーキテクチャとの比較](docs/05-comparison.ja.md)
- [FastAPI構成例](docs/06-fastapi-example.ja.md)

## HUMQの一文定義

HUMQは、曖昧になりがちなアプリケーション責務を4つの実務ルールへ再定義し、秩序を守る場所と業務上のカオスを引き受ける場所を分ける設計です。

## 4つの責務ルール

| 層 | 役割 | 主な責務 |
| --- | --- | --- |
| Handler | 外界の入口 | HTTPやイベントを受け、入出力を変換する |
| Usecase | 意志と現実 | 業務フロー、整合性、トランザクションを扱う |
| Module | 局所的な秩序 | 1テーブルに閉じた読み書きを提供する |
| Query | 観測 | 読み取り専用のJOIN、集計、横断的な読み取りを担当する |

## 基本方針

- Handlerは薄く保つ。
- Usecaseは現実の複雑さを引き受ける。ただし、1つのUsecaseは説明可能な1つの業務意志に対応させる。
- Moduleは正確に1テーブルに閉じ、他のModuleを知らない。
- Queryは読み取り専用にする。Queryはトランザクション境界を所有しない。
- トランザクション境界はUsecaseに置く。
- 壊れることは許すが、責務境界が歪むことは許さない。

## License

[MIT](LICENSE)
