# 整合性の扱い

HUMQでは、複数テーブルの整合性とDBトランザクションの境界をUsecaseの責務として扱います。<br>
この章では、その設計が引き受けるリスクと、Usecase、内部処理、Module、DB、テストによる、<br>
整合性の守り方を説明します。

## HUMQのトレードオフ

HUMQは、複数テーブルの整合性を構造的には保証しません。<br>
Handler、Usecase、Module、Queryの責務境界を正しく守っていても、<br>
Usecaseが必要な業務処理の呼び出しや整合性ルールを実装し忘れれば、<br>
不整合なデータがそのまま保存される可能性があります。

これはHUMQの制約であり、軽量で明確な配置規則と引き換えに、<br>
意図的に引き受けるトレードオフです。DB制約とUsecaseのテストでリスクを軽減しますが、<br>
すべての業務上の整合性を表現し、実装漏れを構造的に防げるわけではありません。<br>
このリスクを許容できない領域では、Aggregate中心のDDDなど、<br>
不変条件をDomain Modelに閉じ込める設計を選びます。

## 責務の分担

- **Usecase**: 複数テーブルをまたぐ整合性の責任、主要な処理順序、失敗時の方針、トランザクション境界を持つ。
- **分離する場合のUsecase内部の業務処理**: 意味のある判断・計算や整合性処理を担い、必要に応じてModuleとQueryを使う。トランザクション境界は所有しない。
- **Module**: 原則として1テーブルの読み書きを提供し、`commit`や`rollback`を呼ばない。
- **Query**: 読み取り専用とし、トランザクション境界を持たない。
- **DB**: DBで表現できる制約と、同時更新を制御する仕組みを持つ。
- **Test**: 純粋な処理は単体テストし、DBを利用する処理は整合性、失敗、呼び出し元による`rollback`を検証する。

## トランザクション境界

トランザクション境界は、テーブルやModuleの都合ではなく、<br>
業務上どの状態変更が一緒に成立しなければならないかで決めます。

例えば、注文の確定、在庫の引き当て、配信要求の登録が、<br>
どれか1つでも欠けると不正な状態になるなら、同じトランザクションで扱います。
次の概略コードでは、トランザクションの流れを示すためにModuleの準備と例外の定義を省略しています。

```python
# usecases/orders/confirm_order.py
from usecases.inventory._reservation import reserve_inventory

def confirm_order(session, order_id: int) -> None:
    with session.begin():
        order = order_module.get_for_update(session, order_id)
        items = order_item_module.list_by_order(session, order_id)
        unavailable_product_id = reserve_inventory(session, items)
        if unavailable_product_id is not None:
            raise InsufficientInventory(unavailable_product_id)
        order_module.mark_confirmed(session, order.id)
        outbox_module.enqueue_order_confirmed(session, order.id)
```

在庫引当は独立した業務上の意味を持つため、所有する領域の内部ファイルへ分離できます。

```python
# usecases/inventory/_reservation.py
def reserve_inventory(session, items) -> int | None:
    for item in items:
        updated = inventory_module.decrease_if_available(
            session,
            product_id=item.product_id,
            quantity=item.quantity,
        )
        if not updated:
            return item.product_id
    return None
```

内部処理は呼び出し元Usecaseと同じSessionを使い、トランザクション境界を所有しません。<br>
在庫が不足するとUsecaseがトランザクション内で例外を出し、<br>
それまでの在庫更新は`rollback`されます。注文確定と配信要求登録には進みません。<br>
主要な順序とトランザクション境界はUsecaseから、<br>
在庫引当の検証・変更対象は参照先から確認できます。

HUMQは、`1 Usecase = 1 Transaction`を強制しません。<br>
読み取りだけのUsecaseは、明示的なトランザクションを必要としない場合があります。<br>
複数リクエストや外部システムをまたぐ業務フローでは、複数のトランザクションを使えます。<br>
その場合も、各トランザクションで確定する状態と、失敗後の方針をUsecaseから追えるようにします。

## DBで守る整合性

Usecaseに処理を書いただけでは、更新漏れや同時更新による不整合は防げません。

`UNIQUE`、`NOT NULL`、`CHECK`、外部キーなど、DBで表現できる規則はDB制約で守ります。<br>
在庫の減算など、複数のリクエストが同じデータを更新する処理には、<br>
条件付き更新、行ロック、楽観ロックなど、必要な競合制御をModuleの操作として実装します。

Usecaseまたは明確に名付けた内部処理が、業務条件に応じてModule操作を呼びます。<br>
失敗時の方針はUsecaseから追えるようにします。

## 外部システムとの整合性

メール送信や決済APIの呼び出しは、DBと同じトランザクションでは扱えません。<br>
外部処理の成功後にDBが`rollback`する場合も、DBの`commit`後に外部処理が失敗する場合もあります。

- 失敗を許容できる通知は、DBの`commit`後に実行する。
- 配信要求を失えない場合は、同じトランザクションでOutboxテーブルへ記録する。
- 決済など外部の状態も変わる場合は、冪等性、再試行、補償処理を設計する。

Outboxを使う場合、Usecaseが保証するのは外部処理の完了ではなく、<br>
DBの状態変更と配信要求の記録が一緒に成立することです。

## 内部処理と検証

在庫引当や重複課金の防止など、独立して説明・検証・変更する意味がある処理は、<br>
使用箇所が1つでもUsecase内部の業務処理として分離できます。<br>
純粋な判断・計算にSessionは不要で、入力と結果を単体テストできます。<br>
複数Usecaseで同じ不変条件を守る場合は、責任と参照先が分かる場所へ処理をまとめ、<br>
検証、エラー、ロック、更新順序が分岐しないようにできます。<br>
ただし、抽出や共通化だけでは全Usecaseによる呼び忘れを構造的には防げません。

DBを利用する内部処理は、呼び出し元Usecaseと同じSessionとトランザクションに参加し、<br>
読み取りをModuleまたはQuery、書き込みをModuleへ委譲します。<br>
ORMやSQLによる直接のデータ取得・永続化を行わず、独自のSessionやトランザクションを作りません。<br>
`begin`、`commit`、`rollback`も行いません。境界と失敗時の方針はUsecaseが所有します。

DBを利用する処理は整合性要件と失敗時の振る舞いを検証します。<br>
後続処理が失敗したときに、呼び出し元Usecaseが変更全体を`rollback`することもテストします。

---

前へ: [設計原則](03-design-principles.ja.md) | 次へ: [既存アーキテクチャ・設計パターンとの比較](05-comparison.ja.md)
