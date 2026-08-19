# 適用限界と発展

HUMQの適用限界は、テーブル数、Usecase数、コード行数だけでは決まりません。<br>
主要な業務フローをUsecaseから追跡でき、共有処理がその補助に留まっているかで判断します。

Operationは、同じ不変条件の実装が分岐してはならない場合に限って使う例外です。<br>
Operationが1つ存在するだけで適用限界にはなりませんが、増加は設計を見直す兆候になります。

## 共通化する前に

処理が似ている、Usecaseが長い、Module呼び出しが多いという理由だけでは共通化しません。

Operationへ切り出す前に、次を確認します。

- 複数Usecaseから実際に使われている。
- 同じ不変条件を守り、実装の分岐を許容できない。
- 不変条件が破られた場合の具体的な不整合を説明できる。
- 同じ検証、エラー、ロック、更新順序を共有する必要がある。
- 切り出した後も、主要なフローをUsecaseから追跡できる。

これらを満たさない場合、重複は追跡可能性を守るための意図的な選択です。

## Operationを例外として使う

上記の条件を満たす場合だけ、処理を所有ドメインの`_operations.py`に置きます。<br>
1ファイルでは読みづらくなった場合は、業務能力や変更理由ごとに、<br>
同じドメイン直下の複数ファイルへ分割できます。

```text
usecases/
└── procurement/
    ├── create_order.py
    ├── receive_goods.py
    ├── _policies.py
    ├── _reservation_operations.py
    └── _billing_operations.py
```

ファイル名は`_<business-capability>_operations.py`を基本とします。<br>
`_helpers.py`、`_common_operations.py`、`_database_operations.py`、`_misc_operations.py`のような、<br>
汎用的・技術的な分類や、1クラスごとの機械的な分割は行いません。

分割後も所有ドメインが明確で、Usecaseから参照先を追いやすく、<br>
主要なフローを隠さないことが条件です。Operationファイルは`__init__.py`から再exportせず、<br>
ルートの`usecases/_operations.py`や`usecases/<domain>/operations/`は作りません。<br>
複数ドメインから使われても、Operationはその業務能力を所有するドメインに置きます。

Operationの配置とルールは、[層と責務のルール](02-layer-rules.ja.md#operation)で説明します。

## Operationの増大を適用限界の目安にする

行数やファイル数だけで上限は決めませんが、Operationは例外であり、<br>
増えることを通常の成長として扱いません。読みやすさのために複数ファイルへ分ける場合も、<br>
Usecaseの主要なフローを補助する役割に留まっているかを確認します。

次の兆候が複数現れた場合は、Operationを増やし続けず、対象ドメインの設計を見直します。

- Operationの所有ドメインや変更理由を説明できない。
- 無関係な共有処理が集まり、実質的に汎用Serviceになっている。
- Operation同士の呼び出しや依存の連鎖を追う必要がある。
- 多くのUsecaseがOperationを順番に呼んで`commit`するだけになる。
- 主要な業務フローや条件分岐がUsecaseからOperationへ移っている。
- 多数のテーブルを常に1つの整合性境界として扱う必要がある。
- 同じ複雑な共有不変条件を守ることが、対象ドメインの中心になる。

## 適用限界に達した場合

Operationファイルを整理するだけで構造上の問題を隠さず、<br>
まず対象ドメインの境界と、共有処理の所有者を見直します。

Operationは、不変条件の実装を1か所に保つことで、Aggregateに近い役割の一部を担えます。<br>
ただし、呼び忘れを構造的には防ぎません。Operationが増え続ける場合や、<br>
呼び忘れ自体を許容できない場合は、対象ドメインだけを、<br>
DDD、Aggregate中心設計、または別の適切な設計へ段階的に移行します。<br>
Handlerから呼ばれるUsecaseを維持すれば、外部APIを変えずに内部設計を移行できます。

---

前へ: [FastAPI構成例](06-fastapi-example.ja.md) | 次へ: [README](../README.ja.md)
