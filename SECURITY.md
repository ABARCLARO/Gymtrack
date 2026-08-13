# GymTrack — security review

Reviewed: v1 (deployed) and v2 (this release). GymTrack is a client-side app with no
server, no accounts, and no network calls of its own, so the attack surface is small —
but "small" is not "none", and v1 had one real bug.

## Threat model

There is no backend, so there is nothing to breach remotely and no other user's data to
reach. What remains:

1. **Self-XSS / stored XSS** — text the user types (or imports) being rendered as HTML.
2. **Malicious import files** — once import exists, a file from someone else is untrusted input.
3. **Data durability** — `localStorage` is not a safe place for a year of training history.
4. **Third-party requests** — anything the page fetches from outside your own domain.

---

## Findings

### GT-01 · Attribute-context XSS — **Medium** — found in v1, fixed in v2

v1 escaped with:

```js
function esc(s){return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');}
```

That handles text nodes but **not quotes**, and v1 used it inside quoted HTML attributes:

```html
<input class="op" value="${esc(S.operator)}">
<button data-add="${esc(name)}">
```

Setting the operator name to `" autofocus onfocus="alert(1)` closes the `value`
attribute and injects a new event-handler attribute. Confirmed by test — two of six
payloads escaped the attribute.

Impact is limited because the only person who can set that value is the device owner,
and there is no session token or server to steal. It stops being merely self-inflicted
once import exists, since a crafted file could carry the payload.

**Fixed in v2 by:**

- A separate `escAttr()` that also encodes `"` and `'`, used everywhere an attribute is built.
- Set inputs now key off the **exercise index** (`data-x="3"`) rather than the exercise
  name, so a user-controlled string never reaches an attribute selector at all.
- All 42 attribute interpolations audited; the remaining un-escaped ones are numeric
  constants and SVG coordinates computed from `Math`, never user input.

### GT-02 · Unvalidated import — **Medium** — designed out of v2

Import is new in v2, so this was addressed at design time rather than found. Every path
into state (`localStorage` **and** imported files) runs through `cleanState()`, which:

- Clamps every string to a maximum length (names 60, notes 2000, operator 24)
- Coerces every number and rejects out-of-range values (weight 0–2000 kg, reps 0–1000)
- Drops sessions with no valid timestamp or no entries
- Filters muscle tags against the known list, discarding anything invented
- Caps collection sizes (12 days, 40 exercises/day, 20 sets/exercise, 5000 sessions)
- Rejects files over 5 MB and shows a confirmation naming the session count before replacing anything

Verified against a hostile object: a 99 KB note truncated to 2000 chars, `w: 1e9` clamped
to 0, `r: -5` clamped to 0, a fake muscle tag dropped, `null` and string entries in the
array discarded, `draft: "nope"` replaced with an object.

### GT-03 · Reset had no meaningful guard — **Low** — fixed in v2

v1's reset used a single `confirm()`. One mis-tap on a phone and a year of training is gone
with no undo and no backup.

v2 requires typing `DELETE`, states how many sessions will be lost, and tells you to export
first. Anything other than `DELETE` cancels and says so. Session deletion and day deletion
also confirm now, naming what is being removed.

### GT-04 · Data loss through storage clearing — **Medium, unresolved by design**

`localStorage` is wiped by "Clear browsing data", by iOS Safari's 7-day eviction for
unused sites, and by uninstalling the PWA. There is no recovery.

v2 mitigates rather than solves: export/import, a warning in Settings, and a footer line
saying the log is device-only. A real fix means IndexedDB plus an opt-in cloud sync, which
brings a backend, accounts, and privacy obligations. Not worth it yet — but this is the
single thing most likely to make a stranger abandon the app, so it is the top candidate
for v3.

### GT-05 · Third-party font requests — **Low, accepted**

The page loads IBM Plex from `fonts.googleapis.com`, which discloses each visitor's IP
and User-Agent to Google, and means fonts are unavailable on a cold offline load. Kept
for now because self-hosting adds ~200 KB of WOFF2 to the repo. If GymTrack ever ships
to an app store, self-host — App Store privacy labels will require declaring this
otherwise.

### GT-06 · No Content-Security-Policy — **Low** — partly fixed in v2

v1 had none. v2 adds a meta CSP: `default-src 'none'` with narrow allowances, so even a
successful injection cannot exfiltrate anything — no `connect-src` to a third party, no
external script loading, `base-uri 'none'`.

Two caveats. `script-src 'unsafe-inline'` is required because the whole app is one inline
script; removing it means splitting out `app.js`, which is worth doing eventually.
And `frame-ancestors` is ignored in a meta tag — it needs an HTTP header, which GitHub
Pages cannot set. Clickjacking risk here is negligible (no privileged actions), so this is
noted rather than fixed.

---

## Checked and found clean

- No `eval`, `new Function`, `innerHTML +=` on user data, or `document.write`
- No `dangerouslySetInnerHTML` equivalent — all rendering goes through `esc`/`escAttr`
- No `postMessage`, no `window.open`, no user-controlled URLs
- Service worker caches same-origin GETs only; opaque cross-origin responses are never stored
- No secrets, tokens, or API keys in the repo
- `"use strict"` at the top of the module
- No dependencies at all, so no supply chain to compromise

## If this ever gets a backend

Everything above assumes client-only. Adding accounts or sync introduces authentication,
authorisation, transport security, and a database holding health-adjacent data about real
people — a different review entirely, and one worth doing before launch rather than after.

---

## v3 addendum — log types

v3 gives each exercise a `logType` (`weighted` / `bodyweight` / `timed` / `carry`), which
adds two input paths worth noting.

- **Row shape is derived from the type, never from the stored data.** `blankRow()` builds
  a row from the type's field list, and `sanitiseDraft()` re-derives every draft row on
  load. A saved draft carrying fields that no longer belong to the exercise is discarded
  rather than rendered.
- **Legacy reshaping is lossy in one direction only.** `reshape()` moves a number into the
  field its type expects (a side plank's 45 in the reps slot becomes 45 seconds) and drops
  fields the type has no place for. It never invents a value.
- `guessKind()` runs on names only and returns one of four fixed strings — no regex is
  built from user input, so there is no ReDoS or injection path through it.

Log types do not change the escaping model, the CSP, or the storage boundary. Findings
GT-01 through GT-06 are unaffected.
