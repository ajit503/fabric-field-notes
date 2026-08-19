# Direct Lake on SQL Endpoint + OneLake Security: A Field Guide to Delegated vs. User Identity Mode

*How a single "You don't have permission to view the content of Direct Lake table" error sent me down a rabbit hole — and everything I learned about securing cross-lakehouse shortcuts, native tables, and SPN-owned semantic models in Microsoft Fabric.*

---

> **TL;DR**
> - In **Direct Lake on SQL Endpoint**, owning the model with a service principal does **not** make it the data reader when **SSO** is on — the **viewer's identity** is what gets checked.
> - **Delegated identity mode** = table access is **SQL-governed** (`GRANT SELECT`, SQL-RLS/CLS). OneLake security roles are **ignored**.
> - **User identity mode** (the new default) = table access is governed by **OneLake security roles**; SQL table grants are **blocked**.
> - With **mixed sources** (a shortcut table + native tables in the same lakehouse), delegated mode uses **one** SQL surface, while user identity mode uses **two** authorization surfaces.
> - The redesigned SQL-Endpoint sync **relaxes** the strict one-to-one Object-ID rule and improves nested-group/SPN support — but **verify on your tenant**, because Microsoft Learn still documents the strict behavior.
> - Prior to the redesign, **different Entra groups across the producer/consumer boundary** silently failed — it was not just "strict", it was broken for any cross-domain deployment where consuming groups differed from producer groups.
> - **Delegated OneLake Shortcuts** (Preview) solve the multi-domain scale problem: configure one delegated identity (SPN or workspace identity) on the producer per consuming workspace; each domain manages its own users on the consumer side. Effective access = producer ∩ consumer.

---

## The setup

The architecture is a common hub-and-spoke / producer–consumer pattern:

```
Producer Lakehouse (physical Delta table: gold.daily_taxi_summary)
        │  internal OneLake shortcut
        ▼
Consumer Lakehouse + SQL Analytics Endpoint
        │  Direct Lake on SQL EP
        ▼
Semantic Model  (owner = Service Principal, SSO on)
        │  Build
        ▼
Power BI App  →  Report Viewer
```

Two Entra security groups are involved:

| Group | Plane | Purpose |
|---|---|---|
| `SD-DataAccess_Master_DataSA-RO` | Data plane | Reads the data (OneLake role or SQL `GRANT`) |
| `SD-Fabric-ConsumerAnalyst-Viewer` | Consumption | Opens the report (Build on the app) |

---

## The error that started it all

```
Error fetching data for this visual
You don't have permission to view the content of Direct Lake table.
Underlying Error: QueryUserError
```

The trap: **the semantic model was owned by a service principal, so surely the SPN reads the data?** No.

### Key principle #1 — model ownership ≠ data access

With **SSO on**, Direct Lake reads OneLake using the **report viewer's** identity — **not** the model owner. The SPN owns and refreshes the model, but at query time it's the signed-in user whose permissions are evaluated. Owning the model does nothing for the read path.

So the error meant **the viewer** lacked data-plane access — not the SPN.

---

## Delegated identity mode

**Governing rule:** table access is **SQL-governed** (`GRANT/REVOKE`, SQL-RLS/CLS/DDM). The shortcut hop to the Producer uses the **item owner's identity** (the Consumer Lakehouse owner). **OneLake security roles are ignored** on the SQL EP path.

### Permission matrix (delegated)

| Level | Principal | Grant / Why |
|---|---|---|
| Producer LH | Consumer LH owner | **ReadAll** — the delegated fetch identity that crosses the shortcut |
| Consumer LH + SQL EP | SPN (model owner) | **Item Read + ReadData** — own & refresh the model |
| Consumer LH + SQL EP | Report viewer | **Item Read + `GRANT SELECT`** — the actual read authorization |
| Semantic model | Report viewer | **Build** (via app) — consume the report |
| Producer LH | SPN / viewers | None required |

### Key principle #2 — SELECT can be inherited via a group

In my case the viewer succeeded **without** an explicit grant on `SD-Fabric-ConsumerAnalyst-Viewer`, because the viewer was **also** a member of `SD-DataAccess_Master_DataSA-RO`, which held `GRANT SELECT`. SQL honors Entra **group** grants, so the SELECT was **inherited** through group membership.

