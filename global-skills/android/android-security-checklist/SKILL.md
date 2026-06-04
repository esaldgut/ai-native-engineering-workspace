---
name: android-security-checklist
description: >-
  Systematic Android security checklist aligned with OWASP MASVS and platform guidance — secrets via
  DataStore + Google Tink (AeadSerializer) + Android Keystore (NOT the deprecated
  EncryptedSharedPreferences), TLS cert pinning via Network Security Config (SPKI hash) and/or OkHttp
  CertificatePinner with a backup pin, OAuth via Chrome Custom Tabs + PKCE + AppAuth-Android (WebView is
  forbidden), and manifest hardening (android:exported, allowBackup=false, usesCleartextTraffic=false).
  Use when handling tokens, network security, sign-in, or attack-surface review on Android.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Android — Network Security Configuration (base-config, domain-config, pin-set SPKI, networkSecurityConfig; applies to all libs at OS level)"
      url: "https://developer.android.com/privacy-and-security/security-config"
      version: "Android (current)"
    - source: "Android — DataStore + Tink (androidx.datastore:datastore-tink AeadSerializer; SharedPreferencesMigration)"
      url: "https://developer.android.com/jetpack/androidx/releases/datastore"
      version: "androidx.datastore (current)"
    - source: "Android — Keystore system"
      url: "https://developer.android.com/training/articles/keystore"
      version: "Android (current)"
    - source: "Google — OAuth 2.0 WebView prohibition (Custom Tabs required)"
      url: "https://developers.googleblog.com/upcoming-security-changes-to-googles-oauth-20-authorization-endpoint-in-embedded-webviews/"
      version: "current policy"
    - source: "OpenID AppAuth-Android (Custom Tabs + PKCE, RFC 8252)"
      url: "https://github.com/openid/AppAuth-Android"
      version: "RFC 8252"
    - source: "OWASP MASVS"
      url: "https://mas.owasp.org/MASVS/"
      version: "MASVS v2"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "Google I/O (May) + Compose BOM major"
    or_date: "2026-12-01"
  decay_risk: high
  status: current
---

# Android security checklist (MASVS-aligned)

