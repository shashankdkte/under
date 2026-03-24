## Azure identity concepts — Mermaid diagram set

Below are multiple Mermaid diagrams explaining the relationships and flows between app registration, enterprise application, service principal, service account, and managed identity, based on `identity.txt`.

### 1) Big-picture mind map

```mermaid
mindmap
  root((Microsoft Entra ID))
    App registration
      Application object
      Client ID, secrets/certs
      Redirect URIs
      Permissions, roles, scopes
      Home tenant
    Enterprise application
      Tenant-local instance
      User/group assignment
      SSO & provisioning
      My Apps visibility
      Local consent controls
    Service principal
      App identity in a tenant
      Used for runtime access
      Gets Azure RBAC roles
      Appears under Enterprise Apps
    Service account
      Non-human identity (category)
      Could be Service Principal
      Could be Managed Identity
    Managed identity
      Azure-managed credentials
      System-assigned
      User-assigned
      Auth to Entra-backed resources
```

### 2) Core object relationships (class diagram style)

```mermaid
classDiagram
  class AppRegistration {
    +applicationObjectId
    +clientId
    +redirectUris
    +permissionsAndScopes
    +appRoles
    +homeTenant
  }
  class ServicePrincipal {
    +objectId
    +tenantId
    +appId(clientId)
    +roleAssignments
    +usedForRuntime
  }
  class EnterpriseApplication {
    +managesServicePrincipal
    +assignUsersGroups
    +configureSSO
    +provisioningSettings
  }
  class ServiceAccount {
    <<category>>
  }
  class ManagedIdentity {
    +azureManagedCredentials
    +systemAssignedOrUserAssigned
  }

  AppRegistration "1" -- "many" ServicePrincipal : "instantiated into tenants"
  ServicePrincipal <.. EnterpriseApplication : "admin/portal view"
  ServiceAccount <|.. ServicePrincipal
  ServiceAccount <|.. ManagedIdentity
```

### 3) Custom application flow (from registration to runtime)

```mermaid
flowchart LR
  A[Create App Registration (home tenant)] --> B[Service Principal auto-created]
  B --> C[Enterprise Applications portal manages SP]
  C --> D[Assign users/groups, configure SSO/consent]
  B --> E[Grant API permissions or Azure RBAC roles]
  E --> F[Runtime: App uses SP identity for access]
```

### 4) Third-party SaaS/gallery app adoption

```mermaid
flowchart LR
  S[Gallery / Third-party App (external blueprint)] --> T[Service Principal in your tenant]
  T --> U[Enterprise Applications admin experience]
  U --> V[Assign users/groups, SSO, provisioning, visibility]
  T --> W[Local access decisions (API/consent/RBAC)]
```

### 5) Managed identity model (Azure-managed credentials)

```mermaid
flowchart TB
  R[Azure Resource (App Service/VM/Function/etc.)] --> MI[Enable Managed Identity]
  MI -->|System-assigned| SA[(Identity tied to resource lifecycle)]
  MI -->|User-assigned| UA[(Reusable identity attached to resources)]
  SA --> AUTH[Authenticate to Entra-backed resources without handling secrets]
  UA --> AUTH
```

### 6) “When to use which?” decision helper

```mermaid
flowchart TB
  Q{Are you defining your own app's identity and auth settings?}
  Q -->|Yes| AR[Use App Registration]
  Q -->|No| Q2{Enable local management of existing app in tenant?}
  Q2 -->|Yes| EA[Use Enterprise Applications (tenant-local mgmt)]
  Q2 -->|No| Q3{Is this a workload identity (not human)?}
  Q3 -->|Yes| Q4{Runs on Azure resource with Entra-backed targets?}
  Q4 -->|Yes| MI[Use Managed Identity]
  Q4 -->|No| SP[Use Service Principal]
  Q3 -->|No| HU[Human user identity (out of scope)]
```

### 7) Layered mental model (stack)

```mermaid
flowchart TB
  L1[Layer 1: App registration - definition]
  L2[Layer 2: Service principal - tenant identity]
  L3[Layer 3: Enterprise application - admin view]
  L4[Layer 4: Service account - non-human category]
  L5[Layer 5: Managed identity - Azure-managed]

  L1 --> L2 --> L3
  L4 -. category includes .- L2
  L4 -. category includes .- L5
```

