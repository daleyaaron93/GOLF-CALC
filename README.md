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

## KNOWN BUG — live, reproducible, unfixed

`autoShot()` returns `g("knock")` for **wind ≥ 12 mph at 100+ yards**. No shot has the
key `knock` — it is `lknock` (Long knockdown). `g()` returns `undefined`, and `compute()`
then throws on `shot.windMult`.

```
S.shot='auto'; S.dist=150; S.wind=14; S.from='turf';
compute()  →  TypeError: Cannot read properties of undefined (reading 'windMult')
```

Reachable any windy hole over 100 yards on Auto. Left unfixed **deliberately**: the
one-word change to `lknock` is obvious, but *which* shot the auto-picker should choose
in that situation is a domain call. Ask Mr. Daley, then fix.

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

Single file, no framework, no persistence.

**Tables:** `BAG`, `SHOTS`, `WINDOWS`, `INTERP`, `SURFACES`, `LANDING`, `WINDS`, `GREENSPEED`.
**State:** `S` — the shot (distance, wind, lie, surface, shot type…).

**Flow:** two views, `vEntry` and `vResult`, toggled by the `.view.on` class.
"Give me the shot" runs `compute()`, which picks the shot (`autoShot()` when on Auto),
filters `BAG` to usable clubs, then `solve()` binary-searches the dial that stops on
target, using `fly()` to model carry → roll → stop across the lie range. Output is painted
into `.screen`.

**Calibration:** four separate `<details>` drawers (bag, shot types, landing surface,
wind/elevation rates) that edit the live tables in place.

**No `localStorage`.** State and calibration reset on reload.

> ⚠️ If you were handed a `build/` folder with its own README, **it describes a rewrite
> that is not what ships here.** That build has one sticky screen, steppers on every
> number, a single tabbed calibration drawer, `localStorage` persistence under
> `playslike.v2`, and the `autoShot` fix. **None of that is in this repo.** Only its
> design system was adopted. Trust this README and the code, not that one.

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

The icon is a deep green field with brass pin and pennant, cream ball, double hairline
ring. Artwork bleeds to the full edge of the square so iOS's own corner mask does all the
rounding — do not bake rounded corners back into the PNGs.

---

## Still unmeasured

- **Driver windows not collected.** Driver is absent from `WINDOWS` entirely, so it falls
  through to the ratio estimate and is flagged. Adding a `WINDOWS["Driver"]` entry with a
  missing shot key would *exclude* Driver from that shot instead — see the usable-club
  filter in `compute()`.
- **Green-roll factors** beyond 5 Iron and 3 Wood.
- Every `GUESS` row in the Shots and Surface drawers.
