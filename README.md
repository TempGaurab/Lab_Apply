# CEAS Outreach Tracker

A single-page site for tracking lab-application outreach to University of
Cincinnati CEAS faculty: every faculty member across six departments, their
UC email, research focus, and a "Reached out" / "Is hiring" status you set
yourself.

Works immediately with no setup — your answers save to the browser's local
storage. Follow the steps below to also sync them to a free cloud database
so the same list, with the same answers, shows up on your phone, laptop,
or any other device.

## Run it locally

No build step. Just open `index.html` in a browser, or serve the folder:

```
npx serve .
```

## Set up free cloud sync (Firebase Firestore)

This uses Firebase's free "Spark" plan — no credit card required, and its
free quota (50k reads / 20k writes per day) is far more than one person
tracking ~135 faculty will ever use.

1. Go to the [Firebase console](https://console.firebase.google.com/) and
   click **Add project**. Give it any name (e.g. `ceas-outreach-tracker`).
   You can decline Google Analytics — it isn't needed.
2. Inside the project, open **Build → Firestore Database** in the left
   sidebar, click **Create database**, pick a location close to you, and
   start it in **test mode** (you'll lock it down with the rules below).
3. Open **Project settings** (gear icon, top left) → scroll to **Your
   apps** → click the **`</>`** (web) icon → give the app any nickname →
   **Register app**. Firebase shows you a `firebaseConfig` object.
4. Copy `.env.example` to `.env` and fill in the seven values from that
   config object. `.env` is gitignored — it never gets committed.
5. Copy `firebase-config.example.js` to `firebase-config.js` (also
   gitignored) and fill in the same values, so the page has something to
   load when you open it locally:

   ```
   cp .env.example .env
   cp firebase-config.example.js firebase-config.js
   ```

   then edit both with the values from step 3. Keeping the same values in
   two places is only needed because this is a plain static site with no
   build step to inject `.env` into the browser at request time — `.env`
   is the source of truth you copy from (and what the GitHub Actions
   deploy in the next section reads from instead, via repository secrets),
   `firebase-config.js` is what the browser actually loads locally.
6. Back in the Firebase console, go to **Firestore Database → Rules** and
   replace the default rules with:

   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /responses/{email} {
         allow read, write: if true;
       }
       match /{document=**} {
         allow read, write: if false;
       }
     }
   }
   ```

   This opens read/write only on the `responses` collection this app uses
   — nothing else in the database is reachable. Because there's no login
   step in this simple version, anyone who has both your site's URL *and*
   your Firebase config values could read or edit your tracker. That's a
   reasonable trade-off for personal, non-sensitive outreach notes; if you
   want it locked to just you, the next step up is adding Firebase
   Anonymous Auth (or Google sign-in) and changing the rule to check
   `request.auth != null` — ask if you'd like that added.
7. Reload the page. The line under the title will switch from "Local-only
   mode" to "Synced" once it connects.

## Put it online (free, GitHub Pages)

This repo already has a GitHub remote, plus a workflow at
`.github/workflows/deploy.yml` that builds `firebase-config.js` from
repository secrets at deploy time — so the real key values never sit in
git history, only in GitHub's encrypted secrets store.

1. On GitHub: **Settings → Secrets and variables → Actions → New
   repository secret**, and add all seven names from `.env.example`
   (`FIREBASE_API_KEY`, `FIREBASE_AUTH_DOMAIN`, `FIREBASE_PROJECT_ID`,
   `FIREBASE_STORAGE_BUCKET`, `FIREBASE_MESSAGING_SENDER_ID`,
   `FIREBASE_APP_ID`, `FIREBASE_MEASUREMENT_ID`), each with the matching
   value from your own `.env`.
2. **Settings → Pages → Source: GitHub Actions** (not "Deploy from a
   branch").
3. Push:

   ```
   git add index.html firebase-config.example.js README.md .gitignore .env.example .github
   git commit -m "Add CEAS outreach tracker"
   git push
   ```

   The workflow runs automatically, generates `firebase-config.js` from
   your secrets, and publishes the site. GitHub gives you a URL like
   `https://<username>.github.io/Lab_Apply/` a minute or two later —
   bookmark that on your phone and laptop.

One honest caveat: a static site has no server to hide anything behind,
so the deployed page's `firebase-config.js` still contains your real
Firebase values in plain view (anyone can view-source it) — that's true
of every Firebase web app, secret or not, which is why access control
lives in Firestore's rules rather than in hiding this config. What the
env-file setup buys you is keeping those values out of your git history
and off GitHub's public "Files changed" view, and making them easy to
rotate in one place (the secrets store) if you ever need to.

## Notes on the data

Faculty names, titles, emails, and research notes were compiled from each
department's public CEAS directory page in September 2026. Appointments
and contact details change — double-check against
[ceas.uc.edu](https://www.ceas.uc.edu) before relying on an entry. Two
people were left out because their source listing had no usable email
(George Suckarieh in Civil Engineering, Amit Bhattacharya in Biomedical
Engineering) — look them up directly if you need them.
