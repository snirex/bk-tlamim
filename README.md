https://claude.ai/chat/5fa444ca-69c5-448a-b241-36f928f99fd9
# bk-tlamim
Betkneset Tlamim Account management
<div align="center">

# 🕍 SynApp

**Congregant accounts & pledges management for synagogue gabbaim — Hebrew, RTL, serverless.**

[English](#-english) · [עברית](#-עברית)

![Status](https://img.shields.io/badge/status-in%20active%20development-brightgreen)
![Stack](https://img.shields.io/badge/stack-Alpine.js%20%7C%20Firebase-orange)
![UI](https://img.shields.io/badge/UI-Hebrew%20RTL-blue)
![Hosting](https://img.shields.io/badge/hosting-GitHub%20Pages-lightgrey)

**Live:** <https://snirex.github.io/bk-tlamim> · **Try it without an account:** <https://snirex.github.io/bk-tlamim/?demo=1>

</div>

---

# 🇬🇧 English

## Overview

**SynApp** (repo: `bk-tlamim`) is a lightweight web application that lets a synagogue's *gabbai* manage the financial accounts of the congregation: members, pledges (commitments), actual payments, funds (*kupot*), reports and backups.

Worshippers can also look up **their own** account: they enter their phone number, confirm it with an SMS one-time code, and see only their own movements. All filtering happens on the server (Cloud Functions), so no member ever receives anyone else's data.

There is no build step and no backend to host: the whole front-end is a single `index.html` served by GitHub Pages, backed by Firebase.

## Features

**For the gabbai (admin)**
- Members management (name, unique nickname/ID, phone, email; configurable member fields)
- Commitments and payments with a movements ledger per member, balance sign conventions (debt = red/minus, credit = green/plus), and negative amounts for opening balances
- **Funds (קופות):** per-fund history and balance, payments/commitments linked to funds, and global *payment purposes*
- Reports: movements, summary mode (one net balance row per member), member search/filter, column visibility, clickable rows
- Export: **Excel**, **PDF** and **PNG screenshots** of tables
- **Backup / restore** to JSON and to **Google Drive** (OAuth, `drive.file` scope), with backup history
- Admin login with email + password, optional Google sign-in, password reset
- "Preview as worshipper" mode to see exactly what a member sees
- Configurable columns and theme colors, saved per synagogue in Firestore
- Debounced auto-save to Firestore with a flush-on-close guard
- Error logging to Firestore for worshipper login problems (`error_logs`)

**For worshippers**
- Phone number + SMS OTP login (Firebase Phone Auth, reCAPTCHA protected)
- Family accounts: one phone can switch between several linked members
- Read-only movements table, built to match the admin's movements report
- `?phone=` deep link to pre-fill the login form

**Other**
- Full RTL / Hebrew UI, `DD/MM/YYYY` dates, LTR currency values
- **Demo mode** (`?demo=1`): runs fully locally with seed data, no Firebase connection
- Privacy policy page (`privacy.html`) linked from the login page and sidebar

## Tech stack

| Layer | Technology |
|---|---|
| UI | HTML + CSS, [Bootstrap 5 RTL](https://getbootstrap.com/), Font Awesome |
| Reactivity | [Alpine.js 3](https://alpinejs.dev/) |
| Tables | [Tabulator 5.5](https://tabulator.info/) |
| Dialogs | SweetAlert2 |
| Export | SheetJS (xlsx), jsPDF + autotable, html2pdf.js, html2canvas |
| Database | Firebase **Cloud Firestore** |
| Auth | Firebase Authentication — Email/Password, Google, **Phone (SMS OTP)** |
| Server logic | Firebase **Cloud Functions** (Node.js, Admin SDK) — region `me-west1` |
| Backup | Google Identity Services + Google Drive API |
| Hosting | GitHub Pages (any static host works) |

## Project layout

```
index.html      # the entire application (HTML + CSS + Alpine.js + Firebase)
privacy.html    # privacy policy
todo.txt        # working notes
old/, old2/     # archived numbered snapshots of earlier versions
google*.html    # Google site-verification file (needed for the Drive OAuth consent screen)
```

## Development pace

The project is built **iteratively and fast**, one real request from the field at a time. Numbers taken from the git history (as of 27 Oct 2026):

| | |
|---|---|
| First commit | 21 Apr 2026 |
| Commits | **157** in ~6 months, across 56 active days |
| Commits per month | Apr 5 · May 15 · Jun 29 · Jul 26 · Aug 21 · Sep 11 · **Oct 50** |
| Archived versions | ~37 numbered snapshots in `old/` and `old2/` |

The story so far: basic members + transactions (Alpine.js + Tabulator) → Firebase/Firestore as the single source of truth → admin authentication → funds, reports and exports → Google Drive backup → worshipper SMS login with server-side filtering → ongoing UX and reporting polish. The pace **accelerated** in October as the app moved from "one synagogue" to "ready to open for more synagogues".

## Running locally

There's nothing to build. Serve the folder with any static server (Firebase Auth does not work from `file://`):

```bash
git clone https://github.com/snirex/bk-tlamim.git
cd bk-tlamim
npx serve .          # or: python3 -m http.server 8080
```

Open `http://localhost:8080/?demo=1` for demo mode (no Firebase needed).

---

## 🚀 Deploying your own copy (new synagogue)

Each synagogue gets **its own Firebase project**. This keeps every congregation's financial data isolated, lets each gabbai own their data and billing, and avoids any cross-synagogue risk.

### 0. Prerequisites

- A Google account, and a GitHub account
- Node.js 20+ and the Firebase CLI: `npm i -g firebase-tools` then `firebase login`
- A **Firebase Blaze (pay-as-you-go) plan** — required for Cloud Functions and for SMS beyond the small free quota. A small congregation typically stays very low-cost; set a **budget alert** in Google Cloud.

### 1. Copy the code

Fork the repo, or create a new repo and push a copy:

```bash
git clone https://github.com/snirex/bk-tlamim.git my-shul
cd my-shul
git remote set-url origin https://github.com/<you>/<new-repo>.git
git push -u origin main
```

Then in GitHub: **Settings → Pages → Deploy from branch → `main` / root**. Your site will be at `https://<you>.github.io/<new-repo>/`.

### 2. Create the Firebase project

1. <https://console.firebase.google.com> → **Add project** (e.g. `my-shul-001`).
2. **Project settings → General → Your apps → Web app (`</>`)** → register an app and copy the `firebaseConfig`.
3. In `index.html`, replace the `firebaseConfig` object (search for `const firebaseConfig`, near the end of the file) with yours.
4. `firebaseConfig` values are **not secrets** — they identify the project. Security comes from Auth + Firestore rules + authorized domains (below).

### 3. Firestore

1. **Build → Firestore Database → Create database**, production mode, and pick a region (e.g. `eur3` or `me-west1`).
2. Deploy these rules (`firestore.rules`):

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // The gabbai's data: only the owner (a non-anonymous signed-in user with this uid)
    match /users/{uid}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }

    // Worshipper login diagnostics: write-only
    match /error_logs/{id} {
      allow create: if true;
      allow read, update, delete: if false;
    }

    // Everything else (including config/singleton) is server-only
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

Worshippers **never read Firestore directly** — their phone-auth uid is not the owner's uid, so the rules block them. They only talk to Cloud Functions.

#### Data model

```
users/{adminUid}/data/members          { list: [ { memberUniqueId, name, nickname, phone, email, ... } ] }
users/{adminUid}/data/transactions     { list: [ { transactionId, memberId, transactionType, date, commitmentAmount, paymentAmount, ... } ] }
users/{adminUid}/data/funds            { list: [...] }
users/{adminUid}/data/paymentPurposes  { list: [...] }
users/{adminUid}/data/config           columns, theme, member fields, Google Drive settings ...
users/{adminUid}/data/backupMeta       backup/restore history
config/singleton                       { ownerUid: "<adminUid>" }   ← read by Cloud Functions only
error_logs/{autoId}                    worshipper login diagnostics
```

> The app creates the empty `users/{uid}/data/*` documents on the admin's first login. `config/singleton` you create by hand (step 6).

### 4. Authentication

In **Build → Authentication → Sign-in method**, enable:

- **Email/Password** — the gabbai's login
- **Google** — optional, used for Google sign-in / Drive
- **Phone** — worshipper (and optional admin-phone) SMS login

Then:

1. **Settings → Authorized domains** → add your site domain (`<you>.github.io`). `localhost` is there by default.
2. **Users → Add user** → create the gabbai's email + password. **Copy that user's UID** — you need it for `config/singleton`.
3. **Phone provider → SMS region policy** → *Allow* **Israel (IL)** (or your country). Without this, SMS to local numbers is blocked.
4. **Phone provider → Phone numbers for testing** → add a fake number + code (e.g. `+972501234567` / `123456`) so you can test with no real SMS and no quota.
5. Optional but recommended: Firebase **App Check** (reCAPTCHA Enterprise / v3) to reduce SMS abuse and cost.

> Phone numbers are normalized to E.164 (`+9725XXXXXXXX`) by the client before being sent. The Cloud Functions must compare numbers in a format-insensitive way (see reference code below).

### 5. Cloud Functions (server side)

The worshipper screen calls exactly two **callable** functions, deployed in region **`me-west1`** (Tel Aviv), which must match `getFunctions(app, 'me-west1')` in `index.html`:

| Function | Input | Returns |
|---|---|---|
| `listMyMembers` | — (uses the caller's verified phone from `request.auth.token.phone_number`) | `{ members: [ { memberUniqueId, name, nickname } ] }` for every member with that phone |
| `getMemberData` | `{ memberUniqueId }` | `{ member, transactions, movementColumns }` — only if that member's phone matches the caller's |

> ⚠️ **Heads-up:** the Cloud Functions source is not currently committed to this repo. Add it under `functions/`. The block below is a **reference implementation** that satisfies the contract above — verify field names against your data before deploying.

```bash
firebase init functions     # JavaScript, Node 20, install dependencies
```

`functions/index.js`:

```js
const { onCall, HttpsError } = require("firebase-functions/v2/https");
const { setGlobalOptions } = require("firebase-functions/v2");
const admin = require("firebase-admin");

admin.initializeApp();
const REGION = "me-west1"; // keep in sync with getFunctions(app, 'me-west1') in index.html
setGlobalOptions({ region: REGION, maxInstances: 10 });
const db = admin.firestore();

// Compare by last 9 digits so +972501234567 === 050-1234567
const digits = (s) => String(s || "").replace(/\D/g, "");
const samePhone = (a, b) => digits(a).length >= 9 && digits(a).slice(-9) === digits(b).slice(-9);

async function ownerUid() {
  const snap = await db.doc("config/singleton").get();
  const uid = snap.exists && snap.data().ownerUid;
  if (!uid) throw new HttpsError("failed-precondition", "config/singleton.ownerUid is missing");
  return uid;
}
const readDoc = async (uid, name) => {
  const s = await db.doc(`users/${uid}/data/${name}`).get();
  return s.exists ? s.data() : {};
};
const callerPhone = (req) => {
  const p = req.auth && req.auth.token && req.auth.token.phone_number;
  if (!p) throw new HttpsError("unauthenticated", "Phone sign-in required");
  return p;
};

exports.listMyMembers = onCall(async (req) => {
  const phone = callerPhone(req);
  const { list = [] } = await readDoc(await ownerUid(), "members");
  const members = list
    .filter((m) => samePhone(m.phone, phone))
    .map((m) => ({ memberUniqueId: m.memberUniqueId, name: m.name, nickname: m.nickname }));
  return { members };
});

exports.getMemberData = onCall(async (req) => {
  const phone = callerPhone(req);
  const id = req.data && req.data.memberUniqueId;
  if (!id) throw new HttpsError("invalid-argument", "memberUniqueId is required");

  const uid = await ownerUid();
  const [{ list: members = [] }, { list: txs = [] }, config] = await Promise.all([
    readDoc(uid, "members"), readDoc(uid, "transactions"), readDoc(uid, "config"),
  ]);

  const member = members.find((m) => m.memberUniqueId === id);
  if (!member || !samePhone(member.phone, phone)) {
    throw new HttpsError("permission-denied", "This member does not belong to your phone number");
  }
  return {
    member: { memberUniqueId: member.memberUniqueId, name: member.name, nickname: member.nickname },
    transactions: txs.filter((t) => t.memberId === id),
    movementColumns: config.movementColumns || null,
  };
});
```

Deploy:

```bash
firebase use <your-project-id>
firebase deploy --only functions,firestore:rules
```

Troubleshooting: the worshipper screen distinguishes "wrong code" from "code accepted but server call failed". The latter usually means: functions not deployed, wrong region, or `config/singleton` missing.

### 6. Link the data owner (`config/singleton`)

In **Firestore → Start collection** → collection `config`, document ID `singleton`, field `ownerUid` (string) = the gabbai's UID from step 4.2.

### 7. Google Drive backup (optional)

1. <https://console.cloud.google.com> → select the same project → **APIs & Services → Library → enable "Google Drive API"**.
2. **OAuth consent screen** → External → add the app name, support email, and links to your site and `privacy.html`. Add the scope `.../auth/drive.file`. Publish to production (or add test users).
3. **Credentials → Create credentials → OAuth client ID → Web application**. Under *Authorized JavaScript origins* add your site origin (`https://<you>.github.io`).
4. Paste the Client ID into `defaultGoogleClientId` in `index.html` — **do not keep the original owner's ID** — or leave it empty and let the gabbai paste their own Client ID in *Settings → Google Drive*.
5. If Google asks to verify domain ownership, put the verification HTML file Google gives you in the repo root (like the existing `google*.html`).

### 8. Make it yours

- Update the footer/contact links and `privacy.html` (data controller, contact details).
- Change the site title and theme colors (theme is editable in the app settings).
- Remove the CallMeBot placeholder snippet if you don't use WhatsApp notifications.

### 9. First run checklist

- [ ] Open the site, sign in with the gabbai's email + password
- [ ] Add a member with your own phone number
- [ ] Log out → worshipper tab → enter that phone → receive SMS (or use the test number) → see the table
- [ ] Settings → connect Google Drive → create a backup → restore it
- [ ] `?demo=1` still works

### Common problems

| Symptom | Likely cause |
|---|---|
| `auth/unauthorized-domain` | Add the domain under Authentication → Settings → Authorized domains |
| `auth/operation-not-allowed` on SMS | Phone provider is disabled, or Israel isn't allowed in the SMS region policy |
| `auth/too-many-requests` / `quota` | SMS quota/abuse limits — use test numbers while developing, enable App Check |
| `auth/invalid-app-credential`, reCAPTCHA errors | Domain not authorized, or ad-blocker/privacy extension |
| "Code accepted but loading data failed" | Functions not deployed, wrong region, or `config/singleton` missing |
| `permission-denied` in Firestore | Rules not deployed, or the logged-in uid ≠ the data owner's uid |
| Drive `redirect_uri_mismatch` / `origin_mismatch` | Site origin missing from OAuth client's authorized JavaScript origins |

### Security notes

- Never commit service-account keys; the Admin SDK in Cloud Functions needs none.
- Keep `config/singleton` and all data inaccessible to clients (rules above).
- The old `readonly=true` URL bypass was removed — don't reintroduce client-side-only access control.
- Use a strong admin password and keep regular backups (Drive + a downloaded JSON copy).

---

# 🇮🇱 עברית

<div dir="rtl">

## סקירה

**SynApp** (שם הריפו: `bk-tlamim`) הוא אתר קל לניהול חשבונות המתפללים בבית הכנסת, עבור הגבאי: מתפללים, התחייבויות, תשלומים בפועל, קופות, דוחות וגיבויים.

המתפללים עצמם יכולים לצפות ב**חשבון שלהם בלבד**: מזינים מספר טלפון, מאמתים בקוד חד-פעמי ב-SMS, ורואים רק את התנועות שלהם. כל הסינון מתבצע בצד השרת (Cloud Functions), כך שאף מתפלל לא מקבל נתונים של אחרים.

אין תהליך build ואין שרת לתחזק: כל הצד הלקוח הוא קובץ `index.html` אחד שמוגש מ-GitHub Pages, עם Firebase כבסיס נתונים.

## תכונות עיקריות

**לגבאי (מנהל)**
- ניהול מתפללים (שם, כינוי/מזהה ייחודי, טלפון, אימייל; שדות ניתנים להגדרה)
- התחייבויות ותשלומים, ריכוז תנועות לכל מתפלל, צבעי יתרה (חוב באדום/מינוס, זכות בירוק/פלוס), ותמיכה בסכומים שליליים ליתרת פתיחה
- **קופות:** היסטוריה ויתרה לכל קופה, שיוך תשלומים והתחייבויות לקופה, ו"מטרות תשלום" גלובליות
- דוחות: ריכוז תנועות, מצב סיכום (שורת יתרה נטו לכל מתפלל), סינון וחיפוש מתפלל, בחירת עמודות מוצגות
- ייצוא ל-**Excel**, ל-**PDF** ולצילום **PNG** של טבלאות
- **גיבוי ושחזור** לקובץ JSON ול-**Google Drive** (הרשאת `drive.file`), כולל היסטוריית גיבויים
- כניסת מנהל באימייל וסיסמה, אפשרות כניסה עם Google ושחזור סיסמה
- מצב "תצוגה כמתפלל" לראות בדיוק מה המתפלל רואה
- עמודות וצבעי ערכת נושא הניתנים להתאמה, נשמרים ב-Firestore לכל בית כנסת
- שמירה אוטומטית מושהית ל-Firestore עם הגנה בעת סגירת הדף
- תיעוד שגיאות כניסת מתפללים ב-Firestore (`error_logs`)

**למתפללים**
- כניסה עם מספר טלפון וקוד SMS (Firebase Phone Auth עם reCAPTCHA)
- חשבונות משפחה: טלפון אחד יכול לעבור בין כמה מתפללים מקושרים
- טבלת תנועות לקריאה בלבד, בנויה כמו דוח "ריכוז תנועות" של המנהל
- קישור עם `?phone=` למילוי מראש של מספר הטלפון

**נוסף**
- ממשק מלא בעברית RTL, תאריכים בפורמט `DD/MM/YYYY`
- **מצב הדגמה** (`?demo=1`): פועל מקומית לגמרי עם נתוני דוגמה, בלי חיבור ל-Firebase
- דף מדיניות פרטיות (`privacy.html`) מקושר מעמוד הכניסה ומהתפריט

## טכנולוגיות

| שכבה | טכנולוגיה |
|---|---|
| ממשק | HTML + CSS, Bootstrap 5 RTL, Font Awesome |
| ריאקטיביות | Alpine.js 3 |
| טבלאות | Tabulator 5.5 |
| חלונות דו-שיח | SweetAlert2 |
| ייצוא | SheetJS, jsPDF + autotable, html2pdf.js, html2canvas |
| מסד נתונים | Firebase **Cloud Firestore** |
| אימות | Firebase Authentication: אימייל/סיסמה, Google, **טלפון (SMS)** |
| לוגיקת שרת | Firebase **Cloud Functions** (Node.js, Admin SDK), אזור `me-west1` |
| גיבוי | Google Identity Services + Google Drive API |
| אחסון האתר | GitHub Pages (כל אחסון סטטי מתאים) |

## קצב ההתקדמות

הפרויקט נבנה **בקצב מהיר ובאופן הדרגתי**, בקשה אמיתית אחת בכל פעם. הנתונים לקוחים מהיסטוריית git (נכון ל-27.10.2026):

| | |
|---|---|
| commit ראשון | 21.4.2026 |
| מספר commits | **157** בכ-6 חודשים, ב-56 ימי עבודה |
| לפי חודש | אפר׳ 5 · מאי 15 · יוני 29 · יולי 26 · אוג׳ 21 · ספט׳ 11 · **אוק׳ 50** |
| גרסאות בארכיון | כ-37 גרסאות ממוספרות ב-`old/` וב-`old2/` |

הסיפור עד כה: מתפללים ותנועות בסיסיים (Alpine.js + Tabulator) ← Firebase כמקור אמת יחיד ← אימות מנהל ← קופות, דוחות וייצוא ← גיבוי ל-Google Drive ← כניסת מתפללים ב-SMS עם סינון בצד שרת ← שיפורי חוויית משתמש ודוחות שוטפים. הקצב **הואץ** באוקטובר, עם המעבר מבית כנסת אחד להיערכות לפתיחת האתר לבתי כנסת נוספים.

## הרצה מקומית

אין מה לבנות. מגישים את התיקייה בשרת סטטי כלשהו (Firebase Auth לא עובד מ-`file://`):

```bash
git clone https://github.com/snirex/bk-tlamim.git
cd bk-tlamim
npx serve .          # או: python3 -m http.server 8080
```

פותחים `http://localhost:8080/?demo=1` למצב הדגמה (בלי Firebase).

---

## 🚀 הקמת עותק משלך (בית כנסת חדש)

לכל בית כנסת **פרויקט Firebase נפרד**. כך הנתונים הכספיים של כל קהילה מבודדים לגמרי, כל גבאי הוא הבעלים של הנתונים והחיוב שלו, ואין שום סיכון בין בתי כנסת.

### 0. דרישות מקדימות

- חשבון Google וחשבון GitHub
- Node.js 20+ ו-Firebase CLI: `npm i -g firebase-tools` ואז `firebase login`
- **תוכנית Blaze של Firebase (תשלום לפי שימוש)**, נדרשת ל-Cloud Functions ול-SMS מעבר למכסה החינמית הקטנה. בקהילה קטנה העלות בדרך כלל נמוכה מאוד; מומלץ להגדיר **התראת תקציב** ב-Google Cloud.

### 1. העתקת הקוד

עושים Fork לריפו, או יוצרים ריפו חדש ודוחפים אליו עותק:

```bash
git clone https://github.com/snirex/bk-tlamim.git my-shul
cd my-shul
git remote set-url origin https://github.com/<you>/<new-repo>.git
git push -u origin main
```

ב-GitHub: **Settings ← Pages ← Deploy from branch ← `main` / root**. האתר יעלה בכתובת `https://<you>.github.io/<new-repo>/`.

### 2. יצירת פרויקט Firebase

1. <https://console.firebase.google.com> ← **Add project** (למשל `my-shul-001`).
2. **Project settings ← General ← Your apps ← Web app (`</>`)** ← רושמים אפליקציה ומעתיקים את `firebaseConfig`.
3. ב-`index.html` מחליפים את האובייקט `firebaseConfig` (מחפשים `const firebaseConfig`, קרוב לסוף הקובץ) בשלך.
4. ערכי `firebaseConfig` **אינם סוד**, הם מזהים את הפרויקט. האבטחה נשענת על Auth, חוקי Firestore ודומיינים מורשים (בהמשך).

### 3. Firestore

1. **Build ← Firestore Database ← Create database**, במצב production, ובוחרים אזור.
2. פורסים את החוקים שבגרסה האנגלית למעלה (`firestore.rules`) באמצעות `firebase deploy --only firestore:rules`.

מתפללים **לעולם לא קוראים מ-Firestore ישירות**. ה-uid שלהם (מאימות טלפון) שונה מה-uid של הבעלים, ולכן החוקים חוסמים אותם. הם מדברים רק עם Cloud Functions. מבנה הנתונים מפורט בגרסה האנגלית (*Data model*).

### 4. אימות (Authentication)

ב-**Build ← Authentication ← Sign-in method** מפעילים: **Email/Password** (כניסת הגבאי), **Google** (אופציונלי), **Phone** (כניסת מתפללים ב-SMS).

אחר כך:

1. **Settings ← Authorized domains** ← מוסיפים את דומיין האתר (`<you>.github.io`).
2. **Users ← Add user** ← יוצרים משתמש לגבאי (אימייל + סיסמה) ו**מעתיקים את ה-UID שלו**.
3. **Phone ← SMS region policy** ← מאפשרים **ישראל (IL)**. בלי זה SMS למספרים מקומיים נחסם.
4. **Phone ← Phone numbers for testing** ← מוסיפים מספר בדיוני וקוד (למשל `+972501234567` / `123456`) כדי לבדוק בלי SMS אמיתי ובלי לצרוך מכסה.
5. מומלץ: להפעיל **App Check** כדי להפחית ניצול לרעה ועלויות SMS.

### 5. Cloud Functions (צד שרת)

מסך המתפלל קורא לשתי פונקציות **callable** בלבד, שנפרסות באזור **`me-west1`** (תל אביב), והאזור חייב להתאים ל-`getFunctions(app, 'me-west1')` ב-`index.html`:

| פונקציה | קלט | פלט |
|---|---|---|
| `listMyMembers` | אין (משתמשת בטלפון המאומת של הקורא) | רשימת המתפללים שמספר הטלפון שלהם תואם |
| `getMemberData` | `{ memberUniqueId }` | `{ member, transactions, movementColumns }`, רק אם הטלפון של המתפלל תואם לקורא |

> ⚠️ **שים לב:** קוד ה-Cloud Functions עדיין לא נמצא בריפו הזה. יש להוסיף אותו בתיקיית `functions/`. בגרסה האנגלית למעלה יש **מימוש ייחוס** שעונה על החוזה, ויש לוודא את שמות השדות מול הנתונים שלך לפני הפריסה.

```bash
firebase init functions            # JavaScript, Node 20
firebase use <your-project-id>
firebase deploy --only functions,firestore:rules
```

אם מסך המתפלל מציג "האימות הצליח, אך טעינת הנתונים נכשלה", הסיבה היא בדרך כלל: הפונקציות לא נפרסו, אזור שגוי, או ש-`config/singleton` חסר.

### 6. קישור בעל הנתונים (`config/singleton`)

ב-**Firestore ← Start collection**: אוסף `config`, מסמך `singleton`, שדה `ownerUid` (מחרוזת) = ה-UID של הגבאי משלב 4.2.

### 7. גיבוי ל-Google Drive (אופציונלי)

1. ב-<https://console.cloud.google.com>, באותו פרויקט: **APIs & Services ← Library ← Google Drive API ← Enable**.
2. **OAuth consent screen** ← External ← שם אפליקציה, אימייל תמיכה וקישורים לאתר ול-`privacy.html`. מוסיפים את ההרשאה `drive.file` ומפרסמים ל-production (או מוסיפים משתמשי בדיקה).
3. **Credentials ← OAuth client ID ← Web application**. תחת *Authorized JavaScript origins* מוסיפים את כתובת האתר (`https://<you>.github.io`).
4. מדביקים את ה-Client ID ב-`defaultGoogleClientId` ב-`index.html`. **אל תשאירו את ה-ID של הבעלים המקורי.** אפשר גם להשאיר ריק ולתת לגבאי להדביק Client ID משלו ב-*הגדרות ← Google Drive*.
5. אם Google מבקשת אימות בעלות על הדומיין, מוסיפים את קובץ ה-HTML שהיא נותנת לשורש הריפו (כמו קובץ `google*.html` הקיים).

### 8. התאמה אישית

- לעדכן קישורי כותרת תחתונה/יצירת קשר ואת `privacy.html` (פרטי הגורם האחראי).
- לשנות כותרת אתר וצבעי ערכת נושא (ניתן בהגדרות).
- להסיר את קטע ה-CallMeBot אם לא משתמשים בהתראות WhatsApp.

### 9. רשימת בדיקה לריצה ראשונה

- [ ] להיכנס לאתר עם אימייל וסיסמת הגבאי
- [ ] להוסיף מתפלל עם מספר הטלפון שלך
- [ ] להתנתק ← לשונית מתפלל ← להזין את הטלפון ← לקבל SMS (או להשתמש במספר הבדיקה) ← לראות את הטבלה
- [ ] הגדרות ← חיבור Google Drive ← גיבוי ← שחזור
- [ ] `?demo=1` עדיין עובד

### בעיות נפוצות

| תסמין | סיבה סבירה |
|---|---|
| `auth/unauthorized-domain` | להוסיף את הדומיין ב-Authorized domains |
| `auth/operation-not-allowed` בשליחת SMS | ספק הטלפון כבוי, או שישראל לא מאושרת ב-SMS region policy |
| `auth/too-many-requests` / מכסה | מגבלות SMS: להשתמש במספרי בדיקה בפיתוח ולהפעיל App Check |
| שגיאות reCAPTCHA / `invalid-app-credential` | דומיין לא מורשה, או חוסם פרסומות/תוסף פרטיות |
| `permission-denied` ב-Firestore | החוקים לא נפרסו, או שה-uid המחובר שונה מה-uid של הבעלים |
| `origin_mismatch` ב-Drive | כתובת האתר חסרה ב-Authorized JavaScript origins של ה-OAuth client |

### הערות אבטחה

- לעולם לא מעלים מפתחות service account לריפו; ל-Admin SDK בתוך Cloud Functions אין בהם צורך.
- שומרים את `config/singleton` וכל הנתונים סגורים בפני לקוחות (החוקים למעלה).
- עקיפת `readonly=true` בכתובת הוסרה. אין להחזיר בקרת גישה בצד לקוח בלבד.
- סיסמת מנהל חזקה וגיבויים קבועים (Drive + עותק JSON מקומי).

</div>
