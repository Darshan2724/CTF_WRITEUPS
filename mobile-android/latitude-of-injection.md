# KPMG CTF 2026: Latitude of Injection — Android & SQLi Writeup

## Challenge Information

| Field | Details |
| :--- | :--- |
| **Challenge Name** | Latitude of Injection |
| **Category** | Android Reverse Engineering / SQL Injection |
| **APK File** | `loi_v1.apk` |
| **Package** | `com.ctf.loi` |
| **Target Web Server** | `https://kicyber-ffe53a857dc6-latitude-of-injectio-0-0.chals.io/` |

---

## 1. Challenge Overview

The challenge provides an Android APK named `loi_v1.apk` and points to a web portal requesting two pieces of information:
1. **City**
2. **Privileged Info**

The objective is to decompile the APK, analyze its location-checking logic to discover the required city coordinates, identify a local SQL injection vulnerability in the embedded SQLite helper to escalate privileges to administrator, and extract the privileged verification token required by the submission server.

---

## 2. Tools Used

- **JADX-GUI / apktool** — Android APK decompilation and resource decoding
- **SQLite3** — Database query testing
- **Browser / curl** — Web form submission

---

## 3. Step-by-Step Solution

### Step 1: APK Decompilation & Manifest Analysis
Decompiled `loi_v1.apk` using JADX:
- Package: `com.ctf.loi`
- Permissions requested in `AndroidManifest.xml`:
  - `ACCESS_COARSE_LOCATION`
  - `ACCESS_FINE_LOCATION`
  - `INTERNET`

Key application classes:
- `com.ctf.loi.MainActivity`
- `com.ctf.loi.ScreensKt`
- `com.ctf.loi.LocalDbHelper`
- `com.ctf.loi.UserRow`

---

### Step 2: Recovering the Target City from Coordinates
Examining `MainActivity` reveals how location verification is performed:

```java
final double allowedLat = Double.parseDouble(getString(R.string.l1));
final double allowedLon = Double.parseDouble(getString(R.string.l2));
final double allowedRadius = Double.parseDouble(getString(R.string.al_r));
```

Inspecting `res/values/strings.xml`:

```xml
<string name="l1">55.7558</string>
<string name="l2">37.6173</string>
<string name="al_r">300000.0</string>
```

Looking up coordinates `(55.7558, 37.6173)` points directly to **Moscow, Russia**. A bundled background asset in the APK depicts Red Square, confirming the city name:
```text
City: Moscow
```

---

### Step 3: Discovering the Local SQLite Injection
Inspecting `LocalDbHelper.java`, the app creates a local SQLite database `loi_local.db`:

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT,
    email TEXT,
    is_admin INTEGER DEFAULT 0
);
```

When a user registers an account, the query is constructed using direct string concatenation:

```java
String query = "INSERT INTO users (name, email, is_admin) VALUES ('"
    + name + "', '"
    + email + "', 0);";

database.execSQL(query);
```

The `name` parameter is completely unescaped and vulnerable to SQL injection.

---

### Step 4: Crafting the Administrator Insertion Payload
By supplying a custom `name` value, we can break out of the single quotes and force `is_admin = 1`:

```text
admin', 'attacker@local', 1);--
```

The resulting SQL statement executed by `execSQL()` becomes:
```sql
INSERT INTO users (name, email, is_admin)
VALUES ('admin', 'attacker@local', 1);--', 'email@example.com', 0);
```

Everything following `--` is treated as a comment. The database successfully inserts a record with:
- `name` = `admin`
- `email` = `attacker@local`
- `is_admin` = `1`

Logging in as `admin` satisfies the condition `is_admin == 1`, unlocking the privileged **Admin Panel**.

---

### Step 5: Extracting the Privileged Secret
The APK strings file contains ten Base64 strings named `not_required1` through `not_required10`. 

Tracing the Admin Panel logic in `ScreensKt.class` reveals that it specifically loads:
```java
getString(R.string.not_required7)
```

Inspecting `res/values/strings.xml`:
```xml
<string name="not_required7">ckdjN340OXQ3V1dKfmhNZjc3</string>
```

Decoding from Base64:
```bash
echo -n "ckdjN340OXQ3V1dKfmhNZjc3" | base64 -d
```

**Output:**
```text
rGc7~49t7WWJ~hMf77
```

---

### Step 6: Submitting to the Target Web Server
Submitting both recovered values to `https://kicyber-ffe53a857dc6-latitude-of-injectio-0-0.chals.io/`:
- **City:** `Moscow`
- **Privileged Info:** `rGc7~49t7WWJ~hMf77`

The server validates both parameters and returns the challenge completion flag.

---

## 4. Key Takeaways
1. **Never use string concatenation for SQL queries:** Even in local mobile databases like SQLite, parameterized queries (`database.insert()` or `SQLiteStatement`) must always be used.
2. **Obfuscation through Decoy Resources:** Challenge authors and malware authors often deploy decoy strings (`not_required1` ... `not_required10`) to confuse automated string dumpers. Tracing decompiled code references reveals the exact target string.
3. **OSINT in Mobile Resources:** Hardcoded coordinates in mobile applications can be cross-referenced against geospatial databases to pinpoint physical operational boundaries.
