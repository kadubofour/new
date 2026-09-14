# University of Ghana Sports and Wellness Directorate — Multi-Activity Registration System

Five activities, one Google Sheets backend, four static front-ends.

## Activities & member codes

| Activity | Code prefix | Example | Categories | Plans |
|---|---|---|---|---|
| Gymnasium | `G` | `G1234567` | UG Student, UG Staff, Non-UG Student, Public | Walk-in, Monthly, Semesterly (UG Student only), Quarterly, Half-yearly, Yearly |
| Leisure Tennis | `T` | `T1234567` | UG Student, UG Staff, UG Staff Relation (Under 17 / 17 & Above), Public Child (Under 17), Public Adult (17 & Above), Family Package (Max 5) | Walk-in, Monthly only |
| Leisure Swimming | `S` | `S1234567` | Same 7 categories as Leisure Tennis | Walk-in, Monthly, Semesterly (UG Student only), Quarterly, Half-yearly, Yearly |
| Tennis Lessons | `TL` | `TL1234567` | Same 7 categories | Walk-in, Monthly (30-day window) |
| Swimming Lessons | `SL` | `SL1234567` | Same 7 categories | **12-Session Package only** — 6-week (42-day) window capped at 12 sessions, whichever limit hits first ends the package. No Walk-in. |

A code is always `<prefix>` + 7 digits, auto-generated for every category except UG Student / UG Staff, who keep entering their own student/staff ID unprefixed. Only Gym keeps the plain 4-category set (UG Student / UG Staff / Non-UG Student / Public) — the other four activities all use the 7-category set above, per the printed rate cards.

### Family Package registrations

"Family Package (Max 5)" is **not** one row for the whole family. The person filling out the form is the primary registrant (full details captured as normal — DOB, gender, medical, photo, signature); each additional family member they list by full name (up to 4 more) gets their own lightweight row with its own auto-generated code, sharing the same phone/email/address/emergency contact. Because every row in a family shares one phone number, the family head can retrieve every member's code at once from the Sign In tab's "Don't have your code?" phone lookup — no need to write each one down separately at the front desk.

### Swimming Lessons session counting

A Swimming Lessons session is only counted as "used" when the member **signs out**, not when they sign in — so an in-progress visit, or one where they forgot to sign out, doesn't burn a session early or accidentally.

## Files

- **`Code.gs`** — the Google Apps Script backend (JSON API). Deploy this as a Web App; every front-end below talks to the same `/exec` URL.
- **`index.html`** (formerly `registration-app.html`) — the public-facing app: pick a program, then Register / Renew / Sign In / Sign Out. Share this one link with members. Named `index.html` so it's the default page GitHub Pages (or any static host) serves at the bare repo/domain root.
- **`front-desk-dashboard.html`** — the main front desk. Approves/rejects everything, for all 5 activities (switch between them with the activity pills at the top), plus registration tables, visit logs, and Excel export.
- **`tennis-front-desk.html`** — satellite front desk for Leisure Tennis + Tennis Lessons. Shows full registrant detail and the visit log; can only approve/reject **walk-ins** (enforced by the backend, not just hidden in the UI) — new registrations and renewals still need the main front desk.
- **`swimming-front-desk.html`** — same, for Leisure Swimming + Swimming Lessons.

## Setup

1. Create a new Google Sheet (sheets.new).
2. Extensions → Apps Script. Delete any starter code, paste in `Code.gs`, save.
3. From the function dropdown, select `setup`, click Run. Authorize when asked. This creates 20 tabs (Pending / Registrations / Visits / Alerts × 5 activities), each with the right headers.
4. Deploy → New deployment → gear icon → Web app.
   - Execute as: **Me**
   - Who has access: **Anyone**
   
   Deploy, authorize (Drive access is needed for photos/exports), copy the URL ending in `/exec`.
