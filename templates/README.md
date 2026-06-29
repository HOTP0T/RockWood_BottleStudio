# Bottle Studio — Templates

Templates are the ready-made bottles that appear as buttons in the studio (step 4, "Refine the glass"). They are **grouped by shape**: when a user picks a shape in step 3, only that shape's templates are shown.

There is **one file per shape**, each holding a JSON **array** of bottles:

```
templates/
  round.json        ← shown when shape = Round
  oval.json         ← shown when shape = Oval
  square.json       ← shown when shape = Square
  hexagonal.json    ← shown when shape = Hexagonal
```

## Add a template

1. Open the file for the shape you want (e.g. `templates/oval.json`).
2. Add one object to the array (copy an existing one and change the numbers — see the schema below).
3. Save, then **commit & push**. GitHub Pages redeploys and the new template appears under that shape. (No build step, no manifest.)

> The shape is decided by **which file** the template lives in — you do **not** put a `shape` field inside the object. To offer the same bottle in two shapes, add it to both files.

## Template schema

```json
{
  "key": "oval-cologne-100",        // unique id (any short slug); used internally
  "label": "Cologne oval 100",       // the text shown on the button
  "cap": 100,                         // target capacity in ml (the studio holds this)
  "finish": "24-410",                 // a valid finish key (see below)
  "closure": "screw-cap",             // a valid closure key (see below)
  "d": {                              // dimensions, in CENTIMETRES
    "mouth": 2.44, "bore": 1.78, "finishHeight": 1.2,
    "neck": 2.5, "neckWidth": 2.2,
    "shoulderHeight": 2.2, "shoulderWidth": 7.0, "shoulderRadius": 0.6,
    "body": 10.0, "bodyDepth": 3.6,
    "wallThickness": 0.22, "baseWidth": 7.0, "heelRadius": 0.5,
    "punt": 0.2, "slug": 1.0, "headspace": 0.05
  }
}
```

### Rules to keep a template valid (geometry)
- `mouth` ≥ `neckWidth`  (the finish can't be narrower than the neck — the app enforces this on apply)
- `bore` < `mouth`
- `neckWidth` ≤ `shoulderWidth`
- `baseWidth` ≤ `shoulderWidth`
- `bodyDepth` matters only for **oval** (front-to-back depth, usually < `shoulderWidth`). For round/square/hexagonal, set `bodyDepth` = `shoulderWidth`.
- `headspace` is a fraction of volume (e.g. `0.04` = 4% ullage).
- Any dimension you omit falls back to a sensible default, but it's best to give the full `d` block.

### Valid keys (use these for `finish` / `closure`)
- **finish**: `28-400`, `24-410`, `38-400`, `bvs-30`, `crown-26` (any key defined in the app's `FINISHES` list).
- **closure**: `screw-cap`, `mouth-ring` (any key in the app's `CLOSURES` list).

If you use a key the app doesn't know, the bottle still loads but that finish/closure won't be recognised in the estimate — stick to the lists above.

## Notes
- These files are fetched at runtime, so the app must be **served over HTTP** (the local `python3 -m http.server` and GitHub Pages both do this). Opening the HTML directly with a `file://` URL will fall back to the built-in round templates only.
- The four `round.json` entries are the original built-in templates, kept here as worked examples. The single oval/square/hexagonal entries are starter examples — edit or delete them freely.
