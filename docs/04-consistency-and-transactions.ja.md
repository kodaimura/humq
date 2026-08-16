# 整合性とトランザクション

HUMQでは、整合性とトランザクション境界をUsecaseの責務として扱います。重要なのは、どこで`commit`するかだけでなく、何を同時に成功させ、何を別の失敗として扱うかがUsecaseから見えることです。

## 基本ルール

- アプリケーション上のトランザクション境界を所有できるのはUsecaseだけとする。
- 原子的に扱うDB書き込みは、原則として1つのトランザクションにまとめる。
- Moduleは`commit`や`rollback`を呼ばない。
- Repositoryを切り出す場合も、`commit`や`rollback`を呼ばない。
- 複数Moduleをまたぐ整合性はUsecaseに明示する。
- Queryは読み取り専用であり、トランザクション境界を所有しない。
- 外部Side EffectをDBトランザクションの途中で実行しない。

## なぜUsecaseに置くのか

トランザクションは、複数の秩序を一時的に束ねる操作です。

Moduleは正確に1テーブルに閉じた秩序です。Moduleがトランザクションを管理すると、そのModuleを使うUsecase全体の整合性を外から制御しにくくなります。

Usecaseがトランザクションを持つことで、次のことが明確になります。

- どの操作が1つの業務単位なのか。
- どのModuleを組み合わせているのか。
- どの範囲を成功または失敗として扱うのか。
- どの整合性を意図的に守っているのか。

`Usecase = Transaction`という1対1対応を強制するわけではありません。読み取りだけのUsecaseは明示的なトランザクションを必要としない場合があります。また、外部システムを含む長い業務処理では、複数のDBトランザクションと補償処理が必要になる場合があります。

その場合も、それぞれの境界、Side Effect、失敗後の方針をUsecaseから追えるようにします。transaction decoratorなどを使う場合も、どのUsecaseのどの範囲を囲むかが宣言から明確でなければなりません。

## Aggregateを中心にした設計との違い

DDDのAggregateは、Aggregate境界内の不変条件を守る有力な方法です。Application Serviceが複数Aggregateを調整する設計も可能であり、DDDが必ず業務フローを隠すわけではありません。

HUMQとの違いは既定の境界です。

| 観点 | Aggregateを中心にした設計 | HUMQ |
| --- | --- | --- |
| 局所的な整合性 | Aggregate内部に表現する | 1テーブルのModuleに表現する |
| 横断的な整合性 | Application層などで調整する | Usecaseに明示する |
| 下位部品の境界 | Domain Modelの判断で決める | 1 tableで機械的に決める |
| 主なトレードオフ | 強い不変条件、境界設計の難しさ | 高い追跡可能性、自動保護の少なさ |

HUMQは、自動的な保護よりも、操作対象と整合性の追跡可能性を優先します。

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

このUsecaseでは、Account、Role、AccountRole、AuditLogに対する操作と、その原子的な範囲が1箇所から見えます。

## 外部Side Effect

メール送信や決済API呼び出しのような外部Side Effectは、DBトランザクションの途中で実行しません。

```text
database transaction
↓
commit
↓
external side effect
```

これにより、メール送信は成功したがDB commitが失敗してrollbackされた、という不整合を避けます。ただし、commit後にメール送信が失敗する可能性は残ります。順番を入れ替えるだけでは、DBと外部システムを原子的には扱えません。

Usecaseでは、必要な保証に応じて失敗後の方針を選びます。

| 状況 | 方針 |
| --- | --- |
| 通知失敗を許容できる | DB commit後に送信し、失敗を記録する |
| 再送が必要 | 冪等な送信とretryを設計する |
| DB更新と配信要求を失いたくない | Outbox Patternを使う |
| 決済など外部状態も変わる | 冪等性、Saga、補償処理を設計する |

Outboxを使う場合、Usecaseは通常のModuleを通してoutboxテーブルへ同一トランザクション内で書き込み、別のworkerが配送します。HUMQ自体はOutboxを必須としません。求めるのは、トランザクション境界、Side Effect、失敗時の保証がUsecaseから追えることです。

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

これは自動的・暗黙的な保護より、明示的な整合性と追跡可能性を選ぶトレードオフです。整合性を軽視するのではなく、設計者が責任を持って扱う場所を明確にするという意味です。

## QueryとTransaction

Queryは読み取り専用です。Queryはトランザクション境界を決定せず、`commit` や `rollback` を呼びません。

これは、利用しているORMやDB接続の内部でトランザクションが一切存在しないという意味ではありません。HUMQの責務として、Queryがその境界を所有しないという意味です。

---

前へ: [設計原則](03-design-principles.ja.md) | 次へ: [既存アーキテクチャ・設計パターンとの比較](05-comparison.ja.md)
