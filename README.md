# Plays Like — 2K25 Yardage Book

Single-page PWA. Vanilla HTML/CSS/JS in one file, no dependencies, no build step.
Deployed via GitHub Pages at **daleyaaron93.github.io/GOLF-CALC/**.

```
index.html            app — markup, tokens, model tables, compute/paint (single file)
manifest.json         PWA manifest
apple-touch-icon.png  180px, iOS home screen
icon-192.png          PWA icon
icon-512.png          PWA icon
README.md             this file
```

Serve over http, not `file://`, or the manifest won't register.

---

## Working rules — read before changing anything

These are Mr. Daley's, and they are the point of the project:

1. **Don't assume or guess. Stop and ask.**
2. **Domain calls about the game belong to Mr. Daley.** How the game behaves, which
   shot the auto-picker should choose, what a number should be — his call, not yours.
3. **Anything unverified stays visibly flagged in the UI.** Never let an estimate
   render as if it were measured. This has broken twice; see "Provenance" below.

---

## Fixed in v10 — the `autoShot` crash

`autoShot()` used to return `g("knock")` for wind ≥ 12 mph at 100+ yards. No shot has
that key — it is `lknock` — so `g()` returned `undefined` and `compute()` threw on
`shot.windMult`, taking down any windy hole over 100 yards on Auto.

Mr. Daley chose **Long knockdown** (`lknock`): wind multiplier 0.60, irons, off turf and
rough. That was his call to make, which is why it sat open rather than being patched on
sight. Verified in-browser: `S.dist=150; S.wind=14; S.from='turf'` now returns
`lknock / Long knockdown` and computes without throwing.

---

## Club selection — three rules that must hold

**1. LEGAL is never relaxed.** `attempt()` builds `legal` (right club class; Driver only off a
tee) and then narrows it to `usable` (has a measured window for this shot). When `usable`
comes back empty it falls back to **`legal`**, on estimated windows, which the card flags.

It used to fall back to `BAG.slice()` — the whole bag — which discarded the class rule and the
Driver rule together. Every bunker shot has an unmeasured window on every wedge, so every
bunker shot emptied `usable` and got handed the full bag: **a Driver out of the sand at 150%
power with "Out of range — nothing reaches" printed under it.** Never widen a fallback past
the rule it is meant to respect.

**2. Auto walks a ladder; an explicit shot never gets substituted.** `autoShot()`'s distance
bands describe the shot you would *want*, and say nothing about whether any club can dial the
number. Pitch's band ran to 100 yd while Pitch tops out at **58 yd** with every wedge in the
bag, so 65–95 yd on Auto reported "nothing reaches" with an Approach sitting right there.
`autoLadder()` now returns that first choice followed by fallbacks, and `compute()` takes the
first one some club actually fits. Order it by **which shot you would reach for**, not by reach
ceiling — ceiling order put Long knockdown ahead of Approach and returned a 7 Iron at 46% power
for a 60 yd shot. If the shot was chosen by hand, it is honoured and the warning stands.

**3. A remaining data gap, not a bug: 59–66 yd.** Pitch tops out at 58 and the shortest
measured Approach (60°) has a *minimum* of 67, so nothing natural covers the gap and Auto falls
to a Long knockdown. Measuring a wedge Runner or a longer Pitch window would close it.

---

## Data provenance — do not overwrite these

| Table | Status |
| --- | --- |
| `WINDOWS` | **Measured off the game screen. Authoritative.** Except the two interpolated pairs listed in `INTERP`. |
| `BAG` max/min/total | **Measured.** The box reading beats the bag screen where they differ by a yard: 9i 149 not 150, 8i 159 not 160, 7i 170, 6i 181, 60° 95 not 97. |
| Wind / elevation rates | From Operation Sports testing. Tagged `SRC` in the UI. |
| Green-roll factors | **Only 5 Iron (0.60) and 3 Wood (1.45) are real.** The rest are interpolated. |
| Green speed 0.80 / 1.00 / 1.25 | Estimate. |
| Shot roll values | Flop 1 and Spinner 5 are Mr. Daley's (`YOURS`). The rest are estimates (`GUESS`). |

### How flagging works

`windowFor(club, shot)` returns `measured:true` **only** when both ends of a window are
positive *and* the pair is not listed in `INTERP`. Anything else returns `measured:false`,
which fires the "Estimated window" warning on the result card.

Two guards, both added after real failures:

- **Zero-end guard.** A present-but-zero minimum used to fall through to `max*0.6` while
  still returning `measured:true`, so 56° Long Flop showed a fabricated 23-yard minimum
  with no warning. Both ends are now required to be positive, and the max is checked for
  the same reason.
- **`INTERP` table.** Interpolated values are complete on both ends, so the guard alone
  cannot tell them from measured data. `INTERP` names the club/shot pairs that were
  derived rather than read off the screen, and forces `measured:false`.

```js
INTERP = { "56°": { lflop:1, flop:1 } }
```

**Measuring a pair takes two steps:** put the real numbers in `WINDOWS` **and** delete the
key from `INTERP`. Miss the second and it keeps reporting estimated — annoying, but it
fails in the safe direction.

The interpolated entries and their derivation:

```
56° lflop:[39,24]   max 39 MEASURED; min 24 interpolated from 50°=26, 60°=22
56° flop:[28,16]    both interpolated from 50° flop 31/18 and 60° flop 26/15
```

Method validation: the same interpolation predicts the 56° Long Flop **max at 38.9
against a measured 39** — accurate to well under a yard on this club, which is why these
ship flagged rather than omitted.

### UI tags

| Tag | Meaning | Treatment |
| --- | --- | --- |
| `MEAS` | Read off the game screen | Quiet green outline |
| `YOURS` | Mr. Daley's own measured value | Quiet green outline (`tag ok`) |
| `GUESS` | Estimate — correct these first | **Terracotta outline + tint, deliberately the loudest thing in the drawer** |

Do not flatten `GUESS` into the calm palette. Its loudness is the point.

---

## Game mechanics the model assumes

- The number you dial in-game is **carry**. Club max does not shrink on a bad lie.
- The lie indicator is a **range** (e.g. 87–95%) — a distance multiplier sampled randomly
  within it. Range width comes from club fittings, not the lie.
- Every club × shot-type pair has its own **dial window**. These are `WINDOWS`.
- Rollout depends on **club trajectory**, not just landing surface.

---

## Architecture

Single file, no framework, no build step.

**Tables:** `BAG`, `SHOTS`, `WINDOWS`, `INTERP`, `SURFACES`, `LANDING`, `WINDS`, `GREENSPEED`.
**State:** `S` — the shot (distance, wind, lie, surface, shot type…).

**Flow:** two views, `vEntry` and `vResult`, toggled by the `.view.on` class.
"Give me the shot" runs `compute()`, which picks the shot (`autoShot()` when on Auto),
filters `BAG` to usable clubs, then `solve()` binary-searches the dial that stops on
target, using `fly()` to model carry → roll → stop across the lie range. Output is painted
into `.screen`.

## What a user can edit, and what they can't

Every drawer is user-editable and persists (see "Persistence"). Each multi-column drawer
carries a `.row.hdr` header row built by its `build*()` function, with header cells fixed at
the same 64px as the inputs so the two line up; `.lbl` flexes and absorbs the MEAS/GUESS tag,
whose width differs between the two. Each drawer also carries a `.gloss` block defining every
column in plain language. **Keep both in step with the fields** — unlabelled numeric columns
are what this section exists to prevent.

| Surface | Editable | Where |
| --- | --- | --- |
| Club distances — min carry, max carry, max total, green roll | yes | My bag |
| Shot behaviour — reach, wind, fixed roll | yes | Shot types |
| Landing surfaces — roll vs fairway | yes | Landing surface |
| The eight wind/elevation rates | yes | Wind & elevation |
| `WINDOWS` — the per-club/shot dial windows | yes | My windows |
| `INTERP`, `SURFACES`, `WINDS`, `GREENSPEED` | **no** | reference data, by design |
| The club lineup itself (13 clubs, fixed) | **no** | no add/remove UI |

### My windows

A window is **read, never inferred** — it is the highest and lowest number the power dial
offers for a club-and-shot pairing, sitting on screen. No quantity of logged shots can produce
one: where a ball finished says nothing about what the box said. Hence data entry, not learning.

Pick a club, get its legal shots grouped by family, type the two numbers. "Only the gaps"
filters to what still needs measuring. Overrides live in `WIN_OVERRIDE` and persist inside the
same calibration key as everything else, so "reset to defaults" restores the shipped tables
whole.

**Typing a pair deletes its `INTERP` flag automatically.** The table used to carry a comment
telling whoever measured a pair to remember to delete that key by hand, or the pair would keep
reporting as estimated. A rule stated only in a comment is dissolved by the first edit that
forgets it, so it is now a mechanism. Clearing a field restores the shipped pair *and* its
`INTERP` flag.

**A missing key means two different things** — "the game does not offer this pairing" and
"nobody has measured it yet" — and the table cannot tell them apart. That is why `attempt()`
widens to `legal` when nothing measured fits. Without the widening, filling in one club's
window drops every other club out of the running: while all three wedges lacked a Splash
window the list emptied and the earlier fallback caught it, but the moment one was filled
`usable` became that single club. **Filling in real data made the answer worse.** Do not
remove the widening step.

### Known dead data: `lflop`

`WINDOWS` carries `lflop` pairs for PW, 50°, 56° and 60°, and `INTERP` flags `56°.lflop` — but
**there is no shot with key `lflop` in `SHOTS`**, so none of it is reachable, and the long
comment in the table describing how that 56° pair was interpolated protects nothing. Either
add a Long flop shot or drop the data; it is a game-domain call, not a code one.

---

**Lie presets.** `buildSurf()` lights the preset whose `lo`/`hi` match the numbers in the
fields *and* whose `from` matches `S.from`. It previously matched on `from` alone, which lit
every preset sharing the bucket — Fairway and Fringe are both `turf`, and Lt rough, Hvy
rough and Mulch are all `rough`, so a single tap lit two or three chips. The seven lo/hi
pairs are unique, so this lights exactly one; typing a lie by hand lights none, which is
correct, since a hand-typed lie is not one of the presets.

**Calibration:** six `<details>` drawers on the entry view — bag, shot types, landing
surface, wind/elevation rates, plus the shot log and suggestions added in v10. The first
four edit the live tables in place.

**Shot conditions still reset on reload.** Only calibration and the log persist; the
distance, wind and lie you type for a given shot are deliberately not remembered.

> ⚠️ If you were handed a `build/` folder with its own README, **it describes a rewrite
> that is not what ships here.** That build has one sticky screen, steppers on every
> number, and a single tabbed calibration drawer. **None of that is in this repo** — the
> shipped app has two views (`vEntry` / `vResult`) and separate drawers. Only its design
> system was adopted. It also claims `localStorage` under `playslike.v2`; the real keys
> are listed under "Persistence" below and are not the same thing. Trust this README and
> the code, not that one.

---

## Persistence (v10)

Three `localStorage` keys, per device:

| Key | Holds |
| --- | --- |
| `playslike.calib.v1` | bag rows, shot-type rows, landing multipliers, the eight wind/elevation rates, and the `learned` registry |
| `playslike.log.v1` | the shot log, an array of entries |
| `playslike.sugg.v1` | dismissed suggestions, keyed by suggestion id |

**The baked-in tables stay the defaults.** `captureDefaults()` snapshots them — and the
rate fields' `defaultValue`, i.e. the `value=""` in the markup — *before* `applyCalib()`
lays any saved values over them. Order matters at boot: snapshot, then apply, then build.
Get it backwards and "reset to defaults" restores whatever was last saved.

Saved values are keyed **by club name and shot key, never by array index**, so reordering
`BAG` or `SHOTS` cannot shift a saved number onto the wrong club.

**`WINDOWS` and `INTERP` are deliberately not persisted.** They are reference data —
measured off the game screen, or interpolated and flagged — not preferences. A saved copy
would also silently outrank a corrected table shipped in a later version.

**Reset** is a two-tap button (no `confirm()`, which would block the page on a phone). It
restores every drawer value, clears the `learned` registry and its tags, and deletes the
calibration key. **It does not touch the shot log** — that is data, not a setting.

Every read and write is wrapped in `try`/`catch`: private windows and blocked site data
throw on access rather than returning empty. If storage is unavailable the app runs on
shipped defaults and says so in a note under the drawers.

---

## Shot log (v10)

"Log this shot" sits under the result plate. Off by default — it is pressed only when the
shot was worth counting: the ball went where it was aimed, and whatever error is left
belongs to the model rather than to a slope or a green running away. **That judgment is
the filter.** There is deliberately no slope tagging and no auto-detection.

Two numbers are entered, **where it landed** and **where it stopped**, because they
separate the two error types the learning pass keeps apart:

- **carry error** (landed short or long) → points at the wind or elevation rates
- **roll error** (landed right, finished wrong) → points at the club's green-roll factor

Everything else is captured automatically from the prescribed shot: club, shot type,
target, dial, power, elevation, wind speed and direction, lie range, landing surface,
green speed, and the predicted lands/rolls/stops with their lie ranges. Thirty fields per
entry.

**Nothing lateral is recorded.** People judge distance far better than direction, so a
left/right figure would be the least reliable number in the file. The predicted aim *is*
stored as context and is never read by the learning pass.

`fly()` records which rate actually fired on the shot (`rateId`, `windYd`, `elevYd`, `gr`)
and `compute()` stashes the lot in `LAST`. The log reads those off the result rather than
re-deriving them — re-deriving is how a learning pass ends up correcting a rate that never
ran on that shot.

**What you actually hit.** Two pickers on the log panel default to the prescribed club and
shot and turn brass the moment they differ. An overruled shot is logged and exported in full,
marked *your pick* in the list — but `followedPlan()` keeps it out of the learning pass, because
the prediction stored on the entry was worked out for the prescribed club and dial. Scoring a
5 Iron result against a Driver prediction measures nothing. Entries written before the pickers
existed have no `actClub` and are treated as prescribed.

**Export** writes a JSON file (`playslike-log-YYYY-MM-DD.json`) so a season is not one
settings reset from gone. Individual entries can be deleted from the drawer, which is the
only way to undo a mistyped yardage.

---

## Learning (v10) — proposes, never applies

Nothing in this pass writes a calibration value. A number changes only when a person
presses **Accept**, and it stays tagged `LEARNED` afterwards so it never reads as
measured. An auto-tuning model that quietly rewrites its own constants is the same failure
as an estimate presented as measured data, one step further removed.

**Threshold: 5 clean shots** (`SUGG_MIN`), Mr. Daley's call. Corrections under 8%
(`SUGG_MIN_CHANGE`) are treated as noise and stay quiet.

Four families, kept strictly separate:

| Family | Reads | Proposes |
| --- | --- | --- |
| Green roll | shots by that club onto green or fringe | that club's `gr` |
| Headwind / tailwind | shots where that rate fired **and the ground was flat** | `cHeadFull`, `cHeadApp`, `cTail` |
| Elevation | shots with elevation **and no wind** | `cElev` |
| Baseline | shots with neither wind nor elevation | *nothing — note only* |

**The clean-bucket rule is the important one.** A shot with both wind and elevation cannot
say which of the two was wrong, so it teaches neither. Splitting the blame between them
would be invented precision.

**The baseline family has no Accept button by design.** If carry is systematically off
with nothing to blame, the bag numbers are the suspect — and those come off the game's
bag screen. The model does not get to overwrite measured data on its own say-so.

**Accepting settles it.** Each row must be logged *after* the last acceptance for that key
(`since()`). Without that watermark the same five shots keep re-proving the same
correction and the factor compounds on every Accept — 0.52 → 0.77 → 1.14 and on, off no
new evidence at all. Dismissing works the same way: gone until the sample has grown by
another full `SUGG_MIN`.

Rows are scored against the rate that actually fired on them, not against today's value,
so changing a rate midway through a season does not corrupt the earlier shots.

**One known limitation.** Roll suggestions are noisiest where roll is smallest — a ±1 yd
reading error on a 2.5 yd predicted roll is a 40% swing. Aggregating five shots by summed
totals (not mean-of-ratios) cancels most of it, since real entry errors scatter both ways,
but treat a green-roll proposal off short irons with more suspicion than one off a wood.

---

## Design system — "Clubhouse"

Linen paper, course green, brass. All tokens are CSS custom properties in `:root` at the
top of `index.html`. Change them there, nowhere else.

| Token | Value | Used for |
| --- | --- | --- |
| `--paper` | `#e7e1cf` | page behind the card |
| `--card` | `#f7f4ea` | the card |
| `--green` | `#1e4032` | result plate, active chips |
| `--brass` | `#9a7b3f` | section labels |
| `--brass-br` | `#e8c98a` | dial figure, power bar |
| `--rule` | `#ddd6c1` | hairline dividers |
| `--field-edge` | `#ccc3a9` | stepper and chip outlines |
| `--warn-edge` | `#b5754a` | GUESS tags, warnings |

**Type — two families, no exceptions.** Cormorant Garamond for every figure, the club
name, and italic asides. Jost for labels — always uppercase, 9px, `letter-spacing:.28em`.

**Rules, not cards.** Sections divide with 1px lines. Radius 2–3px everywhere. One
pronounced shadow, under the green plate. No gradients, no glow.

**Hit targets.** Steppers 46–56px, chips 38px, compass cells 56px.

**Masthead.** Centred stack: `ESTILL COUNTY` eyebrow in brass, `Plays Like` at 44px in
green serif, then a hairline-flanked line. That bottom line is also the view indicator —
`YARDAGE LAB · 2K25` on setup, `YARDAGE LAB · RESULT` on the result view (`#verTag`).
The flanking rules are `::before`/`::after` on `.mh-rule` and shrink before the label wraps.

**Section rules.** One hairline between each top-level section of the entry view, via
`#vEntry > .field, #vEntry > .pair`. Direct children only — the two fields inside `.pair`
(elevation and must-carry) are one section and must not be split by a rule.

**Compass arrows are drawn, not typed.** Each cell renders one inline SVG arrow rotated by
the `deg` field on `WINDS` (0 = up = tail, clockwise). They used to be characters, but the
four diagonals — `U+2196`–`U+2199`, i.e. exactly the four quartering cells — carry emoji
presentation on iOS and rendered as blue glyph tiles beside the plain text arrows on
head/tail/cross. `currentColor` means the arrow flips to card cream on the active cell for
free. The `→` inside the *labels* `CROSS L→R` / `CROSS R→L` is still a character, and is
fine: `U+2192` defaults to text presentation.

The icon is a deep green field with brass pin and pennant, cream ball, double hairline
ring. Artwork bleeds to the full edge of the square so iOS's own corner mask does all the
rounding — do not bake rounded corners back into the PNGs.

---

## Still unmeasured

- **Driver windows not collected.** Driver is absent from `WINDOWS` entirely, so it falls
  through to the ratio estimate and is flagged. Adding a `WINDOWS["Driver"]` entry with a
  missing shot key would *exclude* Driver from that shot instead — see the usable-club
  filter in `compute()`.
- **Green-roll factors** beyond 5 Iron and 3 Wood — though the shot log will now propose
  these from real shots once five clean ones exist for a club.
- Every `GUESS` row in the Shots and Surface drawers.
