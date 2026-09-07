# Djed's Spidey Treasure Hunt App

A digital companion to Djed's 4th birthday party — a Heroes & Villains treasure hunt through the Iziko South African Museum (13 September 2026). Guests RSVP, get randomly assigned a hero or villain character, spin a wheel to pick a co-leader for each stop, and browse a shared photo capsule afterwards.

No backend to run — plain HTML/CSS/JS backed by a free Firebase Realtime Database, deployed on GitHub Pages.

## 1. Set up Firebase (~5 minutes, no credit card)

1. Go to https://console.firebase.google.com and create a project (any name, e.g. "djed-party").
2. In the project, click the `</>` (web) icon to add a web app.
3. Copy the `firebaseConfig` object it gives you into [`firebase-config.js`](firebase-config.js), replacing the `REPLACE_ME` values.
4. In the left menu go to **Build → Realtime Database → Create Database**. Choose **Start in test mode** for the quickest setup.
5. Once the party is set up and before you send the real link to family, go to the Realtime Database **Rules** tab and paste in:

   ```json
   {
     "rules": {
       "guests": { ".read": true, ".write": true },
       "rsvps": { ".read": true, ".write": true },
       "settings": { ".read": true, ".write": true },
       "coLeader": { ".read": true, ".write": true },
       "characterQueue": { ".read": true, ".write": true },
       "characterQueueIndex": { ".read": true, ".write": true },
       "adminAuth": { ".read": true, ".write": true }
     }
   }
   ```

   **Note on `adminAuth`:** the original plan called for `adminAuth.read: false` so the admin password couldn't be read by guests. That turned out to be incompatible with a client-side password check — with no server, the app has to read the stored password to compare it, which means it's also readable by anyone who opens their browser's network tab. `adminAuth.read: true` above is what makes the admin login actually function. This is the same tradeoff already flagged for the admin gate itself (see Section 3 below): it deters casual guests, it does not stop someone who goes looking. Fine for a private family party; don't reuse this pattern for anything that needs real security.

6. Change the default admin password (`spidey2026`) from inside the app's Admin panel the first time you log in, or edit `DEFAULT_ADMIN_PASSWORD` in `firebase-config.js` before your first deploy.

## 2. Deploy on GitHub Pages

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin <your-repo-url>
git push -u origin main
```

Then in the repo on GitHub: **Settings → Pages → Deploy from branch → main → /(root)**. No build step, no GitHub Actions needed.

## 3. What to know before you hand out the link

- **Shared data is visible to anyone with the link.** Guest names, photos, and characters aren't behind a login. Already an accepted tradeoff for a private family party — just worth remembering once this is a real public URL.
- **The admin password gate is a deterrent, not real security.** There's no server, so there's no way to fully hide it from someone determined to look (see the `adminAuth` note above). It keeps curious guests out of Settings; it won't stop someone in dev tools.
- **Photos are compressed client-side** (max 480px, JPEG quality 0.6) before they're written to the database — this keeps you well within Firebase's free tier even with 15+ guests. Don't remove this step.
- **A guest who taps "Get My Mission" but never finishes Suit Up still consumes a character-queue slot.** This keeps the balancing logic simple (see below) at the cost of an occasional skipped character — a non-issue at party scale.

## 4. How character balancing works

With only 6 canonical characters (Spidey, Ghost-Spider, Spin, Doc Ock, Rhino, Green Goblin) and likely more than 6 guests, duplicates are inevitable — the goal is that they happen evenly instead of by chance (no "5 Spideys, 0 Rhinos").

`characterQueue` in the database holds a shuffled, repeating cycle of all 6 characters. Each check-in atomically claims the next index via a Firebase transaction on `characterQueueIndex`, so two simultaneous check-ins can't claim the same slot. When the claimed index runs past the end of the current queue, the queue is extended by another full shuffled cycle before the assignment is read — so duplicates always arrive in complete, evenly-distributed batches ("Rhino II" is guaranteed to arrive alongside a second full cycle, not by chance). This also means arbitrary or unexpected guest counts (walk-ins beyond RSVP numbers) are handled automatically with no separate pool-sizing step.

## 5. Repo structure

```
djed-party-app/
  index.html         structure, loads styles + firebase-config.js + app.js
  styles.css          comic-book pop-art design system
  firebase-config.js  your Firebase project config + default admin password
  app.js              all application logic
  README.md           this file
```

## 6. Open items / nice-to-haves not required for v1

- Expanding the character roster beyond the confirmed six (e.g. adding Hulk, who appears on the sticker book's cover art).
- The Suit Up photo capture defaults to the front camera (`capture="user"`) since it's meant as a personal check-in photo. Switch to `capture="environment"` in `app.js` if you'd rather it default to the rear camera.