A boundary-by-boundary checklist for Android apps: secrets storage, transport security,
authentication, attack surface, and input validation — each anchored to `developer.android.com` or
[OWASP MASVS](https://mas.owasp.org/MASVS/). The headline change since pre-2024 advice:
**`EncryptedSharedPreferences` is deprecated** — any checklist still recommending it is stale.

## When to invoke

- You're storing a token/secret, configuring TLS, building a sign-in flow, or reviewing
  `AndroidManifest.xml` attack surface.
- You see `EncryptedSharedPreferences`, a `WebView` used for OAuth, a single TLS pin with no backup, or
  `android:exported="true"` without a `permission` — fix each per the rules below.

**Announce on invoke:** "Using `android-security-checklist` to review secrets / transport / auth /
manifest against OWASP MASVS and the current Android security guidance."

## 1. Secrets storage — `EncryptedSharedPreferences` is OUT

`androidx.security:security-crypto` (Jetpack Security) reached its **terminal release (1.1.0)** and
`EncryptedSharedPreferences` is **deprecated** — driven by Keystore inconsistency across OEMs (keyset
corruption crashes), main-thread IO StrictMode violations, and no clean encryption-scheme upgrade path.

**Canonical replacement: DataStore + Google Tink + Android Keystore.**

- **DataStore** (`androidx.datastore:datastore`) is the persistence layer — but it is **not encrypted by
  default**. "Migrated to DataStore, done" is a security bug for sensitive values.
- **[Tink](https://developers.google.com/tink)** does the encryption. `androidx.datastore:datastore-tink`
  ships an `AeadSerializer` (Authenticated Encryption with Associated Data) that encrypts/decrypts the
  DataStore payload for you.
- **Android Keystore** protects the master key (`android-keystore://…`), so the key never leaves hardware-
  backed storage.
- Migrate existing data with the `SharedPreferencesMigration` constructor that injects the old
  `SharedPreferences` (including a former `EncryptedSharedPreferences`) into the new DataStore.

```kotlin
// Tink AEAD master key in the Android Keystore; encrypt before persisting via DataStore
val aead: Aead = AndroidKeysetManager.Builder()
    .withSharedPref(context, "master_keyset", "master_key_prefs")
    .withKeyTemplate(AeadKeyTemplates.AES256_GCM)
    .withMasterKeyUri("android-keystore://my_master_key")
    .build()
    .keysetHandle
    .getPrimitive(Aead::class.java)

val ciphertext = aead.encrypt(tokenBytes, /* associatedData */ null)
// persist `ciphertext` through DataStore (e.g. datastore-tink AeadSerializer)
```

For preferences (theme, last tab) DataStore alone is fine — no encryption needed. Encryption is
**mandatory only for sensitive values** (tokens, PII).

## 2. Transport security — Network Security Config + (optional) OkHttp pinning

Two complementary paths (not mutually exclusive):

- **Network Security Config** (`res/xml/network_security_config.xml`) — declarative, enforced at the
  **OS `NetworkSecurityPolicy` level**, so it covers **all** network libraries (OkHttp,
  `HttpsURLConnection`, anything on platform sockets). Use it for the global no-cleartext baseline and
  per-domain `pin-set`. Reference it from the manifest with `android:networkSecurityConfig`.
- **OkHttp `CertificatePinner`** — programmatic, scoped to that one OkHttp client. Same SPKI hash format.
  Many apps use both: NSC for global policy, `CertificatePinner` for the hashes on the API client.

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors><certificates src="system"/></trust-anchors>
    </base-config>
    <domain-config>
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set expiration="2027-01-01">
            <pin digest="SHA-256">PRIMARY_SPKI_HASH_BASE64=</pin>
            <pin digest="SHA-256">BACKUP_SPKI_HASH_BASE64=</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

- **The pin is the SHA-256 of the SubjectPublicKeyInfo (SPKI), NOT the whole certificate.** Generate it:
  `openssl x509 -pubkey -noout -in cert.pem | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64`.
- **Always include ≥1 backup pin.** A single pin makes cert rotation a total outage; the backup (next
  intermediate or next-rotation key) prevents it (MASVS-NETWORK-4).

## 3. Authentication — Custom Tabs + PKCE, never WebView

- **WebView for OAuth is a Google policy violation, not just a best practice.** Google explicitly rejects
  embedded WebViews at the authorization endpoint (a host app can keylog the login form). Any compliant
  IdP refuses the auth. **Chrome Custom Tabs** (`androidx.browser:browser`) is the only compliant browser
  surface.
- **PKCE is mandatory for public clients** (`code_verifier` + `code_challenge` = SHA-256 of the verifier)
  per **RFC 8252** (OAuth 2.0 for Native Apps).
- **[AppAuth-Android](https://github.com/openid/AppAuth-Android)** is the canonical implementation — it
  uses Custom Tabs and PKCE and deliberately does **not** support WebView.
- **MASVS-AUTH-2:** never rely on client-side authorization. The server re-authorizes every request; UI
  gating is UX, not a security boundary.

## 4. Manifest hardening (attack-surface reduction)

```xml
<application
    android:allowBackup="false"
    android:networkSecurityConfig="@xml/network_security_config"
    android:usesCleartextTraffic="false">
    <activity android:name=".MainActivity" android:exported="true">
        <intent-filter>
            <action android:name="android.intent.action.MAIN"/>
            <category android:name="android.intent.category.LAUNCHER"/>
        </intent-filter>
    </activity>
</application>
```

- **`android:exported`** is required from API 31. Default to `false`. Set `true` only for components with
  intent-filters that legitimately receive external intents; protect sensitive ones with
  `android:permission`. `exported="true"` without a `permission` (and without being a launcher/validated
  deep-link target) is a high-severity finding on every static analyzer.
- **`android:allowBackup="false"`** unless you have a deliberate backup story — `true` lets `adb backup`
  copy app data off the device.
- **`android:usesCleartextTraffic="false"`** (or rely on NSC) — HTTPS-only.
- **`android:debuggable="false"`** in release (`release { isDebuggable = false }`; Android Studio sets
  this per build type).

## 5. Input validation (MASVS-CODE-4)

Validate at the **boundary** (Repository / API client), not in the UI. Use a sealed `Result<T>` for
fallible operations; never let an unchecked `IllegalArgumentException` from a malformed server response
become a crash.

## Related skills

- `global-skills/android/compose-feature-scaffold/SKILL.md` — wire the validated repository + Ktor/OkHttp
  client this checklist hardens.
- `global-skills/android/android-testing-patterns/SKILL.md` — negative/boundary tests (injection,
  malformed tokens, token-leakage-in-logs) that prove these controls.
- `global-skills/apple/apple-security-patterns/SKILL.md` — the iOS twin (Keychain, ATS, pinning) for a
  cross-platform app.

## Sources

- [Network Security Configuration](https://developer.android.com/privacy-and-security/security-config) ·
  [Keystore system](https://developer.android.com/training/articles/keystore) ·
  [DataStore release notes (datastore-tink)](https://developer.android.com/jetpack/androidx/releases/datastore) ·
  [Google Tink](https://developers.google.com/tink)
- [Google OAuth WebView policy](https://developers.googleblog.com/upcoming-security-changes-to-googles-oauth-20-authorization-endpoint-in-embedded-webviews/) ·
  [AppAuth-Android](https://github.com/openid/AppAuth-Android) · [OWASP MASVS](https://mas.owasp.org/MASVS/)

---

**Last verified:** 2026-06-03 against developer.android.com (NSC + Keystore + DataStore-tink live),
Google OAuth WebView policy, AppAuth-Android (RFC 8252), OWASP MASVS. `EncryptedSharedPreferences`
confirmed deprecated (`androidx.security:security-crypto` 1.1.0 terminal).
**Re-check after:** Google I/O 2026 + next Compose BOM major, or by 2026-12-01. **Decay risk:** high
(secrets-storage guidance and pinning APIs shift).
**Found a drift?** Run `/skill-pattern-freshness-audit android`.