5. Paste that URL into `SCRIPT_URL` near the top of **each** of the four HTML files (`index.html`, `front-desk-dashboard.html`, `tennis-front-desk.html`, `swimming-front-desk.html`) — they all share the same backend.
6. Host the four HTML files wherever you like (a static host, or just open them locally) and distribute the links: `index.html` to members, `front-desk-dashboard.html` to the main desk, `tennis-front-desk.html` to the tennis court desk, `swimming-front-desk.html` to the pool desk.
7. Any time `Code.gs` is edited again: Deploy → Manage deployments → pencil icon → Version: New version → Deploy, or the live URL won't see the change.

Default staff PIN on every front-desk app is `1234` — change the `STAFF_PIN` constant near the top of each file's `<script>` before going live.

See the large header comment at the top of `Code.gs` for details on photo/signature storage, the date-grouped Registrations sheet, walk-ins, renewals, and the Excel export.

## Live sync (optional)

Without this section, the three front-desk dashboards already work exactly as before: each one polls the Apps Script backend every `AUTO_REFRESH_MS` (5 seconds) and only re-renders whatever actually changed. That means a change made at one desk can take up to a few seconds to show up at another.

Setting this up adds a push: the instant any desk writes something (an approval, a sign-in/out, a walk-in, a renewal...), Code.gs pushes a tiny "something changed" timestamp for that activity to a small Firebase Realtime Database tree — deliberately not the dashboard data itself, so the write action a staff member is waiting on doesn't pay for building and shipping it. Every open dashboard with a live listener sees that timestamp change and immediately does the exact same fetch its poll would have done anyway, just without waiting for the next `AUTO_REFRESH_MS` tick. The poll itself is never removed — it keeps running as a fallback for a dropped connection, so a Firebase outage or a typo in the config just silently falls back to "polls every 5 seconds," not "broken."

**It's entirely optional.** Skip this section and nothing else in this repo changes behavior.

### 1. Create the Firebase project

1. Go to the [Firebase console](https://console.firebase.google.com/) → Add project (the free Spark plan is plenty for this scale).
2. Build → Realtime Database → Create Database. Pick a location close to your users. Start in **locked mode**.
3. Rules tab — paste this (open read, so any front desk can listen; writes blocked entirely for normal clients):
   ```json
   { "rules": { ".read": true, ".write": false } }
   ```
4. Project settings (gear icon) → Service accounts tab → **Database secrets** → copy the legacy secret shown there. A legacy secret is treated as full-admin by Firebase and bypasses `.write: false` automatically when passed as `?auth=<secret>` on a REST call — that's what `Code.gs`'s `firebasePut()` does, and it's what `FIREBASE_DB_SECRET` below is for. No further rules-writing needed.
5. Project settings (gear icon) → General → "Your apps" → Add app → Web (`</>`). Register it (no hosting needed). Copy the `firebaseConfig` object it shows you — you only need `apiKey` and `databaseURL` out of it.

### 2. Configure the backend

In the Apps Script editor: Project Settings (gear icon) → Script Properties → Add property, twice:

| Property | Value |
|---|---|
| `FIREBASE_DB_URL` | The `databaseURL` from step 1.5, e.g. `https://your-project-default-rtdb.firebaseio.com` |
| `FIREBASE_DB_SECRET` | The legacy database secret from step 1.4 (leave unset if your rules don't need one) |

Nothing else changes — `touchActivity()` in `Code.gs` (called from every action that writes to Pending/Registrations/Visits) already calls `pushLiveState()`, which no-ops silently until `FIREBASE_DB_URL` is set.

### 3. Configure each dashboard

Paste the same `apiKey`/`databaseURL` from step 1.5 into the `FIREBASE_CONFIG` object near the top of each front-desk file's `<script>` — `front-desk-dashboard.html`, `swimming-front-desk.html`, `tennis-front-desk.html` (not `index.html` — the public registration form doesn't need this). This config is safe to leave in client-side code; it identifies the project, it isn't a secret — access is controlled by the database rules from step 1.3, not by hiding this object.

```js
const FIREBASE_CONFIG = {
  apiKey: "AIza...",
  databaseURL: "https://your-project-default-rtdb.firebaseio.com"
};
```

Reload the dashboard — a change at any desk (or the registration app) should now appear at every other open dashboard within a second or so, instead of on the next poll.
