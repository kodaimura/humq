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

Place database-backed shared processing in the owning domain's `_operations.py`.<br>
Keep one file per domain and do not split it into an `operations/` directory.

See [Layer Rules](02-layer-rules.md#operation) for Operation placement and rules.

## Use `_operations.py` as an Adoption-Limit Guide

There is no line-count limit. Judge whether `_operations.py` fits naturally in one file<br>
and remains explainable as support for Usecase.

When several of the following signals appear, stop adding Operations and reconsider the affected domain's design.

- You want to split `_operations.py` into multiple files.
- Unrelated Operations accumulate and cannot be explained as one business area.
- One Operation needs to call another Operation.
- Many Usecases do little more than call Operations in order and then `commit`.
- Primary business flows move from Usecase into Operations.
- Many tables must always be treated as one consistency boundary.

## At the Adoption Limit

Do not add an `operations/` directory or split Operations into smaller Operations and chain them.<br>
First reconsider whether the affected domain boundary is too broad.

When shared invariants become the center of complexity, incrementally migrate only the affected domain<br>
to DDD, aggregate-centered design, or another appropriate design.<br>
Keeping the Handler-called Usecase allows the internal design to change without changing the external API.

---

Previous: [FastAPI Example](06-fastapi-example.md) | Next: [README](../README.md)
