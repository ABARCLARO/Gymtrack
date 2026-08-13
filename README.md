# GymTrack

A training log built as an installable PWA. No account, no server, no dependencies —
one HTML file, a manifest, and a service worker.

## Log types

Every exercise declares how a set is measured. The entry row, the personal-best
comparison and the chart all branch on it.

| Type | Fields | Personal best | Examples |
|---|---|---|---|
| `weighted` | kg × reps | weight × reps | bench, squat, rows, curls |
| `bodyweight` | reps, optional +kg | (bodyweight + added) × reps | pull-ups, dips, hanging leg raises |
| `timed` | seconds | longest hold | side plank, dead hang, wall sit |
| `carry` | kg × metres | weight × distance | farmer's carry, sled push |

Set the type when adding or editing any exercise. Timed exercises get a **time a hold**
button that counts up and writes the result straight into the set.

Set your bodyweight in Settings so bodyweight exercises can be scored — that's what lets
12 strict pull-ups be compared against 8 with a 10 kg belt.

## What it does

- **Any split.** Add, rename, reorder and delete training days and exercises. Ships with
  push/pull/legs as a starting point.
- **Last session's numbers** print under every exercise, with your all-time best beside them.
- **Beat detection.** A set turns green with a ▲ the moment it out-scores your previous best.
- **Rest timer** starts when a set is complete. Chimes and vibrates when it's up.
- **Progress charts** per exercise, with the change since your first session.
- **Export / import** your whole log as JSON.
- Works offline, installs to the home screen.

## Deploy

Push every file to a repo root, then Settings → Pages → Deploy from a branch → `main` / root.

```
index.html  manifest.json  sw.js
icon-180.png  icon-192.png  icon-512.png  icon-512-maskable.png
```

## Upgrading

Nothing to do. On first load v3 reads any `gymtrack:v2` or `gymtrack:v1` log and migrates it:

- Renamed exercises carry their history across, so personal bests survive
  (`Barbell curl` → `Dumbbell curl`, `Wrist curl + reverse` → `Wrist curl`,
  `Dip or decline press` → `Dip`, `Pull-up or lat pulldown` → `Lat pulldown`).
- Numbers logged in the wrong field are moved to the right one — a side plank stored as
  `0 × 45` becomes a 45-second hold rather than being dropped.
- Anything already committed keeps the format it was logged in.

## Data

Everything lives in `localStorage` on the device. **Clearing browser data deletes it and
there is no recovery** — use Export in Settings to keep a backup.

See [SECURITY.md](SECURITY.md) for the threat model and audit findings.
