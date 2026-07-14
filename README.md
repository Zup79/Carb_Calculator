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
- Recheck prompt: countdown banner after logging a low (default 15 min), recheck BG linked to the event, so the record shows low → treatment → outcome — with an audible chime, vibration and a system notification when due
- Low classification: every treatment auto-stamped `confirmed` (BG < 3.9) or `preemptive` — because counting judgement-call top-ups together with true hypoglycaemia distorts everything downstream
- Watch-first reminder: treating flat or rising at 4.4+ shows the educator's guidance (watch, check pump suspension time) alongside the suggestion — the calculation still runs; the reminder is context, not a block
- "Pump suspended" context chip on the treat tab — suspension state at the time of a low is the first thing an educator asks about

**Bolus compare (second tab)**
- Standard carb ÷ ICR + correction − IOB, every term visible
- Log the Omnipod's actual figure alongside — the diary captures pump-calculated insulin against the plain maths
- Meal tag (breakfast / lunch / afternoon / dinner / snack), pre-selected from the clock, one tap to change — the note field then only carries what's different ("orange + usual lunch")
- Bolus timing relative to first bite: before / with food / after, with optional minutes — captures pre-bolus behaviour for review; logging a "before X min" bolus starts an eat-timer countdown with chime and notification when it's time to eat
- Context chips on both tabs (illness / stress / activity / unusual day), multi-select — flow through diary, CSV, and print
- **IOB method** (Settings → Bolus): Method 1 (default) subtracts IOB from carbs + correction combined, so a large IOB can eat into the meal bolus itself. Method 2 subtracts IOB from the correction only, capped at the correction amount — it never spills into the meal bolus, and does nothing at all when BG is at or below target. Matches the two methods offered by Ypsomed's mylife app; the ledger shows which was used and, on Method 2, any IOB left over uncounted.

**Time-block ratios — independent schedules**
- ICR, correction factor (CF/ISF) and target BG each have their **own** schedule of up to 8 time blocks — boundaries don't need to align, matching how pumps program them
- Each block runs from its start to the next block in its schedule, wrapping overnight
- Calculations resolve all three schedules independently at the time of entry; the resolved blocks (e.g. `ICR@06:30 · CF@09:00 · tgt@00:00`) and values are shown in the result ledger and stamped on every log entry and CSV row

**Diary**
- Every low, meal, recheck and note in one timeline
- Free-text notes (+ Note button): capture a trend or situation, with an editable timestamp for retrospective flagging
- 14-day stats: lows count, average treatment size, recheck rate, rebound rate (>10 mmol/L), % of meals pre-bolused, lows by time of day
- CSV export and print/PDF for the educator, ratios and stats stamped in the header

## The maths

**IOB method (bolus tab, not-low branch)**

```
Method 1 (default): dose = max(0, carbBolus + correction − IOB)
Method 2:           iobApplied = correction > 0 ? min(IOB, correction) : 0
                     dose = max(0, carbBolus + (correction − iobApplied))
```

Method 2's IOB never exceeds the correction term and is zero whenever BG is at or below target — so a large IOB can reduce or cancel a correction but can never reduce the meal bolus below the full carb amount. Method 1 has no such floor: excess IOB keeps subtracting until the whole dose, meal bolus included, floors at zero. The low-BG treat-carbs branch is unaffected by this setting either way.

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

## Diary data report

`report.html` is a standalone, durable data report — the diary rendered faithfully and nothing more. Opened from the same site as the app (Diary tab → Educator report), "Load from this device" pulls the diary and ratio schedules straight from the app's local storage — no copy-paste, built for doing the whole thing from a phone. From any other device, paste a Carb_Calc CSV export instead. Print to A4 PDF.

Design rule: **no analysis in code.** The page groups by day, orders by time, and totals by arithmetic (counts and sums only — no thresholds, no pattern detection, no interpretive sentences). It is schema-agnostic: known columns get friendly labels, unknown future columns are rendered under their raw header names, and columns empty across the whole dataset are dropped — so new app fields appear in the report with no code change, and old CSVs still build cleanly.

## Analysis workflow

Pattern-spotting, thresholds and judgment deliberately live **outside** the code, in a live review conversation that adapts to whatever the data shows. `carb_calc_analysis_prompt.md` (in this repo) is the standing prompt: paste it into a Claude Project with the latest CSV export, and it produces findings with row references plus questions for the educator — under strict framing rules (association ≠ cause, small-n humility, no medical advice, nothing pump software already reports).

Pipeline: **app logs → CSV export → data report (durable record, PDF) → analysis chat (interpretation, regenerated fresh each time)**.

## Known limitations

- AID-displayed IOB includes automated delivery. The weighting % is a blunt instrument — start conservative and tune with your educator.
- Decay curves are linear approximations. Real absorption varies by food, activity and everything else.
- The app only knows what you log. Garbage in, garbage out.
- Alarms (recheck and eat timer) fire reliably while the app is open or recently backgrounded — notifications go via the service worker (the constructor form silently fails on iOS and Android Chrome), with an in-app chime and vibration as well. With no push server (by design — nothing leaves the device), a fully closed app cannot be woken; the banner on reopen is the backstop. iOS additionally requires the app installed to the home screen (iOS 16.4+) for notifications at all.

## Settings

| Setting | Unit | Notes |
|---|---|---|
| ICR schedule (up to 8 blocks) | time + g/U | from your care team; independent boundaries |
| CF schedule (up to 8 blocks) | time + mmol/L per U | from your care team; independent boundaries |
| Target schedule (up to 8 blocks) | time + mmol/L | from your care team; independent boundaries; nothing calculates until all three schedules have at least one block |
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
