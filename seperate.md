# OLS and RLS Use Cases and Diagrams

This document describes how **OLS** (Object-Level Security) and **RLS** (Row-Level Security) work in Sakura, with Mermaid diagrams for each use case.

---

## 1. What OLS and RLS Give

| | OLS | RLS |
|---|-----|-----|
| **What the request gives** | Access to the **object** (app, audience, or report) | Access to **data** (which rows the user can see) |
| **Meaning** | "Can they open it?" | "What rows can they see inside it?" |

- **OLS** = which apps/audiences/reports the user can open.
- **RLS** = which rows (Entity, Client, PC, etc.) the user sees inside the dataset.

### Important clarifications

- **OLS = access to object only, not data.** OLS does not define or control which rows the user sees; it only controls whether they can open the app/report/audience. When we say "no RLS = they see all data," that means: they already have object access (OLS), and because no RLS filter is applied, the app/report shows all rows — so "all data" is due to the *absence* of a row filter, not because OLS "gives" data.

- **RLS is across the workspace/dataset, not tied to one report.** RLS is defined per workspace security model and applies to the dataset. Any report or app in that workspace that uses that dataset sees the same RLS filter for that user. So RLS applies **across** all reports/apps using that dataset, not only to one particular report.

---

## 2. Use Case: OLS Only (No RLS)

**What’s approved:** OLS only. No RLS permission.

**Result:** User can open the app/report and sees **all data** in that object (no row filter).

```mermaid
flowchart LR
    A[Request] --> B[OLS approved]
    B --> C[User in Entra group]
    C --> D[Can OPEN app/report]
    D --> E[Sees ALL data - no RLS filter]
```

---

## 3. Use Case: RLS Only (No OLS)

**What’s approved:** RLS only. No OLS (no audience/report/Entra group).

**Result:** User has a row filter defined but **cannot open** the app/report (no group membership). RLS is useless until they also get OLS.

```mermaid
flowchart LR
    A[Request] --> B[RLS approved]
    B --> C[Share*.RLS has user + keys]
    C --> D[No Entra group - cannot open app]
    D --> E[No access in practice]
```

---

## 4. Use Case: Both OLS and RLS

**What’s approved:** OLS (access to app/report) and RLS (which rows they can see).

**Result:** User can open the app and sees **only** the rows allowed by RLS.

```mermaid
flowchart LR
    A[Request] --> B[OLS + RLS approved]
    B --> C[User in group + RLS keys in Share*.RLS]
    C --> D[Can OPEN app]
    D --> E[Sees ONLY allowed rows]
```

---

## 5. Summary: All Three Cases

```mermaid
flowchart TB
    subgraph OLS_only["OLS only"]
        O1[Request] --> O2[OLS approved]
        O2 --> O3[User in group]
        O3 --> O4[Can open app - Sees ALL data]
    end

    subgraph RLS_only["RLS only"]
        R1[Request] --> R2[RLS approved]
        R2 --> R3[Share*.RLS has keys]
        R3 --> R4[Cannot open app - No OLS]
    end

    subgraph Both["Both OLS + RLS"]
        B1[Request] --> B2[OLS + RLS approved]
        B2 --> B3[User in group + RLS keys]
        B3 --> B4[Can open app - Sees ONLY allowed rows]
    end
```

