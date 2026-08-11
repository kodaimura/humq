# 整合性とトランザクション

HUMQでは、整合性とトランザクションをUsecaseの責務として扱います。

## 基本ルール

- トランザクションはUsecaseで開始し、Usecaseで終える。
- Moduleは`commit`や`rollback`を呼ばない。
- Repositoryを切り出す場合も、`commit`や`rollback`を呼ばない。
- 複数Moduleをまたぐ整合性はUsecaseに明示する。
- Queryは読み取り専用であり、トランザクション境界を所有しない。

## なぜUsecaseに置くのか

トランザクションは、複数の秩序を一時的に束ねる操作です。

Moduleは正確に1テーブルに閉じた秩序です。Moduleがトランザクションを管理すると、そのModuleを使うUsecase全体の整合性を外から制御しにくくなります。

Usecaseがトランザクションを持つことで、次のことが明確になります。

- どの操作が1つの業務単位なのか。
- どのModuleを組み合わせているのか。
- どの範囲を成功または失敗として扱うのか。
- どの整合性を意図的に守っているのか。

## DDDとの違い

DDDでは、Aggregate Rootが整合性を守ります。HUMQでは、Usecaseが整合性を明示します。

| 観点 | DDD | HUMQ |
| --- | --- | --- |
| 整合性の責務 | Aggregate | Usecase |
| トランザクション境界 | Aggregate単位 | Usecase単位 |
| 柔軟性 | 境界に強く依存する | テーブル単位のModuleを組み合わせやすい |
| 可視性 | 内部に隠れる | コード上に見える |
| 再利用性 | Aggregateに閉じやすい | Module単位で使いやすい |

HUMQは、自動的な整合性よりも、明示的な秩序を優先します。

## 実装イメージ

```python
# usecases/accounts/assign_role.py

def assign_role(session, account_id: int, role_id: int) -> None:
    with session.begin():
        account = account_module.get(session, account_id)
        role = role_module.get(session, role_id)

        account_role_module.register(
            session=session,
            account_id=account.id,
            role_id=role.id,
        )

        audit_log_module.record(
            session=session,
            action="assign_role",
            actor_id=account.id,
        )
```

このUsecaseでは、Account、Role、AccountRole、AuditLogの秩序を1つの業務意志として束ねています。

## 外部Side Effect

メール送信や決済API呼び出しのような外部Side Effectは、DBトランザクションの途中で実行しません。

```text
database transaction
↓
commit
↓
external side effect
```

これにより、メール送信は成功したがDB commitが失敗してrollbackされた、という単純な不整合を避けます。より強い整合性が必要な場合は、Outbox Patternなどを利用できます。HUMQ自体はOutboxを前提にしません。求めるのは、トランザクション境界とSide EffectがUsecase上で見えることです。

mailer、payment gateway、外部API client などは1テーブルに対応しないため、Moduleではありません。

## 避ける例

### Moduleがcommitする

```python
# modules/account/module.py

def create(session, name: str):
    account = Account(name=name)
    session.add(account)
    session.commit()
    return account
```

この設計では、Usecaseが複数Moduleを1トランザクションにまとめられません。Moduleが勝手に境界を閉じてしまうためです。

### 永続化補助が業務整合性を持つ

```python
# modules/account/repository.py

def create_account_and_role(session, name: str, role_id: int):
    ...
```

Repositoryを切り出す場合でも、それはModule内部の補助です。業務上の組み合わせを置くと、UsecaseとModuleの責務が崩れます。

## HUMQにおける整合性の割り切り

HUMQは、すべての整合性を自動保証する設計ではありません。

代わりに、整合性を見える場所に出します。Usecaseに書かれた処理を見ることで、どの秩序を結び、どの順序で実行し、どの範囲で失敗させるのかが分かります。

これは「整合性を犠牲にして秩序を得る」選択です。整合性を軽視するのではなく、設計者が責任を持って扱う場所を明確にするという意味です。

## QueryとTransaction

Queryは読み取り専用です。Queryはトランザクション境界を決定せず、`commit` や `rollback` を呼びません。

これは、利用しているORMやDB接続の内部でトランザクションが一切存在しないという意味ではありません。HUMQの責務として、Queryがその境界を所有しないという意味です。

---

前へ: [設計原則](03-design-principles.ja.md) | 次へ: [既存アーキテクチャとの比較](05-comparison.ja.md)
