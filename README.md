# Bolus Compare

A single-file PWA for comparing insulin bolus calculations against your pump. Enter a blood glucose level (mmol/L) and carbohydrates; it shows a suggested dose with every term of the calculation visible. Built because AID systems (Omnipod 5, t:slim Control-IQ) don't expose a plain carb + correction − IOB number to sanity-check against.

> **This is not a medical device.** It delivers nothing and decides nothing. Every output is a comparison figure only. Insulin ratios, targets and dosing decisions belong to you and your diabetes care team. If this app and your pump disagree, trust the pump and your training.

## What it does

- BG + carbs in, suggested dose out — shown as a ledger, each term visible, no black box
- Optional manual IOB entry from the pump display (overrides the app estimate, flagged `*` in the log)
- Carbs-on-board and insulin-on-board decay from your logged entries, stepped every 30 minutes
- Low-BG logic: treatment carbs get no insulin; only excess carbs are dosed
- COB correction damping (optional): halves the correction while carbs are still absorbing — only ever reduces a dose
- Food diary: every entry logged with note, exportable as CSV or print/PDF for your diabetes educator
- Fully offline once installed; all data stays in localStorage on the device

## The maths

Standard bolus calculation, nothing novel:

**Above target**

```
carb bolus  = carbs ÷ ICR
correction  = (BG − target) ÷ ISF        [× 0.5 if damping on and COB > 0.5 g]
suggested   = max(0, carb bolus + correction − IOB)
```

**Below target**

```
treatment carbs = (target − BG) × ICR ÷ ISF     ← no insulin on these
excess carbs    = max(0, carbs − treatment carbs)
suggested       = max(0, excess ÷ ICR − IOB)
```

**Decay (COB and IOB from logged entries)**

Linear, stepped at 30-minute intervals:

```
remaining = amount × max(0, 1 − floor(elapsed_min / 30) × 30 ÷ (duration_hr × 60))
```

Defaults are placeholders only — carb absorption 3 h, insulin duration 4 h — adjustable in Settings. Treatment carbs from low-BG entries are excluded from COB.

**Rails**

- Suggested dose floors at zero — never negative
- Damping only ever reduces, never adds
- Optional max-dose display cap
- Dose rounded to 0.05 U
- Nothing calculates until ICR, ISF and target are entered

## Known limitations

- On an AID system, the pump's auto-corrections are invisible to this app. The app's own IOB estimate only knows what you log — it **will** drift from the pump. Use the manual IOB field (read it off the pump screen) for anything that matters.
- Decay curves are linear approximations. Real absorption varies by food, activity, stress and everything else.
- The app trusts whatever "Delivered" value you log. Garbage in, garbage out.

## Settings

| Setting | Unit | Notes |
|---|---|---|
| ICR | g per U | from your care team |
| ISF | mmol/L per U | from your care team |
| Target BG | mmol/L | from your care team |
| Max shown dose | U | display cap, optional |
| Carb absorption | hours | decay duration for COB |
| Insulin duration | hours | decay duration for IOB |
| COB damping | on/off | halves correction while carbs absorbing |

All settings stored in localStorage. Blank on first run — no baked-in numbers.

## Deploy

GitHub Pages, no build step:

1. Put `index.html`, `sw.js`, `manifest.json` in the repo root
2. Settings → Pages → deploy from `main` branch, root
3. Open the URL on your phone → Add to Home Screen

Service worker is network-first with cache fallback: updates land when online, app still opens offline.

## Stack

Vanilla JS/CSS, single-file `index.html`, localStorage, service worker. No frameworks, no analytics, no network calls, no data leaves the device.

## Licence

Personal use. No warranty of any kind — see the disclaimer at the top, then read it again.
