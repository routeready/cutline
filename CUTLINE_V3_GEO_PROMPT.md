# CUTLINE V3 — GEO CAPTURE & ROUTE INTELLIGENCE

**Execution prompt for Claude Code. Repo: Cutline (Gateway Lawn & Landscape field ops platform).**

---

## 0. Read this first

You are working on a **live production app with real customer data in it.** Roughly 19 residential
accounts, active billing records, active schedule. There is no staging environment. Data loss is the
worst possible outcome of this session — worse than shipping nothing.

Before you write a single line of code:

1. Read the entire single-file app and build a mental model of the existing state shape, the
   localStorage keys, the migration mechanism, the timer implementation, and the Today tab render path.
2. Report back what you found — specifically the current schema of a property record and a cut record,
   and how the existing migration stamp works. **Wait for confirmation before proceeding.**

Do not guess at names. Everything below describes *intent*. Match the repo's actual conventions for
casing, ID format, and record structure. If my field names conflict with existing ones, the repo wins.

---

## 1. Hard constraints (non-negotiable)

- **Single file.** One HTML file, inline CSS and JS. No build step, no bundler, no npm, no modules.
- **Vanilla JS only.** No framework. Third-party code arrives by CDN `<script>` tag or not at all.
- **localStorage persistence.** Existing pattern. Don't introduce IndexedDB.
- **Existing design system wins.** OKLCH tokens, `light-dark()` theming, native `<dialog>` modals,
  View Transitions. Derive every colour from the existing token set. Do not introduce a new palette,
  a new typeface, or a new modal pattern.
- **Offline-tolerant.** This runs in a truck with patchy LTE. Nothing may block, hang, or throw when
  the network is unavailable.
- **Field-usable.** One hand, phone, direct sunlight, sometimes gloves. 44px minimum tap targets.
- Deployed to `cutline.mineready.io` via Cloudflare tunnel on the Mac Mini. HTTPS is available,
  which matters — see §4.

---

## 2. Why we're building this (context for your judgment calls)

The owner cuts 15–19 residential lawns part-time around a day job. He wants, eventually, to see on a
map which lawn is worth driving to next — expressed as **dollars per hour including the drive.**

The app currently knows what each lawn pays and how long each cut took. It does **not** know where any
property is, or how long the drive between any two properties takes.

Rather than geocode addresses (unreliable for rural East Ferris and Callander) or buy a routing API
(licence terms forbid deriving metrics from the results and rendering them on a non-vendor map), the
app will **learn both facts from ordinary use**: the phone already knows where it is when the timer
starts, and it knows what time it is when the timer stops.

This means **the capture layer is the whole project and everything else is downstream of it.** Every
cut performed before capture ships is a drive time that can never be recovered. That's why this
session ships capture and nothing else.

---

## 3. Session scope

**In scope: Phase 0 and Phase 1 only.** Then stop.

Phases 2 and 3 are specified in §7 for a later session so you understand where this is heading and
don't design yourself into a corner. **Do not build them now.** Phase 1 needs to run in the field for
two to three weeks before the numbers in Phase 3 mean anything, so building the display layer now
produces a screen full of guesses that will teach the owner to distrust it.

---

## 4. PHASE 0 — Schema migration

One migration, adding everything at once, so this never has to happen twice.

### Safety, in this order

1. **Before migrating, write a full JSON export of current state to a file download** and make the
   user confirm they have it. Not a console log — an actual downloaded file they can put somewhere.
2. Migration must be **forward-only and idempotent.** Running it twice changes nothing.
3. **Never delete or rename an existing key or field.** Additive only.
4. If the migration throws for any reason, restore prior state and surface a clear error. Never leave
   state half-migrated.

### Fields to add

On each **property** record:

| Purpose | Notes |
|---|---|
| latitude, longitude | Nullable. Populated by capture or manual placement. |
| coordinate source | `"gps"` or `"manual"`. Lets us trust GPS fixes over taps. |
| coordinate accuracy | Metres, from the GPS fix. Needed to decide whether to overwrite. |
| last aerated | Date, nullable. Unrelated feature, free to add while we're in here. |
| aeration eligible | Boolean/null — result of the screwdriver compaction test. |

