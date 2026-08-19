# Direct Lake on SQL Endpoint + OneLake Security

> **Why this folder?** Securing a Direct Lake on SQL Endpoint model *looks* simple until a report throws `QueryUserError` and you realize model ownership, SSO, delegated vs. user-identity mode, shortcuts, and native tables all pull in different directions. Then you scale to a multi-domain hub-and-spoke architecture and discover the cross-workspace group mismatch problem. This is the guide I wish I'd had — battle-tested notes, permission matrices, and clear diagrams for getting OneLake security right in Microsoft Fabric.

A field guide to securing **Direct Lake on SQL Endpoint** semantic models and **OneLake shortcuts** in a multi-domain architecture — covering **Delegated** vs. **User identity** mode, cross-lakehouse shortcuts, native tables, SPN-owned models, and **Delegated OneLake Shortcuts** (Preview) for hub-and-spoke gold layers.

## 📖 Read the article

The full write-up is in **[`direct-lake-on-sqlep-onelake-security.md`](direct-lake-on-sqlep-onelake-security.md)**.

## 🗂 Contents

```
direct-lake-on-sqlep-onelake-security/
├── README.md
├── direct-lake-on-sqlep-onelake-security.md   # the article
└── images/
    ├── delegated-mode.png                                              # Delegated identity — SELECT inheritance
    ├── user-identity-mode.png                                          # User identity — OneLake role governance
    ├── delegated-mixed-sources.png                                     # Delegated mode with shortcut + native tables
    ├── user-identity-mixed-sources.png                                 # User identity — two authorization surfaces
    ├── Cross-Workspace Shortcut Security with Different Entra Groups.png  # Passthrough vs. delegated shortcut comparison
    └── Consumer Engines - Where the Delegated-Shortcut Security Holds.png # Engine matrix — where per-user security holds
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
- The **one-to-one Object-ID** rule — and how the redesigned sync relaxes it, and why it was historically broken for different groups across workspace boundaries.
- Why **refresh uses the SPN**, and how to avoid silent refresh failures.
- The **two-group reconciliation** trap and how to avoid it.
- How **Delegated OneLake Shortcuts** (Preview) solve the cross-workspace group mismatch for multi-domain hub-and-spoke architectures.
- The critical **naming trap**: "Delegated shortcut" vs. "Delegated identity mode" — same word, opposite effects.
- Which **engine + config combinations** enforce per-user OneLake security vs. bypass it entirely.
- Where to place **RLS and CLS** in a delegated shortcut topology (producer vs. consumer side).

## 🏷 Tags

`microsoft-fabric` `onelake` `direct-lake` `power-bi` `data-governance` `sql-analytics-endpoint` `onelake-shortcuts` `delegated-shortcuts` `multi-domain`
