# Patches — kaise lagane hain (laptop on hote hi, ~5 min)

Is folder me 2 patches hain. Dono alag repos ke liye hain:

| Patch | Repo | Kya fix karta hai |
|---|---|---|
| `mcq-bulk-publish.patch` | `ujjwal-pathak-project` | MCQ publish stuck + bulk approve buttons (chapter / subject / sab 4700) |
| `apk-workflow-fix.patch` | `student-dashboard-frontend` | Signed APK workflow ka broken signing wiring |

## Patch 1 — MCQ bulk publish (mentor dashboard)

PowerShell me (jahan bhi repo clone ho, ya pehle clone karo):

```powershell
cd path\to\ujjwal-pathak-project
curl -L -o mcq-bulk-publish.patch https://github.com/anujvkarewad-cyber/UPM-Android-Releases/raw/arena/01a013a8-upm-android-releases/mcq-fix/mcq-bulk-publish.patch
git apply --check mcq-bulk-publish.patch   # "koi error nahi" = theek
git apply mcq-bulk-publish.patch
git add -A
git commit -m "Add one-click bulk approve & publish"
git push origin main
```

Push karte hi **Render backend auto-redeploy** ho jayega (blueprint se connected hai).
Deploy complete hone ke baad mentor dashboard kholo:

1. **Dashboard → MCQ Review card** → naya bada button: **"Approve ALL & publish"** — ek click me pura bank (4,700) publish. Confirm popup aayega.
2. **AI Content → Chapter Coverage** → har chapter row me **"Approve & publish"** button + upar toolbar me subject select karke **"Approve subject & publish"** + **"Approve ALL & publish (whole bank)"**.
3. Per-question Approve button pehle jaisa hi kaam karta hai.

Kya hota hai andar se:
- Ek hi release revision me saare eligible questions + scenarios + chapters → `published`
- Jo questions me **validation errors** hain wo publish NAHI hote — review me rehte hain, count response me dikhta hai (kharab question students tak nahi jayega)
- Students ko turant `bank.json` se naya content mil jata hai

Extra fix is patch me: `/api/admin/fast-publish-all` aur `/fast-status` pehle **bina login** public the (koi bhi internet pe sab kuch publish kar sakta tha) — ab mentor login zaroori hai.

Test status: backend 58/58 pass (6 naye tests isi feature ke), frontend production build clean.

## Patch 2 — APK workflow signing fix

```powershell
cd path\to\student-dashboard-frontend
curl -L -o apk-workflow-fix.patch https://github.com/anujvkarewad-cyber/UPM-Android-Releases/raw/arena/01a013a8-upm-android-releases/mcq-fix/apk-workflow-fix.patch
git apply --check apk-workflow-fix.patch
git apply apk-workflow-fix.patch
git add -A
git commit -m "Fix release APK signing"
git push origin main
```

⚠️ **Order matter karta hai:** ye patch pehle push karo, PHIR keystore wale steps (PLAYBOOK.md) — kyunki app.json me versionCode 17 bump karne wala push build trigger karega, aur us waqt tak fixed workflow + secrets dono ready hone chahiye.

## Agar `git apply` fail ho jaye

Iska matlab repo me naye commits aa chuke hain jo patch se takraate hain. Tab batao, main patch fresh main ke upar rebase kar dunga. (`git apply -3` bhi try kar sakte ho agar merge conflicts simple hon.)