On each **cut** record: ensure the start coordinate is captured alongside the existing duration. If
cut records already store a start timestamp, good — we need it.

**New top-level collection: drive legs.** Append-only. Each record:

- from property ID
- to property ID
- elapsed seconds
- timestamp of observation

Store **raw observations, one row per drive.** Do not aggregate on write. Medians get computed at read
time. This keeps the data auditable and lets the algorithm change later without losing history.

Volume is trivial — expect a few hundred rows a season. No pruning needed, but log total serialized
state size to console after migration so we can watch it.

---

## 5. PHASE 1 — Capture

**This phase adds almost no visible UI.** It quietly fills a notebook. Resist the urge to show it off.

### On timer START

Fire `navigator.geolocation.getCurrentPosition()`. Timer start is a user gesture, which is what iOS
Safari requires to raise the permission prompt — so this is the correct hook.

- **Never block the timer on the geolocation call.** Start the timer immediately, resolve the fix
  asynchronously. If the fix never arrives, the cut is still recorded normally.
- Options: high accuracy on, ~15s timeout, don't accept a cached fix older than about a minute.
- On success: if the property has no coordinates, write them plus source `"gps"` plus accuracy.
- **Reject fixes with accuracy worse than 50m.** A 2km-wide fix will put the pin in the lake.
- **If coordinates already exist, only overwrite when the new fix is meaningfully more accurate**
  than the stored one, and only when the stored source is `"gps"`. A manual placement is a
  deliberate human decision — never let a GPS fix silently overwrite it.

### On timer STOP

Capture position and timestamp again, then decide whether a drive leg can be inferred.

Append a leg **only if all of these hold:**

- A previous stop was completed in this working session
- Both properties have known coordinates
- Elapsed time between the previous stop and this start is **between 30 seconds and 90 minutes**

The 90-minute ceiling matters: anything longer means he went home, ate dinner, or got a phone call,
and recording it as drive time would poison the median. Discard silently, don't warn.

### Permission and failure handling

The app must be **fully functional with location permission denied.** Handle all three error cases
(`PERMISSION_DENIED`, `POSITION_UNAVAILABLE`, `TIMEOUT`) without a thrown exception and without a
modal. Degrade to exactly the app's current behaviour.

Do not ask for permission on page load. Do not re-prompt after a denial. If permission is denied,
surface it once, quietly, in Settings — not on Today.

### The only visible UI in this phase

A small, unobtrusive indicator on the Today tab showing coverage — how many properties have
coordinates yet, e.g. `11 of 19 mapped`. That's it. It tells him the notebook is filling without
pretending the feature is finished.

**Copy discipline:** name things by what he controls, not how the system works. "11 of 19 mapped,"
not "geolocation cache 58% populated." Active voice. Sentence case. If location is off, say what to
do about it, don't apologize.

---

## 6. Acceptance criteria for this session

Verify each of these and report results. Where you can't verify in a headless environment, say so
explicitly rather than assuming it passes.

1. Fresh install with no prior data: app loads, migration runs clean, nothing throws.
2. Existing production data: after migration, all 19 properties intact, all cut history intact, all
   billing records intact, new fields present and null.
3. Migration run twice: no duplicate fields, no data change, no error.
4. Location permission denied: timer starts, cut records normally, no modal, no console error.
5. Location unavailable / airplane mode: same.
6. Two consecutive simulated cuts with plausible coordinates: exactly one leg appended, correct
   direction, plausible elapsed seconds.
7. Two cuts 3 hours apart: **no** leg appended.
8. Low-accuracy fix (accuracy > 50m): coordinates **not** written.
9. Manually placed coordinates followed by a GPS fix: manual placement **survives.**
10. Total serialized state size logged and reported.

---

## 7. DEFERRED — do not build this session

Included so your Phase 0/1 decisions don't box these in.

### Phase 2 — Map on the Today tab