![Delegated identity mode — SELECT inheritance](https://raw.githubusercontent.com/ajit503/fabric-field-notes/main/direct-lake-on-sqlep-onelake-security/images/delegated-mode.png)
*Delegated mode: table access is SQL-governed. The viewer inherits `GRANT SELECT` through the `SD-DataAccess_Master_DataSA-RO` group; OneLake roles are ignored.*

### SQL grant / rollback

```sql
-- On the Consumer LH SQL analytics endpoint
GRANT SELECT ON [gold].[daily_taxi_summary] TO [SD-DataAccess_Master_DataSA-RO];

-- Rollback
REVOKE SELECT ON [gold].[daily_taxi_summary] FROM [SD-DataAccess_Master_DataSA-RO];
```

> ⚠️ Grant on the **Consumer** EP, never the Producer. If you run it on the Producer by mistake, `REVOKE` it on that same (Producer) endpoint.

---

## Then Microsoft changed the default

From the *New OneLake security improvements for Microsoft Fabric* announcement:

> "We've completely redesigned how SQL Analytics Endpoint integrates with OneLake security. The new design has lower latency, works better with nested security groups, and solves existing limitations with service principals. As part of this change, the default mode for SQL Analytics Endpoint's is now **User's identity mode**."

This reframes everything.

---

## User identity mode

**Governing rule:** the SQL EP passes the **signed-in user's identity** to OneLake; table reads are governed by **OneLake security roles**. **SQL `GRANT/REVOKE` on tables is not allowed.** Non-table objects (views, procs, functions) still use SQL permissions.

### The one-to-one identity mapping (the classic gotcha)

Historically, User identity mode required a **strict one-to-one mapping**:

1. **Fabric Read on the Consumer** — the exact principal in the OneLake role must be *directly* granted Read on the Consumer lakehouse (role membership alone is not enough).
2. **Object ID must match** at Producer ↔ Consumer — nested/effective group membership isn't resolved across that boundary.
3. **Artifact Read** on the SQL EP — always required.

> 🟢 **The redesign relaxes this** — the newest update reduces the cross-lakehouse one-to-one friction and improves nested-group + SPN support (including SPN-owned lakehouses). But **Microsoft Learn still documents the strict behavior**, so treat the relaxation as *rolling out* and verify on your tenant.

> **Historical context:** The cross-workspace Object-ID requirement was not just "strict" — groups with *different* Object IDs across the producer/consumer boundary were outright broken. If `SG-Finance-Consumers` (Object ID `2222`) was not directly on a producer role that referenced `SG-MDM-Readers` (Object ID `1111`), consumers got access-denied even through logically valid nesting. The symptom: Lakehouse Explorer access worked fine, but the SQL Analytics Endpoint returned permission-denied for shortcut tables. Adding the individual account directly to the producer role resolved it — exactly what you cannot scale. The redesign improves this, but **Delegated OneLake Shortcuts** (covered in the next section) are the architectural fix for multi-domain hubs.

### "Will a user in BOTH groups work?"

**Yes.** A viewer in both `SD-Fabric-ConsumerAnalyst-Viewer` (Build) and `SD-DataAccess_Master_DataSA-RO` (OneLake role + Consumer Read) renders the visual — guaranteed regardless of rollout state.

### Permission matrix (user identity)

| Level | Principal | Grant / Why |
|---|---|---|
| Producer LH | `SD-DataAccess_Master_DataSA-RO` | Member of a OneLake security role attaching the table (Read) |
| Consumer LH | `SD-DataAccess_Master_DataSA-RO` | **Direct Fabric Read** + artifact Read (Object-ID match) |
| Consumer SQL EP | — | Access mode = **User identity** (new default). No SQL table GRANT |
| Semantic model | SPN | Owner — own/refresh (SPN-owned lakehouses now supported) |
| Power BI App | `SD-Fabric-ConsumerAnalyst-Viewer` | **Build** — consume the report |

![User identity mode — OneLake role governance](https://raw.githubusercontent.com/ajit503/fabric-field-notes/main/direct-lake-on-sqlep-onelake-security/images/user-identity-mode.png)
*User identity mode: the signed-in viewer's identity is passed to OneLake, and data access is authorized by the OneLake role via `SD-DataAccess_Master_DataSA-RO`. The viewer must be in both groups.*

---

## Cross-workspace group mismatch — the multi-domain problem

The Object-ID requirement above creates a scaling problem in any hub-and-spoke gold layer where each consuming domain has its own Entra group.

**The scenario:** A producer lakehouse (e.g., MDM) has a OneLake Security role referencing `SG-MDM-Readers` (Object ID `1111`). A Finance consumer workspace is governed by `SG-Finance-Consumers` (Object ID `2222`). With passthrough shortcuts and User identity mode, Fabric performs a literal Object-ID match across the producer/consumer boundary — `2222` does not match `1111`, so Finance users are denied even if they should have access.

Adding every consumer group directly to the producer does not scale across Finance, Supply Chain, Commercial, and more — each with their own groups.

### Delegated OneLake Shortcuts (Preview)

Delegated shortcuts break the identity passthrough at the shortcut boundary. The producer evaluates a single **delegated connection identity** — an SPN or workspace identity — instead of the end user. The result:

- The producer only needs to trust **one identity per consuming workspace**.
- Each consuming domain manages its **own OneLake Security roles** on the consumer side.
- Effective access is the **intersection** of both sides (both must allow).

```
Producer Lakehouse (MDM)
  OneLake role → SPN-Finance  (delegated identity)
        │  Delegated OneLake shortcut
        ▼
Consumer Lakehouse (Finance)
  OneLake role → SG-Finance-Consumers  (end user evaluated here)
        │  SQL EP / Spark / Direct Lake
        ▼
Report Viewer
```

Adding Supply Chain later means granting `SPN-SupplyChain` on the producer once — no changes to Finance or any other domain.

![Cross-workspace shortcut security with different Entra groups](https://raw.githubusercontent.com/ajit503/fabric-field-notes/main/direct-lake-on-sqlep-onelake-security/images/Cross-Workspace%20Shortcut%20Security%20with%20Different%20Entra%20Groups.png)
*Passthrough (left): strict Object-ID match across the boundary fails when groups differ. Delegated (right): producer trusts one SPN; Finance owns its consumer-side role. Effective access = producer ∩ consumer.*

### Effective access is an intersection

| Layer | Identity evaluated | Purpose |
|---|---|---|
| Consumer-side role on shortcut | End user | Per-user control; CLS supported |
| Producer-side role | Delegated identity (SPN / WI) | Grant once; producer RLS = domain slice |
| **Effective access** | **Producer ∩ Consumer** | Both must allow — consumer can only restrict further |

> 💡 **SPN or workspace identity** — both work as the delegated identity. Prefer workspace identity to avoid SPN secret rotation unless existing SPN governance is in place.

### RLS and CLS placement

| Goal | Where to define | Supported today? |
|---|---|---|
| Show only this domain's rows | RLS on **producer** side | ✅ Yes |
| Show each user their own rows | RLS on **consumer** side | ❌ Not yet — use semantic-model RLS (Power BI) as interim |
| Column-level filtering | CLS on **either** side | ✅ Yes (both sides) |

### Microsoft PM confirmation

The intended pattern was confirmed by the Microsoft OneLake Security PM:

> *"Yes, this is the intended pattern. It does not have to be a workspace identity — it can be a regular SPN. Once that's done: access to the producer is evaluated as the delegated identity. Only that identity needs access to the producer. You can then optionally set OneLake security on the consumer side as well, to manage end user access. The effective result is an intersection between both OneLake security roles.*
>
> *You can define RLS on the producer side if your goal is 'only show this domain's data'. It must be set on the consumer side if your goal is 'show each user their own rows'. Note that we only support RLS on the producer side at the moment, but CLS is supported on both sides.*
>
> *Yes, this is why we released delegated shortcuts — customers want to limit end user access to their central lakehouse."*

---

## ⚠️ Naming trap: "Delegated shortcut" ≠ "Delegated identity mode"

These two features share the word *delegated* but have **opposite effects on end-user security**:

| Term | What it is | Effect on end-user identity |
|---|---|---|
| **Delegated OneLake Shortcut** | Shortcut-level feature (Preview) | Breaks identity passthrough at the shortcut; **enables** consumer-side per-user roles |
| **Delegated identity mode** (SQL EP) | SQL Analytics Endpoint access mode | Terminates end-user identity at the endpoint; OneLake roles **ignored**, SQL GRANT governs |

One *scales* access across domains. The other *bypasses* per-user OneLake enforcement entirely. The consequences are silent — no error message, just wrong access behavior.

---

## Engine matrix — where per-user OneLake security holds

When using delegated shortcuts, the consumer-side role (evaluated as the end user) only holds if the engine passes the user's identity through to OneLake. Some engine configurations terminate the identity and bypass per-user enforcement.

| Engine | Configuration | Per-user OneLake security holds? |
|---|---|---|
| SQL Analytics Endpoint | User identity mode | ✅ Yes |
| SQL Analytics Endpoint | Delegated identity mode | ❌ SQL GRANT/REVOKE governs |
| Spark / Spark SQL | Default passthrough | ✅ Yes |
| Direct Lake on OneLake | SSO on (recommended) | ✅ Yes |
| Direct Lake on OneLake | Fixed identity (SSO off) | ❌ Model-level RLS only |
| Direct Lake on SQL EP | Endpoint = User identity mode | ✅ Yes |
| Direct Lake on SQL EP | Endpoint = Delegated identity mode | ❌ Endpoint RLS/CLS/OLS only |

![Consumer engines — where delegated-shortcut security holds](https://raw.githubusercontent.com/ajit503/fabric-field-notes/main/direct-lake-on-sqlep-onelake-security/images/Consumer%20Engines%20-%20Where%20the%20Delegated-Shortcut%20Security%20Holds.png)
*Green = per-user OneLake Security enforced; red = identity terminates and per-user OneLake is bypassed. The naming trap is called out at the bottom of the diagram.*

> 💡 **Direct Lake on OneLake** (reads Delta files directly via OneLake APIs) is the recommended flavor for a cross-domain gold layer. It can span multiple lakehouses/workspaces and with SSO on, the effective identity is always the end user — no SQL EP in the path.

---

## Mixed sources: shortcut **and** native tables

This is where the two modes diverge most. Suppose the Consumer Lakehouse holds:

- **(A)** a **shortcut** table → `daily_taxi_summary` (data physically in the **Producer**, `gold.daily_taxi_summary`)
- **(B)** a **native** Delta table → `silver.trip_events` (data physically in the **Consumer**)

### Key principle #3 — where the data lives = where security is defined

| Table | Physical data | Governed on | Object-ID mapping? |
|---|---|---|---|
| (A) Shortcut | Producer | **Producer** (Role #1) | **Yes** — cross-lakehouse match |
| (B) Native | Consumer | **Consumer** (Role #2) | **No** — co-located |

### What this means per mode

**Delegated mode — one surface.** Both tables are queried through the same SQL EP, so a **single `GRANT SELECT`** on the Consumer EP authorizes both. Only the shortcut needs the owner's ReadAll on the Producer.

```sql
-- Delegated: one surface covers both
GRANT SELECT ON [gold].[daily_taxi_summary] TO [SD-DataAccess_Master_DataSA-RO];
GRANT SELECT ON [silver].[trip_events]      TO [SD-DataAccess_Master_DataSA-RO];
```

![Delegated mode with shortcut and native tables](https://raw.githubusercontent.com/ajit503/fabric-field-notes/main/direct-lake-on-sqlep-onelake-security/images/delegated-mixed-sources.png)
*Delegated mode with mixed sources: one SQL GRANT surface on the Consumer EP authorizes both the shortcut (A) and native (B) tables. Only the shortcut fetch reaches the Producer via the owner identity.*

**User identity mode — two surfaces.** The shortcut is governed by a OneLake role on the **Producer**; native tables by a OneLake role on the **Consumer**. A viewer must be authorized on **both**, or you get **partial failure**:

> ⚠️ **Partial failure:** a viewer in the Consumer role only will see native visuals render while the shortcut visual throws `QueryUserError` — on the same report page.

> ⚠️ **DefaultReader caveat:** enabling OneLake security on the Consumer governs its native tables too. `DefaultReader` preserves prior access; tightening it can silently lock out native tables unless you add the group to the Consumer role.

![User identity mode with two authorization surfaces](https://raw.githubusercontent.com/ajit503/fabric-field-notes/main/direct-lake-on-sqlep-onelake-security/images/user-identity-mixed-sources.png)
*User identity mode with mixed sources: the shortcut (A) is authorized by Role #1 on the Producer; native tables (B) by Role #2 on the Consumer. One model reads both surfaces — miss one and that table's visuals fail while others render.*

---

## Refresh behavior (don't get caught)

A Direct Lake refresh is a **reframe** that reads Delta metadata. At refresh time there's **no interactive user** — the **SPN owner is the reader**.

- **Delegated:** SPN needs **Item Read + ReadData** on the Consumer EP; the owner identity still fetches the shortcut from the Producer.
- **User identity:** the SPN must be in the **OneLake role(s)** (Producer for shortcut, Consumer for native) with direct Consumer Read — or the corresponding table **fails on scheduled refresh** even though user queries work.

> 🪤 Common trap: reports render for users (identity passthrough) but **scheduled refresh fails** because the SPN owner was never added to the OneLake role.

---

## Delegated vs. User identity — side by side

| Dimension | Delegated identity | User identity (new default) |
|---|---|---|
| Table access governed by | SQL `GRANT/REVOKE` (+ SQL-RLS/CLS) | OneLake security roles (SQL table grants blocked) |
| Authorization surfaces (mixed) | **One** (Consumer EP) | **Two** (Producer + Consumer roles) |
| Shortcut fetch identity | Item owner (Consumer LH owner) | Signed-in viewer (passthrough) |
| Object-ID one-to-one | N/A | Required for shortcut (relaxing, rolling out) |
| RLS / CLS lives in | SQL engine | OneLake role (RLS/CLS/OLS) |
| Aligns with Spark | No (SQL-only) | Yes (enforced everywhere) |
| Best when | Classic DBA / SQL-specific security | Centralized "define once, enforce everywhere" |

---

## The operational risk you'll actually hit: group reconciliation

Because **data access** (`SD-DataAccess_Master_DataSA-RO`) and **app access** (`SD-Fabric-ConsumerAnalyst-Viewer`) ride on **two separate groups**, their memberships can silently drift:

- Added to the **app** group but not the **data** group → report opens, every visual = `QueryUserError`.
- In the **data** group but not the **app** group → can't open the report at all.

It's the primary risk because it's **frequent** (membership changes constantly), **silent** (nothing errors at config time), and **user-facing** (it produces the exact error end users see).

**Fixes:** collapse to one group, nest one inside the other, or automate the sync (dynamic groups).

---

## Validation checklist

1. Confirm the Consumer SQL EP access mode (**Delegated** vs. **User identity**).
2. **Delegated:** `GRANT SELECT` to the data group on **both** tables; verify owner **ReadAll** on the Producer (shortcut).
3. **User identity:** group in the **Producer role** (shortcut) *and* **Consumer role** (native); direct Consumer Read + Object-ID match.
4. Add the **SPN** to the required role(s) so scheduled refresh succeeds.
5. **Test each source with its own visual** to catch partial failures.
6. Verify **DefaultReader** handling on the Consumer before tightening.
7. **Negative test:** a user in only the app group should fail — confirms group separation.

---

## Takeaways

- **Model ownership ≠ data access.** With SSO, the viewer's identity is the reader.
- **Know your mode.** Delegated = SQL-governed & uniform; User identity = OneLake-governed & centralized.
- **Mixed sources multiply surfaces.** One SQL grant (delegated) vs. two OneLake roles (user identity).
- **Refresh uses the SPN.** Authorize the SPN explicitly, or scheduled refresh breaks silently.
- **Reconcile your groups** — or better, collapse them.
- **Verify the redesign on your tenant** before relying on relaxed nesting/Object-ID behavior.
- **Cross-workspace group mismatch does not scale with passthrough shortcuts** — delegated shortcuts are the architectural answer for multi-domain hubs.
- **"Delegated shortcut" ≠ "Delegated identity mode"** — one enables per-user consumer roles on the shortcut; the other bypasses per-user OneLake enforcement entirely.

---

*Written by Ajit Kumar Singh — Fabric Admin & Architect. Based on hands-on troubleshooting of Direct Lake on SQL Endpoint with OneLake security across producer/consumer lakehouses.*

*Have a different experience with the new User-identity default? I'd love to hear how the rollout is behaving on your tenant.*
