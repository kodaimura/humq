# Adoption Limits and Evolution

HUMQ's adoption limit is not determined by table count, Usecase count, or lines of code alone.<br>
Judge whether primary business flows remain traceable from Usecase and shared processing remains supportive.

Database-backed shared processing may be placed in Operation, so the existence of Operation is not itself an adoption limit.

## Before Sharing Processing

Similar processing, a long Usecase, or many Module calls alone do not require sharing.

Before extracting an Operation, verify that it:

- Is genuinely used by multiple Usecases.
- Has the same business meaning and changes for the same reason.
- Must share the same validation, errors, locking, and update order.
- Leaves the primary flow traceable from Usecase after extraction.

When these conditions are not met, duplication is an intentional choice that protects traceability.

## Use Operation as the Normal Sharing Mechanism

Place database-backed shared processing in the owning domain's `_operations.py` first.<br>
When one file becomes difficult to read, split it into multiple files directly under the same domain,<br>
grouped by business capability or reason to change.

```text
usecases/
└── procurement/
    ├── create_order.py
    ├── receive_goods.py
    ├── _policies.py
    ├── _authorization_operations.py
    └── _numbering_operations.py
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

There is no limit based on lines or number of files.<br>
Splitting Operations across multiple files is not itself an adoption limit.<br>
Judge whether their business ownership remains explainable and they remain supportive of the primary flow in Usecase.

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

When shared invariants become the center of complexity, incrementally migrate only the affected domain<br>
to DDD, aggregate-centered design, or another appropriate design.<br>
Keeping the Handler-called Usecase allows the internal design to change without changing the external API.

---

Previous: [FastAPI Example](06-fastapi-example.md) | Next: [README](../README.md)
