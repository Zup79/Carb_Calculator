# Carb_Calc

A single-file PWA that answers the question AID systems don't: **how many grams of carbs should I eat for this low?** Enter BG (mmol/L), IOB from the pump, and CGM trend; it suggests grams with every term of the calculation visible — and logs the treatment, because the pump records every bolus but hypo treatments vanish from the record entirely.

Also includes a bolus comparison tab (carb + correction − IOB, shown as a ledger) and a diary with 14-day stats for your diabetes educator.

> **This is not a medical device.** It delivers nothing and decides nothing. Every output is a comparison or record only. Ratios, targets and treatment decisions belong to you and your diabetes care team. If this app and your training disagree, trust your training.

## What it does

**Treat low (primary)**
- BG + pump IOB + CGM trend in, suggested grams out — ledger format, no black box
- IOB weighting: only a settings-defined % of displayed IOB is counted, because AID-displayed IOB includes automated delivery and counting it in full over-treats
- Trend scaling: falling fast needs more than drifting low (↓↓ ×1.5, ↓ ×1.25, → ×1.0, ↑ ×0.75 — all adjustable)
- Previous treatments still absorbing are subtracted — no stacking jelly beans
- Recheck prompt: countdown banner after logging a low (default 15 min), recheck BG linked to the event, so the record shows low → treatment → outcome

**Bolus compare (second tab)**
- Standard carb ÷ ICR + correction − IOB, every term visible
- Log the Omnipod's actual figure alongside — the diary captures pump-calculated insulin against the plain maths

**Diary**
- Every low, meal and recheck in one timeline
- 14-day stats: lows count, average treatment size, recheck rate, rebound rate (>10 mmol/L), lows by time of day
- CSV export and print/PDF for the educator, ratios and stats stamped in the header

## The maths

**Low treatment (grams)**

```
lift        = (target − BG) × ICR ÷ ISF        [0 if BG ≥ target]
trended     = lift × trend multiplier
IOB cover   = IOB × ICR × weighting%
grams       = trended + IOB cover − previous treatment still absorbing
```

Clamped to min/max treatment settings; if ≤ 0, suggestion is 0 g with a "recheck rather than eat" message. BG at or above target with active insulin gives a pre-emptive figure, clearly labelled.

**Bolus comparison (units)** — standard, nothing novel:

```
Above target:  max(0, carbs ÷ ICR + (BG − target) ÷ ISF − IOB)
Below target:  treatment carbs = (target − BG) × ICR ÷ ISF get no insulin;
               only excess carbs are dosed
```

Optional damping halves the correction while meal carbs are absorbing — only ever reduces.

**Decay** — linear, stepped every 30 minutes:

```
remaining = amount × max(0, 1 − floor(elapsed_min / 30) × 30 ÷ (duration_hr × 60))
```

Meal carbs and low-treatment carbs are tracked separately: meal carbs count as covered by their insulin, so only previous *treatment* carbs offset a new low suggestion, and only *meal* carbs drive bolus-side damping.

## Known limitations

- AID-displayed IOB includes automated delivery. The weighting % is a blunt instrument — start conservative and tune with your educator.
- Decay curves are linear approximations. Real absorption varies by food, activity and everything else.
- The app only knows what you log. Garbage in, garbage out.
- Recheck notifications fire only while the app is open; the in-app banner is the reliable prompt.

## Settings

| Setting | Unit | Notes |
|---|---|---|
| ICR / ISF / Target BG | g/U · mmol/L/U · mmol/L | from your care team; nothing calculates until set |
| IOB weighting | % | share of pump IOB counted toward treatment |
| Min / max treatment | g | floor and cap on suggestions |
| Recheck timer | min | countdown after logging a low |
| Trend multipliers | × | ↓↓ / ↓ / → / ↑ |
| Carb absorption / insulin duration | hours | decay durations |
| COB damping | on/off | bolus tab only; reduces only |

All blank on first run, stored in localStorage, nothing leaves the device.

## Deploy

GitHub Pages, no build step: put `index.html`, `sw.js`, `manifest.json` and the icon PNGs in the repo root, enable Pages from `main`, open on your phone, Add to Home Screen. Service worker is network-first with cache fallback — updates land when online, app still opens offline.

## Icons

Lowercase `cc` set like a unit of measure (as in mL/cc), with graduation ticks like a measuring device and a small `g` for what it counts. Files: `icon-192.png`, `icon-512.png`, `icon-512-maskable.png` (Android adaptive), `apple-touch-icon.png` (iOS home screen), `favicon-32.png`.

## Stack

Vanilla JS/CSS, single-file `index.html`, localStorage, service worker. No frameworks, no analytics, no network calls.

## Licence

Personal use. No warranty of any kind — see the disclaimer at the top, then read it again.
