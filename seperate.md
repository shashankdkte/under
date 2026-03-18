## Why “Same vs Separate AD Groups for OLS and RLS” Makes Sense in V1 but Not in V2

This section explains in detail why the question *“If we have the same AD group for OLS and RLS we have no issue; what if we have separate AD groups for OLS and RLS?”* is meaningful in **Sakura V1** but **does not apply** in **Sakura V2**.

---

### V1: One Mechanism for Both OLS and RLS (Group-Driven)

In V1, **both** “can the user open the app?” (OLS) and “which rows can they see?” (RLS) were enforced by **the same thing**: **Azure AD group membership**.

```mermaid
flowchart TB
    subgraph V1_Storage["V1: Sakura database"]
        A1[Approved requests<br/>Orga / Cost Center / MSS]
        A2[dbo.RDSecurityGroupPermission<br/>VIEW: RequestedFor, SecurityGroupGUID]
        A1 --> A2
    end

    subgraph V1_Sync["V1: Single sync"]
        B1[SakuraADSync.ps1]
        B2[Reads RDSecurityGroupPermission]
        B3[For each group: add/remove users in Entra]
        B2 --> B1
        B1 --> B3
    end

    subgraph V1_PowerBI["V1: Power BI / downstream"]
        C1[User in group G]
        C2[Group G = app access AND data scope]
        C3[One group = both OLS and RLS]
        C1 --> C2
        C2 --> C3
    end

    A2 -->|"desired (user, group)"| B2
    B3 -->|"actual membership"| C1
```

**What this means:**

