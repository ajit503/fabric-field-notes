# Direct Lake on SQL Endpoint + OneLake Security

> **Why this folder?** Securing a Direct Lake on SQL Endpoint model *looks* simple until a report throws `QueryUserError` and you realize model ownership, SSO, delegated vs. user-identity mode, shortcuts, and native tables all pull in different directions. This is the guide I wish I'd had — battle-tested notes, permission matrices, and clear diagrams for getting OneLake security right in Microsoft Fabric.

A field guide to securing **Direct Lake on SQL Endpoint** semantic models — covering **Delegated** vs. **User identity** mode, cross-lakehouse shortcuts, native tables, and SPN-owned models.

## 📖 Read the article

The full write-up is in **[`direct-lake-on-sqlep-onelake-security.md`](direct-lake-on-sqlep-onelake-security.md)**.

## 🗂 Contents

```
direct-lake-on-sqlep-onelake-security/
├── README.md
├── direct-lake-on-sqlep-onelake-security.md   # the article
└── images/
    ├── delegated-mode.png                  # Delegated identity — SELECT inheritance
    ├── user-identity-mode.png              # User identity — OneLake role governance
    ├── delegated-mixed-sources.png         # Delegated mode with shortcut + native tables
    └── user-identity-mixed-sources.png     # User identity — two authorization surfaces
```

## 🔑 Groups used in the examples

| Group | Plane | Role |
|---|---|---|
| `SD-DataAccess_Master_DataSA-RO` | Data plane | Reads the data (OneLake role or SQL `GRANT`) |
| `SD-Fabric-ConsumerAnalyst-Viewer` | Consumption | Opens the report (Build) |

## 🎯 What you'll learn

- Why **model ownership ≠ data access** when SSO is on.
- How **Delegated mode** (SQL-governed) differs from **User identity mode** (OneLake-role-governed).
- How **mixed sources** (shortcut + native tables) create **one** vs. **two** authorization surfaces.
- The **one-to-one Object-ID** rule — and how the redesigned sync relaxes it.
- Why **refresh uses the SPN**, and how to avoid silent refresh failures.
- The **two-group reconciliation** trap and how to avoid it.

## 🏷 Tags

`microsoft-fabric` `onelake` `direct-lake` `power-bi` `data-governance` `sql-analytics-endpoint`
