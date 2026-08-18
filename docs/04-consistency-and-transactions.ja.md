# 整合性の扱い

HUMQは、複数テーブルの整合性を構造によって自動的に守る設計ではありません。<br>
必要なModuleの呼び出しや検証をUsecaseに書き忘れれば、データ不整合は起こりやすくなります。<br>
これは、HUMQが引き受ける大きなトレードオフです。

だからこそ、整合性をどこで守るかを曖昧にはできません。<br>
HUMQでは、複数テーブルにまたがる業務上の整合性と、<br>
DBトランザクションの境界をUsecaseの責務として扱います。

## 責務の分担

- **Usecase**: 複数テーブルをまたぐ整合性、処理順序、失敗条件、トランザクション境界を持つ。
- **Operation**: 呼び出し元と同じSessionでModuleやQueryを利用し、共有内部処理を実行する。
- **Module**: 正確に1テーブルの読み書きを提供し、`commit`や`rollback`を呼ばない。
- **Query**: 読み取り専用とし、トランザクション境界を持たない。
- **DB**: DBで表現できる制約と、同時更新を制御する仕組みを持つ。
- **Test**: 業務上の分岐、失敗、`rollback`の振る舞いを検証する。

## トランザクション境界

トランザクション境界は、テーブルやModuleの都合ではなく、<br>
業務上どの状態変更が一緒に成立しなければならないかで決めます。

例えば、注文の確定、在庫の引き当て、配信要求の登録が、<br>
どれか1つでも欠けると不正な状態になるなら、同じトランザクションで扱います。

```python
# usecases/orders/confirm_order.py

def confirm_order(session, order_id: int) -> None:
    with session.begin():
        order = order_module.get_for_update(session, order_id)
        items = order_item_module.list_by_order(session, order_id)

        for item in items:
            updated = inventory_module.decrease_if_available(
                session,
                product_id=item.product_id,
                quantity=item.quantity,
            )
            if not updated:
                raise InsufficientInventory(item.product_id)

        order_module.mark_confirmed(session, order.id)
        outbox_module.enqueue_order_confirmed(session, order.id)
```

在庫が不足すれば例外が発生し、それまでの在庫更新、注文確定、<br>
配信要求の登録はまとめて`rollback`されます。<br>
どのModuleを組み合わせ、どの書き込みをまとめて成功または失敗させるかはUsecaseから確認できます。

HUMQは、`1 Usecase = 1 Transaction`を強制しません。<br>
読み取りだけのUsecaseは、明示的なトランザクションを必要としない場合があります。<br>
複数リクエストや外部システムをまたぐ業務フローでは、複数のトランザクションを使えます。<br>
その場合も、各トランザクションで確定する状態と、失敗後の方針をUsecaseから追えるようにします。

## DBで守る整合性

Usecaseに処理を書いただけでは、更新漏れや同時更新による不整合は防げません。

`UNIQUE`、`NOT NULL`、`CHECK`、外部キーなど、DBで表現できる規則はDB制約で守ります。<br>
在庫の減算など、複数のリクエストが同じデータを更新する処理には、<br>
条件付き更新、行ロック、楽観ロックなど、必要な競合制御をModuleの操作として実装します。

Usecaseは、その操作をどの業務条件で呼び、失敗をどう扱うかを明示します。

## 外部システムとの整合性

メール送信や決済APIの呼び出しは、DBと同じトランザクションでは扱えません。<br>
外部処理の成功後にDBが`rollback`する場合も、DBの`commit`後に外部処理が失敗する場合もあります。

- 失敗を許容できる通知は、DBの`commit`後に実行する。
- 配信要求を失えない場合は、同じトランザクションでOutboxテーブルへ記録する。
- 決済など外部の状態も変わる場合は、冪等性、再試行、補償処理を設計する。

Outboxを使う場合、Usecaseが保証するのは外部処理の完了ではなく、<br>
DBの状態変更と配信要求の記録が一緒に成立することです。

## 同じ不変条件が繰り返される場合

同じ複数テーブル更新を、常に同じ検証、順序、ロック、エラー処理で、<br>
複数Usecaseから実行する場合は、Operationとして共有できます。

単にUsecaseが長い、Module呼び出しが多い、コードが似ているだけでは、<br>
Operationへの抽出を必須としません。同じ業務上の意味と変更理由を持ち、<br>
共有しなければ変更漏れが現実のリスクになるかを確認します。

Operationは呼び出し元Usecaseと同じSessionへ参加し、書き込みを各Moduleへ委譲します。<br>
自身では`begin`、`commit`、`rollback`を行わず、トランザクション境界の所有者は呼び出し元Usecaseのままです。

## HUMQのトレードオフ

HUMQは、必要なModuleの呼び出しや整合性ルールの実装漏れを自動では防ぎません。<br>
Usecaseの実装に漏れがあれば、不整合は起こり得ます。<br>
Operationへ共有しても、その呼び忘れを構造的に防ぐものではありません。

そのため、DB制約とUsecaseのテストを併用します。<br>
HUMQが提供するのは自動的な整合性の保証ではなく、<br>
整合性を扱う場所と、変更時に確認する範囲が分散しない構造です。

---

前へ: [設計原則](03-design-principles.ja.md) | 次へ: [既存アーキテクチャ・設計パターンとの比較](05-comparison.ja.md)