1. **One view:** `RDSecurityGroupPermission` produced rows like `(RequestedFor, SecurityGroupGUID)`. That view was built from approved requests (Orga, Cost Center, MSS). There was **no separate** “OLS table” vs “RLS table” for sync — one view encoded “this user should be in this group.”
2. **One sync script:** `SakuraADSync.ps1` read that view and updated Entra so that membership matched. So **one pipeline** fed **all** security groups used for access.
3. **One meaning per group:** In Power BI (or the semantic model), **each group** effectively meant both:
   - **OLS:** “User can open this app/report” (because the app was configured to allow that group).
   - **RLS:** “User sees this data scope” (because the report’s RLS rules were written to use **group membership** — e.g. “if user is in #SG-UN-SAKURA-FIN then show rows where Entity = X”).

So in V1, **OLS and RLS were not separate pipelines** — they were **two uses of the same AD group membership**.

```mermaid
flowchart LR
    subgraph Same_Group["If SAME group for OLS and RLS (V1 typical)"]
        S1[Sync adds user to Group G]
        S2[Power BI: Group G → can open app]
        S3[Power BI: Group G → see rows for scope A]
        S1 --> S2
        S1 --> S3
    end

    subgraph Separate_Groups["If SEPARATE groups (V1 hypothetical)"]
        P1[Sync must add user to Group OLS]
        P2[Sync must add user to Group RLS]
        P3[Power BI: Group OLS → can open app]
        P4[Power BI: Group RLS → see rows]
        P1 --> P3
        P2 --> P4
    end
```

**Why the question makes sense in V1:**

- **Same group:** One sync run adds the user to one group; that group is used for both “can open” and “which rows.” No gap.
- **Separate groups:** You would need the sync to add the user to **both** “OLS group” and “RLS group.” If the view or script only fed one of them, the user would have app access but no data (or the reverse). So the question “what if we have separate AD groups for OLS and RLS?” is exactly about: *we must ensure both groups get the right members from the same source of truth.*

So in V1, the question is **valid and important**: it’s about whether one group carries both meanings or two groups do, and in the latter case, ensuring the **same** sync/view populates **both**.

---

### V2: Two Completely Separate Mechanisms (OLS = Groups, RLS = Views)

In V2, OLS and RLS use **different mechanisms**. Only OLS uses AD groups; RLS does **not** use AD groups at all.

```mermaid
flowchart TB
    subgraph V2_OLS["V2: OLS path (AD groups)"]
        O1[Approved OLS]
        O2[OLSPermissions → AppAudiences]
        O3[Auto.OLSGroupMemberships<br/>RequestedFor, EntraGroupUID]
        O4[SakuraV2ADSync.ps1]
        O5[User added to Entra group]
        O6[Power BI: group → can open app]
        O1 --> O2 --> O3 --> O4 --> O5 --> O6
    end

    subgraph V2_RLS["V2: RLS path (NO AD groups)"]
        R1[Approved RLS]
        R2[RLSPermissions + domain detail tables]
        R3[ShareAMER.RLS, ShareEMEA.RLS, ...]
        R4[Power BI / Fabric reads views]
        R5[Filter rows by user + dimension keys]
        R1 --> R2 --> R3 --> R4 --> R5
    end

    Request[Permission request] --> O1
    Request --> R1
```

**What this means:**

1. **OLS:** Stored in `OLSPermissions` (and related tables). The **only** place that drives AD group membership is `Auto.OLSGroupMemberships`. The sync script reads **only** that view and updates **only** Entra. So in V2, **only OLS** is “group-driven.”
2. **RLS:** Stored in `RLSPermissions` and domain detail tables (e.g. `RLSPermissionEMEADetails`). It is exposed to downstream via **Share*.RLS views**. Power BI (or Fabric) reads those views (or tables built from them) and applies row filters by **user identity + dimension keys**. There is **no** “RLS group” and **no** sync that adds users to an “RLS group.”
3. So in V2 there is **no** “same AD group for OLS and RLS” vs “separate AD groups for OLS and RLS” — because **RLS does not use any AD group** in the design.

```mermaid
flowchart TB
    subgraph Reality_V2["V2 reality"]
        direction TB
        A[OLS] --> B[Uses AD groups<br/>Auto.OLSGroupMemberships → Sync]
        C[RLS] --> D[Uses Share*.RLS views<br/>No AD group, no sync]
        B -.->|"not used for RLS"| E[Entra groups]
        D --> F[Power BI reads views]
    end

    subgraph Question_Assumption["What the question assumes"]
        Q[Two kinds of AD groups:<br/>one for OLS, one for RLS]
    end

    Question_Assumption -.->|"does not exist in V2"| Reality_V2
```

**Why the question does not apply in V2:**

- There is **no** “RLS AD group” maintained by Sakura. The sync script does **not** add users to any group for RLS.
- So you **cannot** have “the same AD group for OLS and RLS” in the V2 sense — OLS has groups, RLS has views.
- You also **cannot** have “separate AD groups for OLS and RLS” in the V2 design — there is only one set of groups (OLS), and RLS is handled entirely outside the group/sync pipeline.

The question implicitly assumes **both** OLS and RLS are enforced via AD groups (same or different). That is true in V1; it is **false** in V2 for RLS.

---

### Side-by-Side Summary (Mermaid)

```mermaid
flowchart TB
    subgraph V1_Flow["V1: Single pipeline"]
        V1A[Approved request] --> V1B[RDSecurityGroupPermission]
        V1B --> V1C[Sync script]
        V1C --> V1D[User in AD group]
        V1D --> V1E[Power BI uses SAME group<br/>for OLS and RLS]
    end

    subgraph V2_Flow["V2: Two pipelines"]
        V2A[Approved request] --> V2B_OLS[OLS → Auto.OLSGroupMemberships]
        V2A --> V2B_RLS[RLS → RLSPermissions / Share*.RLS]
        V2B_OLS --> V2C[Sync → AD group]
        V2B_RLS --> V2D[No sync — views only]
        V2C --> V2E[Power BI: group = open app]
        V2D --> V2F[Power BI: view = row filter]
    end

    V1_Flow ~~~ V2_Flow
```

| | V1 | V2 |
|---|----|----|
| **OLS** | AD group (from RDSecurityGroupPermission) | AD group (from Auto.OLSGroupMemberships + sync) |
| **RLS** | **Same AD group** (group membership = data scope) | **No AD group** — Share*.RLS views, read by Power BI |
| **Sync script** | One script updates all groups used for both OLS and RLS | One script updates **only** OLS groups |
| **“Same vs separate AD groups for OLS and RLS?”** | **Makes sense** — both go through groups; same = one group, separate = two groups to populate | **Does not apply** — RLS has no AD group in the design |

---

### One-Sentence Takeaway

- **V1:** OLS and RLS were both enforced by **group membership**, so the question “same group vs separate groups?” is about how many groups you use and whether the sync fills both.
- **V2:** OLS is enforced by **group membership**; RLS is enforced by **views** (Share*.RLS). There is no “RLS group,” so the question does not apply.
