# Adoption Limits and Evolution

HUMQ's adoption limit is not determined by table count, Usecase count, or lines of code alone.<br>
Judge whether primary business flows remain traceable from Usecase and shared processing remains supportive.

Operation is an exception used only when the implementation of the same invariant cannot be allowed to diverge.<br>
One Operation does not itself mark the adoption limit, but growing numbers of them are a signal to revisit the design.

## Before Sharing Processing

Similar processing, a long Usecase, or many Module calls alone do not require sharing.

Before extracting an Operation, verify that it:

- Is genuinely used by multiple Usecases.
- Protects the same invariant and cannot be allowed to diverge.
- Has a concrete inconsistency that can be named as the consequence of violating the invariant.
- Must share the same validation, errors, locking, and update order.
- Leaves the primary flow traceable from Usecase after extraction.

When these conditions are not met, duplication is an intentional choice that protects traceability.

## Use Operation as an Exception

Place the processing in the owning domain's `_operations.py` only when the criteria above are satisfied.<br>
When one file becomes difficult to read, split it into multiple files directly under the same domain,<br>
grouped by business capability or reason to change.

```text
usecases/
└── procurement/
    ├── create_order.py
    ├── receive_goods.py
    ├── _policies.py
    ├── _reservation_operations.py
    └── _billing_operations.py
```

Use `_<business-capability>_operations.py` as the naming pattern.<br>
Do not group files by generic or technical categories such as `_helpers.py`, `_common_operations.py`,<br>
`_database_operations.py`, or `_misc_operations.py`, and do not split mechanically into one file per class.

After splitting, the owning domain must remain clear, references must be easier to trace from Usecase,<br>
and the primary flow must remain visible. Do not re-export Operation files through `__init__.py`,<br>
and do not create root-level `usecases/_operations.py` or `usecases/<domain>/operations/`.<br>
Even when several domains use it, Operation remains in the domain that owns the business capability.

See [Layer Rules](02-layer-rules.md#operation) for Operation placement and rules.

## Use Operation Growth as an Adoption-Limit Guide

Lines or file count alone do not set a limit, but Operation is an exception;<br>
do not treat its growth as normal. Even when splitting Operations across files for readability,<br>
verify that they remain supportive of the primary flow in Usecase.

When several of the following signals appear, stop adding Operations and reconsider the affected domain's design.

- The owning domain or reason to change for an Operation cannot be explained.
- Unrelated shared processing accumulates into a de facto generic Service.
- Calls or dependency chains among Operations must be traced.
- Many Usecases do little more than call Operations in order and then `commit`.
- Primary business flows or branches move from Usecase into Operations.
- Many tables must always be treated as one consistency boundary.
- Protecting the same complex shared invariants becomes central to the domain.

## At the Adoption Limit

Do not hide a structural problem by only reorganizing Operation files.<br>
First reconsider the affected domain boundary and ownership of shared processing.

By keeping one invariant implementation in one place, Operation can provide part of the role of an Aggregate.<br>
It prevents divergence and partial-update omissions across Usecases. When Operations continue to grow,<br>
or complex shared invariants become central to the affected domain, incrementally migrate only that domain<br>
to DDD, aggregate-centered design, or another appropriate design.<br>
Keeping the Handler-called Usecase allows the internal design to change without changing the external API.

---

Previous: [FastAPI Example](06-fastapi-example.md) | Next: [README](../README.md)
