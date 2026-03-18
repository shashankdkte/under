# One AD Group for OLS App-Level + RLS — Proposal (Mermaid)

This document explains the proposal using Mermaid diagrams: **one** AD group for both OLS (app-level) and RLS, plus **separate** groups per audience handled by the existing OLS AD sync.

---

## 1. The problem

When the **same** AD group is used for OLS and RLS, sync adds the user once → they get both. When **different** groups are used for RLS, that RLS group is never filled by Sakura → someone must add the user manually.

```mermaid
flowchart LR
    subgraph Same["Same group for OLS + RLS"]
        S1[User approved OLS + RLS]
        S2[Sync adds to ONE group]
        S3[User gets app + data access]
        S1 --> S2 --> S3
    end

    subgraph Different["Different groups"]
        D1[User approved OLS + RLS]
        D2[Sync adds to OLS group only]
        D3[RLS group empty]
        D4[Manual add needed]
        D1 --> D2 --> D3 --> D4
    end
```

---

## 2. The proposal in one picture

- **One AD group** = OLS (app-level) **and** RLS → sync fills it when user has both; no manual “add to RLS group.”
- **Different groups per audience** = OLS only → existing OLS AD sync fills them.

```mermaid
flowchart TB
    subgraph Proposal["Proposal"]
        P1[One shared group per app<br/>OLS app-level + RLS]
        P2[Sync adds user when they have<br/>both OLS and RLS]
        P3[Separate groups per audience<br/>OLS only]
        P4[Existing OLS sync fills<br/>audience groups]
        P1 --> P2
        P3 --> P4
    end
```

---

## 3. Group layout

```mermaid
flowchart LR
    subgraph Entra["Entra (Azure AD) groups"]
        G0["Shared group (one per app)<br/>→ App-level OLS + RLS gate"]
        G1["Audience A group<br/>→ Can see Audience A"]
        G2["Audience B group<br/>→ Can see Audience B"]
    end

    subgraph Sakura["Sakura sync"]
        S0["New: add user to shared group<br/>when OLS + RLS for this app"]
        S1["Existing: add user to<br/>audience groups"]
    end

    S0 --> G0
    S1 --> G1
    S1 --> G2
```

---

## 4. End-to-end flow (user gets both OLS and RLS)

```mermaid
sequenceDiagram
    participant User
    participant Sakura
    participant DB as Sakura DB
    participant Sync as AD Sync
    participant Entra as Entra groups
    participant PBI as Power BI

    User->>Sakura: Request OLS + RLS
    Sakura->>DB: OLS + RLS approved
    Note over DB: OLSPermissions, RLSPermissions<br/>Share*.RLS updated

    Sync->>DB: Read Auto.OLSGroupMemberships<br/>+ new AppRLS view
    Sync->>Entra: Add user to shared group
    Sync->>Entra: Add user to audience group(s)

    User->>PBI: Open app
    PBI->>Entra: Check shared group → can open
    PBI->>DB: Read Share*.RLS → which rows
    PBI->>User: Show app + filtered data
```

---

## 5. What sync fills today vs proposed

```mermaid
flowchart TB
    subgraph Today["Today"]
        T1[Auto.OLSGroupMemberships]
        T2[Audience groups only]
        T3[No app-level group]
        T4[No RLS group]
        T1 --> T2 --> T3
        T2 --> T4
    end

    subgraph Proposed["Proposed"]
        R1[Auto.OLSGroupMemberships]
        R2[New: AppRLS view]
        R3[Audience groups]
        R4[Shared app+RLS group]
        R1 --> R3
        R2 --> R4
    end
```

---

## 6. RLS: still from Share*.RLS (no group per dimension)

The shared group is only a **gate** (“has RLS for this app”). Which rows the user sees still comes from **Share*.RLS** — no extra AD group per dimension.

```mermaid
flowchart LR
    subgraph Gate["Shared group (gate)"]
        A[User in shared group]
        B[Can open app]
        C[Eligible for RLS]
        A --> B
        A --> C
    end

    subgraph Rows["Which rows"]
        D[Share*.RLS views]
        E[User + dimension keys]
        F[Power BI filters rows]
        D --> E --> F
    end

    C --> F
```

---

## 7. Summary

| Item | Description |
|------|-------------|
| **Shared group** | One per app: app-level OLS + “has RLS.” Sync fills when user has both OLS and RLS. |
| **Audience groups** | One per audience: OLS only. Existing sync fills from `Auto.OLSGroupMemberships`. |
| **RLS row filter** | Still **Share*.RLS** (user + dimension keys). No AD group per dimension. |
| **Upside** | No manual add to a separate “RLS group”; one sync covers shared + audience groups. |

---

*See also: `OLS-RLS-Use-Cases-And-Diagrams.md`, `OLS_RLS_ORIGINAL_VS_PROPOSED_QA.md`.*
