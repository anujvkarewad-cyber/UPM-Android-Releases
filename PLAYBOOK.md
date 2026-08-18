# UPM Android v1.10.2 — Live Update Playbook (corrected)

> **Date:** 18 Aug 2026 · **Status jo maine verify kiya:** last successful build (13 ghante pehle) **signed NAHI tha**, aur current workflow mein signing ka **wiring hi toota hua hai**. Neeche 3 fixes + exact steps hain.

---

## 🔴 Maine kya problema (evidence ke saath)

### Problem 1 — Secrets set hi nahi hain
Latest successful run (`Production Signed APK - Live Channel`, run #32051841696) pe annotation:

```
! No keystore secret - building unsigned preview
```

Matlab `ANDROID_KEYSTORE_BASE64` GitHub pe exist hi nahi karta. Jo v1.10.2-16 APK abhi live hai wo **properly signed nahi hai**.

### Problem 2 — Secrets set karne ke baad bhi sign NAHI hota (ye sabse bada bug hai)
Tumhara workflow `MYAPP_RELEASE_*` values `mobile/android/gradle.properties` me likhta hai. Lekin:

- `mobile/android/` repo me hai hi nahi — CI me `npx expo prebuild --clean` ise har baar **fresh generate** karta hai
- Maine Expo SDK 57 ka actual prebuild template (`expo-template-bare-minimum@57.0.16`) download karke check kiya — uska generated `build.gradle` release buildType me **hardcoded** hai:

```gradle
release {
    // Caution! In production, you need to generate your own keystore file.
    signingConfig signingConfigs.debug   // ← MYAPP_RELEASE_* kahin use hi nahi hota!
}
```

**Matlab: tum Step 1–3 pura sahi karke bhi, green tick ke baad bhi APK debug-signed hi aata.** Silent failure — sab kuch "successful" dikhta but signing kabhi lagti hi nahi.

### Problem 3 — Store path bhi galat tha
Purane workflow me `MYAPP_RELEASE_STORE_FILE=../android-release-key.jks` tha, jo `mobile/android/app/` se resolve hoke `mobile/android/android-release-key.jks` pe jaata — jabke keystore `mobile/android-release-key.jks` pe decode hota tha. Ye bhi fix kar diya hai.

---

## ⚠️ Ek honest baat — "bina uninstall ke update" ke baare me

Maine ye bhi verify kiya ki purane installs (v1.10.0 — 17 downloads, aur v1.10.2-16 — 2 downloads) **kis signature** pe hain:

- Expo template ke andar ek **public** `debug.keystore` committed hai (store password `android`, alias `androiddebugkey`) — yehi sab CI builds me use hua hai
- Matlab abhi ke ~19 users ke phones pe **public Expo debug key** wala APK hai

**Iska matlab:**
1. Naya personal keystore banaoge → signature alag → **existing users ko EK baar uninstall + fresh install karna padega.** Data server pe hai (login-based app), to students bas dobara login kar lenge. Ye one-time hi hai — uske baad hamesha smooth updates.
2. Yehi sahi move hai — public debug key pe sign karna security hole hai (Expo template wali key duniya ke paas hai; koi bhi malicious APK bana ke tumhare students ke phones pe "update" install karwa sakta hai).

**Students ko message suggestion:** *"App update: ek baar purana app uninstall karke naya install karna hoga (security upgrade). Apna login use karke sign-in kar lena — saara progress account pe safe hai."*

> ⚡ Ek aur trap jo avoid karni hai: v1.10.2-16 (code 16) pehle 2 logo ne install kar liya hai. Naya build **versionCode 17** ke saath banao (neeche Step 4) — warna un 2 logo ko update prompt hi nahi milega (app `versionCode > current` pe hi prompt karta hai — `AppUpdateContext.tsx` line 53).

---

## ✅ Ab kya karna hai — exact steps

### Step 0 — Fixed workflow lagaao (5 min)
Maine fixed workflow bana diya hai: **`fix/production-apk.yml`** (is repo me).

`student-dashboard-frontend` → `.github/workflows/production-apk.yml` ka **poora content replace** kar do is file se. (Mere paas us repo ka write access nahi hai, isliye main khud push nahi kar sakta.)

Kya fix hua:
1. **build.gradle patch step add** — prebuild ke baad generated `build.gradle` me release signingConfig inject hota hai (maine real SDK-57 template pe test kiya hai)
2. **Fail-fast** — secrets missing honge to build RED ho jayega, silent debug build nahi banega
3. **Signature verification step** — build ke baad `apksigner` se confirm hota hai ki APK tumhare keystore se hi signed hai; warna fail
4. **Auto-publish to this repo** — APK GitHub Release asset ban ke UPM-Android-Releases pe auto-upload hoga (agar `RELEASES_PAT` set hai), git history me 50MB APK commit hona band

### Step 1 — Keystore (2 min) — ye tumhe karna hai
- **Best:** purana `upm-release-key.jks` mil gaya to wahi use karo
- Warna Android Studio → Build → Generate Signed Bundle/APK → Create new → `upm-release-key.jks` Desktop pe banao
- ⚠️ **Keystore + passwords ka backup lo** (Google Drive/Email). Ye khoya to future updates kabhi nahi jayenge — hamesha uninstall karke hi install karna padega. **Isko git me kabhi commit mat karna.**

### Step 2 — GitHub Secrets (3 min)
PowerShell (path apne hisaab se):

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("C:\Users\AapkaNaam\Desktop\upm-release-key.jks")) | Set-Content b64.txt
```

`student-dashboard-frontend` → Settings → Secrets and variables → Actions → New repository secret (**5 baar**):

| Secret | Value |
|---|---|
| `ANDROID_KEYSTORE_BASE64` | b64.txt ka pura content |
| `STORE_PASSWORD` | jo password diya (e.g. `Upm@2024`) |
| `KEY_ALIAS` | `upm-release` (jo alias diya) |
| `KEY_PASSWORD` | same password |
| `RELEASES_PAT` *(optional)* | fine-grained PAT: repo = `UPM-Android-Releases`, permission = Contents: Read & write — isse APK auto-upload hoga yahan (nahi to manually artifact download karke upload karna padega) |

### Step 3 — versionCode bump (1 min) — **mat bhoolna**
`student-dashboard-frontend` → `mobile/app.json`:

```json
"version": "1.10.2",
"android": { "versionCode": 17 }   // 16 se 17 kiya
```

Commit + push to `main` — `mobile/**` path se workflow **automatically** trigger hoga.

### Step 4 — Build & verify
- Ya to push se auto-run hoga, ya Actions → Production Signed APK - Live Channel → Run workflow
- Green tick se pehle **"Verify APK is signed with OUR keystore"** step me ye dikhna chahiye:

```
Keystore cert SHA-256: a1b2c3...
APK signer  SHA-256: a1b2c3...   ← dono SAME hone chahiye
SIGNATURE VERIFIED - APK signed with upm-release-key.jks
```

Agar ye step red hai to mat ship karna.

### Step 5 — Live channel update
- `RELEASES_PAT` set hai to APK yahan (UPM-Android-Releases) naye tag `v1.10.2-17` pe **auto-upload** ho jayega + `latest.json` update hoga
- Nahi to Artifacts se `UPM-v1.10.2-17-release.apk` download karo → yahan naya Release `v1.10.2-17` banao → upload karo

### Step 6 — Vercel env (2 min) + redeploy
Vercel → student-dashboard-frontend → Settings → Environment Variables:

```
APP_ANDROID_APK_URL  = https://github.com/anujvkarewad-cyber/UPM-Android-Releases/releases/download/v1.10.2-17/UPM-v1.10.2-17-release.apk
APP_ANDROID_VERSION  = 1.10.2
APP_ANDROID_VERSION_CODE = 17
APP_ANDROID_MIN_VERSION_CODE = 17   ← ise 17 karne se sabko mandatory update milega
APP_ANDROID_RELEASE_NOTES = Security upgrade - ek baar uninstall karke install karein
```

Redeploy kar do (agar auto nahi hua).

### Step 7 — Test karo (5 min)
1. Purana install wale phone pe: update prompt aayega (17 > 15/16) → download → install pe "App not installed" aayega (expected — signature change)
2. Purana uninstall → naya install → login → kaam kare
3. **Next update se yeh problem kabhi nahi aayegi** — usi keystore se build hoga to seedha in-place update milega

---

## Future releases (ab se simple)

1. `mobile/app.json` me version/versionCode bump + changes
2. Push to main → auto build → auto verify → auto publish (RELEASES_PAT ke saath)
3. Vercel me nayi APK URL + versionCode env update

Bas — teen steps, koi Android Studio nahi, koi manual signing nahi.
