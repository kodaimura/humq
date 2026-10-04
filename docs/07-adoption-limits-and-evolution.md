# Adoption Limits and Evolution

HUMQ's adoption limit is not determined by table count, Usecase count, internal-file count, or lines of code alone.<br>
Judge whether primary business flows remain traceable from Usecase and any extracted processing still supports them.

Internal processing is part of the Usecase responsibility, not an exceptional extra layer.<br>
If it grows, check ownership and readability rather than setting a numerical limit.

## Before Extracting Processing

A long Usecase, similar code, or many Module calls alone do not require extraction.<br>
Separating a policy or other business processing into a file is optional.<br>
When considering it, ask whether the processing has an independent business meaning worth explaining, verifying, and changing.<br>
Pricing, cancellation eligibility, returnable quantity, approval routing, authorization,<br>
and inventory reservation may meet this criterion even with one caller.

Independent business meaning makes extraction possible, not mandatory.<br>
Small local decisions may remain in Usecase. After extraction, its purpose and the primary branch<br>
based on its result must still be visible in Usecase. Whether the processing uses the database<br>
affects how it is implemented and tested, not whether it may be extracted.

## Placement as a Domain Grows

If processing is extracted, one starting point is to place files named for business meaning<br>
directly in the owning domain's Usecase directory:

```text
usecases/
├── orders/
│   ├── cancel.py
│   ├── _cancellation.py
│   └── _pricing.py
└── inventory/
    └── _reservation.py
```

For cross-domain rules, teams can use an existing owning domain, define a domain for the business capability,<br>
or choose a cohesive top-level `policy/` or other shared package. HUMQ does not prescribe one layout.<br>
For example, inventory can own reservation used by both orders and shipping.<br>
Shared use alone does not make unrelated processing a coherent package.

A folder per Usecase or a shared folder for internal processing is a project choice.<br>
Keep the primary flow traceable and avoid a catch-all of unrelated rules.<br>
If files in one domain become difficult to scan, a folder within that domain may help.

For an extracted file, use a business rule or processing name, normally `_<business-rule>.py`.<br>
No class or method naming pattern is required. Do not expose an internal file as a public Usecase.<br>
See [Layer Rules](02-layer-rules.md#internal-business-processing) for data access and transaction rules.

## Signals to Revisit the Design

When several of the following appear, reconsider ownership and boundaries before adding more indirection:

- The business owner or reason to change for an internal file cannot be explained.
- Unrelated processing accumulates in generic shared files.
- Many layers of internal calls or cyclic cross-domain dependencies make the flow hard to follow.
- Usecases do little more than call internal processing in order and then `commit`.
- Primary branches, transaction boundaries, or external I/O disappear from Usecase.
- Many tables must always be treated as one consistency boundary.
- Protecting the same complex shared invariants becomes central to the domain.

These are design signals, not a file-count threshold. Reorganizing files can improve scanning,<br>
but it cannot resolve unclear ownership or a hidden business flow by itself.

## At the Adoption Limit

First reconsider the affected domain boundary and ownership of business processing.<br>
If complex shared invariants are central to that domain, incrementally adopt DDD,<br>
aggregate-centered design, or another appropriate design for that domain.<br>
Keeping the Handler-called Usecase allows the internal design to change without changing the external API.

---

Previous: [FastAPI Example](06-fastapi-example.md) | Next: [README](../README.md)