| Case | Can open app/report? | What data they see |
|------|----------------------|---------------------|
| **OLS only** | Yes | All data (no RLS filter) |
| **RLS only** | No | Nothing (can't open it) |
| **Both** | Yes | Only rows allowed by RLS |

---

## 6. Managed OLS Only — End-to-End

Full flow from request to user in Entra group (managed apps only, OLSMode = 0).

```mermaid
flowchart TB
    A[1. Request - PermissionRequests<br/>RequestedFor, WorkspaceId] --> B[2. OLS Approval - PermissionHeaders<br/>PermissionType=0, ApprovalStatus=2]
    B --> C[3. OLSPermissions<br/>OLSItemType=1 Audience, OLSItemId→AppAudiences]
    C --> D[4. AppAudiences + WorkspaceApps<br/>OLSMode=0, AudienceEntraGroupUID]
    D --> E[5. Auto.OLSGroupMemberships<br/>RequestedFor, EntraGroupUID]
    E --> F[6. Sync - SakuraV2ADSync.ps1<br/>Add user to Entra group]
    F --> G[7. User can OPEN app/audience]
```

---

## 7. Managed OLS + RLS — End-to-End (Both Branches)

One request with both OLS and RLS; managed OLS path + RLS path to filtered data.

```mermaid
flowchart TB
    A[1. Request - PermissionRequests] --> B1[2a. OLS Approval<br/>PermissionType=0, Approved]
    A --> B2[2b. RLS Approval<br/>PermissionType=1, Approved]

    B1 --> C1[3a. OLSPermissions → AppAudiences<br/>OLSMode=0]
    B2 --> C2[3b. RLSPermissions<br/>SecurityModelId, SecurityTypeLoVId]

    C1 --> D1[4a. Auto.OLSGroupMemberships]
    C2 --> D2[4b. RLSPermission*Details<br/>EntityKey, ClientKey, PCKey...]

    D1 --> E1[5a. Sync → user in Entra group]
    D2 --> E2[5b. ShareAMER.RLS / ShareEMEA.RLS...]

    E1 --> F1[6a. User can OPEN app/report]
    E2 --> F2[6b. Power BI / App filters rows]

    F1 --> G[End: User has access + filtered data]
    F2 --> G
```

---

## 8. Managed vs Not Managed OLS (Split)

Where the OLS path splits: managed (sync) vs not managed (app owner).

```mermaid
flowchart TB
    A[OLSPermissions - Audience] --> B{WorkspaceApps.OLSMode}
    B -->|0 = Managed| C[Auto.OLSGroupMemberships]
    B -->|1 = NotManaged| D[ShareAMER.OLS / ShareEMEA.OLS...]

    C --> E[Sync adds user to Entra group]
    D --> F[App owner adds user manually]

    E --> G[User in group automatically after sync]
    F --> H[User in group only after owner adds]
```

---

## 9. RLS Flow — Per Domain

RLS is stored per domain in detail tables and exposed via Share schema views.

```mermaid
flowchart TB
    A[PermissionHeaders - RLS Approved] --> B[RLSPermissions<br/>PermissionHeaderId, SecurityModelId]
    B --> C{Domain}
    C --> D1[RLSPermissionAMERDetails]
    C --> D2[RLSPermissionEMEADetails]
    C --> D3[RLSPermissionCDIDetails]
    C --> D4[RLSPermissionGIDetails]
    C --> D5[RLSPermissionWFIDetails]
    C --> D6[RLSPermissionFUMDetails]

    D1 --> E1[ShareAMER.RLS]
    D2 --> E2[ShareEMEA.RLS]
    D3 --> E3[ShareCDI.RLS]
    D4 --> E4[ShareGI.RLS]
    D5 --> E5[ShareWFI.RLS]
    D6 --> E6[ShareFUM.RLS]

    E1 --> F[Power BI / App - Filter rows by user]
    E2 --> F
    E3 --> F
    E4 --> F
    E5 --> F
    E6 --> F
```

---

## 10. OLS Item Types (Audience vs Report)

OLS can point to an **audience** (app) or a **standalone report** (SAR). Only audiences with OLSMode=0 feed the managed sync.

```mermaid
flowchart TB
    A[OLSPermissions] --> B{OLSItemType}
    B -->|1 = Audience| C[AppAudiences<br/>AudienceEntraGroupUID]
    B -->|0 = Report SAR| D[WorkspaceReports<br/>ReportEntraGroupUID]

    C --> E{OLSMode?}
    E -->|0 Managed| F[Auto.OLSGroupMemberships → Sync]
    E -->|1 NotManaged| G[Share*.OLS → Owner adds]

    D --> H[Share*.OLS only - no sync<br/>Report owner adds user]
```

---

## 11. Sequence: Managed OLS + RLS (One User)

```mermaid
sequenceDiagram
    participant Req as Request
    participant PH as PermissionHeaders
    participant OLS as OLSPermissions
    participant Auto as Auto.OLSGroupMemberships
    participant Sync as Sync Script
    participant RLS_T as RLSPermissions
    participant Share as Share*.RLS
    participant User as User

    Req->>PH: OLS + RLS headers
    PH->>OLS: OLS item = Audience
    OLS->>Auto: RequestedFor, EntraGroupUID
    Auto->>Sync: Read view
    Sync->>User: Add to Entra group → can open app

    PH->>RLS_T: RLS permission
    RLS_T->>Share: Detail keys (Client, PC...)
    Share->>User: Power BI filters rows → sees only allowed data
```

---

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

---

## Proposal: One AD Group for App-Level OLS + RLS; Separate Groups per Audience

**Context:** When the downstream uses the **same** AD group for both OLS (app access) and RLS (data access), the sync adds the user once and they get both. When **different** groups are used for RLS, someone must manually add the user to the RLS group for data access.

**Proposed design:**

- **One AD group** used for **both** app-level OLS and RLS (so one sync/add gives the user app access and data scope; no manual "add to RLS group").
- **Separate AD groups per audience** for OLS only; these are already handled by the existing OLS AD sync process (e.g. `Auto.OLSGroupMemberships` → audience `AudienceEntraGroupUID`).

**Feasibility:**

- **Conceptually feasible.** Downstream (e.g. Power BI) can be configured so that:
  - The **shared** group grants both "can open the app" and the RLS role/data scope.
  - **Per-audience** groups grant which audience(s) the user belongs to; the current Sakura OLS sync already populates these from `Auto.OLSGroupMemberships`.
- **Gap today:** Sakura's sync only adds users to **audience-level** (and optionally report-level) groups from `Auto.OLSGroupMemberships`. It does not currently add users to a single "app + RLS" group. To support the shared group, you would need either:
  - A **sync source** (e.g. a view or feed) that lists "add this user to this app+RLS group" when they have both OLS (to that app) and RLS (for that workspace/domain), and extend the sync script to consume it, or
  - A separate process (e.g. pipeline or small script) that reads approved OLS+RLS state and adds users to the shared group.
- **Summary:** Using one AD group for app-level OLS and RLS, and different groups per audience handled by the existing OLS sync, is a feasible and sensible approach; it requires the downstream to use the shared group for both app access and data scope, and an extension to Sakura (or a side process) to populate that shared group, while per-audience groups continue to be handled by the current OLS AD sync process.

---

## Reference: Key Tables and Views

| Purpose | Table / View |
|--------|---------------|
| Request | `dbo.PermissionRequests` |
| OLS/RLS approval | `dbo.PermissionHeaders` (PermissionType 0=OLS, 1=RLS) |
| OLS item | `dbo.OLSPermissions` → AppAudiences or WorkspaceReports |
| RLS permission | `dbo.RLSPermissions` → `dbo.RLSPermission*Details` (per domain) |
| Managed OLS sync source | `Auto.OLSGroupMemberships` |
| Not-managed OLS (owner view) | `ShareAMER.OLS`, `ShareEMEA.OLS`, ... |
| RLS (per domain) | `ShareAMER.RLS`, `ShareEMEA.RLS`, ... |