### 8) Common misunderstandings (contrast diagram)

```mermaid
flowchart LR
  classDef note fill:#fff5b1,stroke:#d4b106,color:#333;
  A1[App registration (blueprint)] -. related but not same .- A2[Enterprise application (tenant admin view)]
  N1[Registration = definition]:::note
  N2[Enterprise app manages SP in-tenant]:::note
  A1 -.-> N1
  A2 -.-> N2

  B1[Service principal] --- A2
  N3[Enterprise application is the admin experience around the SP]:::note
  B1 -.-> N3

  C1[Service account] ---|broad term| C2[Service principal]
  C1 ---|broad term| C3[Managed identity]
```

### 9) Permissions and access contexts

```mermaid
flowchart LR
  subgraph Entra
    AR1[App Registration]
    SP1[Service Principal (tenant identity)]
    EA1[Enterprise Apps (admin view)]
    AR1 --> SP1
    SP1 <--> EA1
  end

  subgraph Azure
    RES[Azure Resource]
    ROLE[RBAC Role Assignment]
  end

  SP1 -- assigned --> ROLE
  ROLE -- grants access to --> RES
```

### 10) Delegated vs application permissions (high-level)

```mermaid
flowchart TB
  D[Delegated permissions (user present)] -->|consent| API1[Target API]
  D -->|access token includes user| OUT1[Access as user]

  A[Application permissions (no user)] -->|admin consent| API2[Target API]
  A -->|access token as app| OUT2[Daemon/automation access]
```

### 11) Service principal creation pathways

```mermaid
flowchart LR
  R1[Register new app] --> SP1[Service principal auto-created in same tenant]
  R2[Consent to multi-tenant app from other tenant/gallery] --> SP2[Service principal created in your tenant]
  SP1 --> EA1[Manage via Enterprise Apps]
  SP2 --> EA1
```

### 12) Managed identity types side-by-side

```mermaid
classDiagram
  class SystemAssignedMI {
    +Created with resource
    +Deleted with resource
    +One-to-one binding
  }
  class UserAssignedMI {
    +Created independently
    +Reusable across resources
    +Independent lifecycle
  }
  SystemAssignedMI <|-- ManagedIdentityTypes
  UserAssignedMI <|-- ManagedIdentityTypes
  class ManagedIdentityTypes {
    <<abstract>>
  }
```

### 13) Admin touchpoints vs runtime

```mermaid
sequenceDiagram
  participant Admin
  participant EnterpriseApps as Enterprise Applications
  participant Entra as Entra Directory
  participant App as Application/Workload
  participant Resource as Azure Resource/API

  Admin->>EnterpriseApps: Assign users/groups, configure SSO
  EnterpriseApps->>Entra: Update SP settings/consent
  App->>Entra: Obtain token (as app or delegated)
  App->>Resource: Call with token
  Resource-->>App: Authorized response (per RBAC/consent)
```

### 14) Full lifecycle overview

```mermaid
flowchart TB
  subgraph Design
    DR[Define App Registration]
  end
  subgraph Tenant Instance
    SP[Service Principal]
    EA[Enterprise Applications Admin]
  end
  subgraph Access
    PERM[API permissions / Admin consent]
    RBAC[Azure RBAC assignments]
  end
  subgraph Runtime
    TOK[Token acquisition]
    USE[Resource/API access]
  end

  DR --> SP
  SP <--> EA
  SP --> PERM
  SP --> RBAC
  PERM --> TOK
  RBAC --> USE
  TOK --> USE
```

### 15) “Which identity is it?” quick categorization

```mermaid
flowchart LR
  SCAT[Service account (category)] --> SPN[Service principal (specific Entra object)]
  SCAT --> MI1[Managed identity (Azure-managed)]
  classDef note fill:#fff5b1,stroke:#d4b106,color:#333;
  NOTE1[Category label only; not a single object type.]:::note
  SCAT -.-> NOTE1
```

