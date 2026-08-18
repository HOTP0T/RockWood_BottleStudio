# Pricing formula — as currently implemented

Everything below is the live behaviour of `estimate(s)` in [index.html:241-282](index.html#L241-L282).
Units are centimetres and cm³ internally; prices are USD.

---

## 1. Constants

| Constant | Value | Where |
|---|---|---|
| `GLASS_DENSITY` | `2.5` g/cm³ | [index.html:112](index.html#L112) |
| `COST_PER_GRAM` | `0.00335` USD/g | [index.html:112](index.html#L112) |

**Area factor** `AREAF` — fraction of the bounding rectangle the cross-section fills:

| Shape | `aF` |
|---|---|
| round | 0.785 |
| oval | 0.785 |
| square | 0.96 |
| hexagonal | 0.83 |

**Tooling** `TOOLING` — one-off mould cost for the whole order:

| Shape | Tooling (USD) |
|---|---|
| round | 8,000 |
| oval | 11,000 |
| square | 13,000 |
| hexagonal | 16,000 |

**Material** `MATERIALS` — cost multiplier + minimum order quantity:

| Material | `mult` | MOQ |
|---|---|---|
| Extra Flint | 1.00 | 20,000 |
| Flint | 0.92 | 100,000 |
| Opal / Ceramic | 1.85 | 7,000 |

**Process** `PROCESS` — weight factor `wf` (and glass-distribution factor `dist`, used only for the min-wall check, not for price):

| Process | `wf` | `dist` |
|---|---|---|
| Blow–Blow (`bb`) | 1.00 | 0.55 |
| Press–Blow / NNPB (`nnpb`) | 0.82 | 0.85 |

---

## 2. Step by step

### 2.1 Perimeter and depth ratio

`k` (depth ratio) is `bodyDepth / shoulderWidth` clamped to `[0.3, 1.6]` for **oval**, and exactly `1` for every other shape.

Perimeter at a given width `w` (with depth `w·k`):

```
square      P = 2 · (w + w·k)
hexagonal   P = 1.732 · (w + w·k)
round/oval  P = π · (3(a+b) − √((3a+b)(a+3b)))     a = w/2, b = w·k/2   (Ramanujan ellipse)
```

### 2.2 Glass volume (cm³)

The bottle is treated as four stacked wall bands plus a solid base slug.
Each band contributes *average perimeter × height × wall thickness*:

```
glass = Σ  ((P(w0) + P(w1)) / 2) · h · t
```

| Band | `w0 → w1` | height `h` |
|---|---|---|
| Finish | mouth → mouth | finishHeight |
| Neck | neckWidth → neckWidth | neck |
| Shoulder | neckWidth → shoulderWidth | shoulderHeight |
| Body | shoulderWidth → baseWidth | body |

Plus the base:

```
baseArea = aF · baseWidth · (baseWidth · k)
glass   += baseArea · slug · 0.70
```

The `0.70` is a packing fudge — the base slug is not a perfect solid disc.

### 2.3 Glass weight

```
weightG = glass · GLASS_DENSITY · process.wf
        = glass · 2.5 · wf
```

### 2.4 Unit price (the number shown as "Est. price")

```
wallMult  = 1 + 0.08 · |wallCurvature|
unitGlass = weightG · COST_PER_GRAM · material.mult · wallMult
unit      = unitGlass          ← glass only; no closure, no decoration, no freight
```

`unit` is `costUSD` — the big price on the hero, the refine panel and the estimate page.

### 2.5 Tooling and order totals

```
qty            = max(1, quantity)
tooling        = TOOLING[shape]
toolingPerUnit = tooling / qty
unitAllIn      = unit + toolingPerUnit
totalOrder     = unit · qty + tooling
belowMoq       = qty < material.moq        → red warning, does not change the price
```

### 2.6 Capacity (does not enter the price directly)

Capacity is computed from the same four bands as conical frustums of the **inner** cross-section
(`innerA(w) = aF · (w − 2t) · (w·k − 2t)`):

```
cap  = Σ  h/3 · (A0 + A1 + √(A0·A1))
cap -= innerA(baseWidth) · punt · 0.5
fill = cap · (1 − headspace)
```

It only affects price *indirectly*: with **hold capacity** on, changing any dimension re-solves body
height to hit the target ml, and a taller/shorter body changes the glass volume — and so the price.

---

## 3. Complete formula in one line

```
price/bottle = ( Σ_bands ((P(w0)+P(w1))/2 · h · t)  +  aF·baseW²·k · slug · 0.70 )
               × 2.5                        (density, g/cm³)
               × process.wf                 (1.00 bb / 0.82 nnpb)
               × 0.00335                    (USD per gram)
               × material.mult              (0.92 – 1.85)
               × (1 + 0.08·|wallCurvature|)

order total  = price/bottle × qty + tooling[shape]
```

---

## 4. Worked example — the default bottle

Round · Flint · 50,000 units · default dimensions (Ø 8.6 cm, body 15 cm, wall 0.25 cm, slug 1.4 cm):

| Quantity | Value |
|---|---|
| Glass volume | 181.85 cm³ |
| Glass weight | 454.6 g |
| Brimful capacity | 838.5 ml |
| Fill capacity (4.5 % headspace) | 800.8 ml |
| **Price / bottle** | **$1.40** |
| Tooling / unit (8,000 / 50,000) | $0.16 |
| All-in / bottle | $1.56 |
| **Order total** | **$78,058.39** |

---

## 5. Things worth knowing about the current model

- **`wallCurvature` is always 0.** The state field `wall` exists ([index.html:173](index.html#L173)) and
  feeds `wallMult`, but no UI control writes to it any more — so `wallMult` is effectively always `1.00`.
- **`process` is always `bb`.** Same story: initialised to `'bb'` ([index.html:171](index.html#L171)) with no
  control to switch it, so `wf` is effectively always `1.00`. The NNPB 0.82 factor is unreachable in the UI.
- **The headline price excludes tooling.** "Per bottle" = glass only. "All-in / bottle" adds
  amortised tooling; the order total includes tooling once.
- **MOQ is advisory.** Falling below it triggers a red warning but never changes the price.
- **Closures are not priced.** `unit = unitGlass` by design — the studio only sells the bottle.
- **Everything is a placeholder.** The quote PDF labels the figure `placeholder · ±10%`
  ([index.html:1064](index.html#L1064)); the UI says `est. ±10%`.
- **Cost is purely a function of glass mass.** Shape affects price only through perimeter/area
  geometry and tooling — there is no per-shape or per-complexity manufacturing surcharge, no
  decoration cost, no freight, no scrap/yield allowance, and no volume discount beyond tooling
  amortisation.
