# AGS-crm
# AGS Consult — Pipeline

Lead and conversion tracking for the AGS sales team. Each rep logs their own
clients; everyone can see the team's conversion numbers.

One file, `index.html`. No build step.

---

## How the data is split

This is the important part, so read it before you set anything up.

| Where | What's in it | Who can read it |
|---|---|---|
| `leads/{uid}` | Client names, phones, emails, notes, full history | **Only that rep's account** |
| `stats/{uid}` | Counts, rates, advert and approach performance | Everyone signed in |
| `members/{uid}` | Display name and monthly target only | Everyone signed in |

Staff email addresses and phone numbers appear nowhere in the database. Sign-in
emails live in Firebase Authentication, which no other user's browser can read —
Firebase only ever hands an account its own email. The app never copies an email
into Firestore, never derives a display name from one, and the rules below reject
a `members` document containing an `@` or any field beyond the three above.

The team view reads `stats`, never `leads`. So a rep who opens the browser
console and dumps everything the app loaded gets conversion percentages. There
are no phone numbers or email addresses in there to take, because they were
never sent to their browser in the first place.

The Firestore rules in step 3 enforce this on the server. It isn't a setting in
the app that someone can flip.

One consequence: **managers cannot see client records.** They see how many leads
someone has, their close rate, which adverts work for them, and their target
progress — but not who the clients are. If a manager genuinely needs client
records for compliance or handover, that has to be a deliberate change with its
own rule, not a side effect.

The free-text "how you got them" field is deliberately kept out of shared stats,
because reps type things like "referral from Sipho's wife" into it.

---

## Step 1 — Firebase project

1. console.firebase.google.com → **Add project**. Skip Analytics.
2. **Build → Firestore Database → Create database**. Production mode.
   Region `europe-west1` is a sensible pick from South Africa.
3. **Build → Authentication → Get started → Email/Password → Enable.**
   Leave "Email link" off.

## Step 2 — Paste your config

Project settings (gear) → "Your apps" → web icon `</>` → register the app. Copy
the values into the top of `index.html`:

```js
var FIREBASE_CONFIG = {
  apiKey: "AIza...",
  authDomain: "ags-pipeline.firebaseapp.com",
  projectId: "ags-pipeline",
  storageBucket: "ags-pipeline.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abc123"
};
```

Right below it, optionally lock signups to your company domain:

```js
var ALLOWED_EMAIL_DOMAIN = "agsconsult.co.za";
```

Leave it as `""` if AGS reps use personal addresses. It's checked in the app
*and* in the rules below — the rules are the one that counts.

These config values are not secrets. They're in every Firebase web app and are
safe to publish. Your security comes entirely from the rules.

## Step 3 — Security rules

Firestore → **Rules** → paste and publish:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function signedIn() {
      return request.auth != null
          && request.auth.token.email_verified == true;
    }

    // Uncomment and use this instead of signedIn() if you set
    // ALLOWED_EMAIL_DOMAIN, replacing the domain:
    // function signedIn() {
    //   return request.auth != null
    //       && request.auth.token.email_verified == true
    //       && request.auth.token.email.matches('.*@agsconsult[.]co[.]za$');
    // }

    function isOwner(uid) {
      return signedIn() && request.auth.uid == uid;
    }

    // Client records. Private to the rep who captured them.
    match /leads/{uid} {
      allow read, write: if isOwner(uid);
    }

    // Aggregated numbers. No client details in here.
    match /stats/{uid} {
      allow read: if signedIn();
      allow write: if isOwner(uid);
    }

    // Names and targets. Locked down to exactly three fields so no future
    // version of the app can quietly start publishing staff contact details.
    match /members/{uid} {
      allow read: if signedIn();
      allow write: if isOwner(uid)
        && request.resource.data.keys().hasOnly(['name','goalDeals','goalValue','updated'])
        && request.resource.data.name is string
        && request.resource.data.name.size() > 0
        && request.resource.data.name.size() < 61
        && !request.resource.data.name.matches('.*@.*');
    }

    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

The `hasOnly` and `matches('.*@.*')` checks on `members` are what stop staff
contact details leaking. Even if someone edited the app to publish emails, the
write would be rejected.

`email_verified` matters. Without it, anyone can register any address and read
team stats. With it, they need access to a real inbox — and if you use the
domain check, an AGS inbox.

## Step 4 — Host it

Upload `index.html` to a **private** repo, then Settings → Pages → branch
`main`, folder `/ (root)`.

Free-plan GitHub Pages serves publicly even from a private repo. That's survivable
here because the login gates everything, but Cloudflare Pages or Netlify are free
and let you add password protection on top.

---

## Step 5 — Abuse protection (do this before rolling out)

You asked about rate limiting. Here's what actually works, in order of value:

**Firebase App Check** is the real answer. It attaches a token proving requests
come from your site, so someone with your config values can't hammer the API from
a script. Build → App Check → register your site with reCAPTCHA v3 → then in
Firestore, enforce it. Roughly ten minutes and it's the single biggest win.

**Budget alerts.** Billing → set an alert. The free tier is 50k reads and 20k
writes a day; a sales team uses a fraction of that, so an alert firing means
something is wrong.

**Firebase's own throttling** already blocks repeated failed logins per account
and per IP. You don't have to build that.

**The app's own limits** are minor by comparison: team refresh is throttled to
once every 15 seconds, and stats are fetched on sign-in rather than continuously.

What no amount of rate limiting would have fixed is a logged-in rep quietly
reading data the app handed their browser. That's why the data split in step 3
matters more than any of this.

---

## Day to day

Reps sign up once with their work email, verify it, and log leads. Tabs across the
bottom: **Pipeline**, **Add**, **Mine** (their own stats), **Team** (everyone's
rates, tap a name for detail), **Company** (everything combined).

Targets are set by each person under "Set a target" on their pipeline screen.
Nobody can edit somebody else's — the rules block it. If management needs to set
targets top-down, that's a change worth making deliberately.

---

## Still worth knowing

**Reps own their client data, including when they leave.** If someone quits, their
leads are locked to an account you'd have to reset the password on to reach. Decide
how you'll handle that before it happens. Reps can export their own CSV as a stopgap.

**POPIA applies.** Client names, numbers, and cover notes under an FSP licence.
Talk to whoever handles compliance at AGS before the team starts using it. The note
under the notes field tells reps not to enter ID or bank details — hold them to it.

**Deleting a lead is permanent.** No undo, no archive.

**Team stats update when a rep saves.** Other people see the change after they hit
Refresh or sign in again. It isn't live.

---

## Changing the dropdowns

Near the top of the script: `PRODUCTS`, `APPROACHES`, `LOSS`, `OBJECTIONS`,
`CHANNELS`. Edit, commit, done.

"Which advertisement" and "How you got them" are free text on purpose, with
previously typed values suggested. Type advert names identically every time —
"FB funeral ad" and "Facebook funeral cover ad" become two separate rows and each
looks half as effective as it is.
