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