Leaflet via CDN with OpenStreetMap tiles and required attribution. Chosen over vendor SDKs because
their terms prohibit using vendor drive times alongside a non-vendor map, and prohibit deriving new
metrics from their content — which is exactly what the $/hr figure does.

- Markers numbered in scheduled route order. Numbering is justified here because route order is a
  real sequence carrying information the reader needs — not decoration.
- Marker fill: **three or four discrete bands** by $/hr, thresholds user-configurable in Settings.
  Discrete, not a gradient — this gets read at arm's length in sunlight. Derive band colours from
  existing OKLCH tokens; perceptually uniform lightness means the ramp reads in the correct order.
- Tap a marker → detail sheet using the **existing** native `<dialog>` pattern. Shows price, median
  on-site time, current $/hr, and a **Navigate** button.
- Navigate deep-links to the native map app (`https://maps.apple.com/?daddr=<lat>,<lng>`). Deep
  linking is free and unrestricted — the licence problems only arise from consuming vendor data
  programmatically, not from handing a destination off.
- Long-press or explicit "place pin" mode to set coordinates manually for unvisited properties.
  Writes source `"manual"`.
- **Tile load failure must not break the Today tab.** Map is additive; the list is the source of truth.

### Phase 3 — The metrics

Three read-time functions:

- **Median on-site time** for a property over its last ~5 cuts. Return the value *and the sample
  count* — the count drives display confidence.
- **Drive estimate** from a coordinate to a property: use the observed median from the legs
  collection when there are 2+ observations; otherwise fall back to straight-line distance × 1.4.
  **Return which source was used.**
- **Next-stop rate**: `price ÷ (drive seconds + on-site seconds)`, normalized to an hourly figure.

Display rules:

- Sort remaining Today cards by next-stop rate, descending.
- Recompute **on timer stop only.** Not on an interval, not on position change — battery.
- While on-site sample count < 3, show a **range**, not a hard number.
- Mark estimated (non-measured) drive times visibly so he knows which figures to trust.
- One passive, uncoloured line at the bottom of the list: drive time home from current position.
  Informational only — it must not reorder anything or nag.

---

## 8. Explicit non-goals

Do not build, suggest, or scaffold any of these. They have been considered and deliberately rejected.

- ❌ Route optimization of any kind. No nearest-neighbour, no 2-opt, no TSP solver, no "suggested
  order." The owner has explicitly declined this. The app informs; he decides.
- ❌ Google Maps, Apple MapKit, Mapbox, or any keyed routing/tile API.
- ❌ Address geocoding.
- ❌ Any API key anywhere in the client file.
- ❌ Continuous or background location tracking. Two fixes per cut, on explicit user action.
- ❌ Automatic dropping, flagging, or hiding of low-$/hr properties.
- ❌ Build tooling, framework migration, or file splitting.

---

## 9. Things that will bite you

- **Geolocation needs a secure context.** Fine on `cutline.mineready.io`, silently broken on
  `file://` or plain-HTTP localhost. Test accordingly.
- **iOS Safari geolocation is slower and less accurate than you expect** on first call after a cold
  start. The 15s timeout is deliberate. Don't tighten it.
- **A driveway fix and a house fix differ by 20–30m.** That's fine for a route map and useless for
  anything finer. Don't build anything that assumes precision better than ~30m.
- **Elapsed time between cuts is not drive time.** It includes gate opening, trailer unloading, and
  answering the phone. That's actually desirable — it's the real cost of the stop — but never label
  it "drive time" in the UI. Call it what it is: time between stops.
- **The 90-minute leg ceiling is a heuristic, not a truth.** Make it a named constant, not a magic
  number buried in a conditional.
- **Don't aggregate on write.** Raw legs only. The median algorithm will change.

---

## 10. Working style for this session

- Report the schema audit from §0 and **stop for confirmation** before writing code.
- Phase 0 and Phase 1 as **separate commits**, each independently verifiable.
- After Phase 1, stop. Report acceptance criteria results. Do not proceed to Phase 2.
- If any instruction here conflicts with something you find in the repo, **raise the conflict rather
  than resolving it silently.**
- If you find yourself needing to rename or delete an existing field to make something work, stop and
  ask instead.
