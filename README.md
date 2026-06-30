# DXF-CSV v1.0

A human- and AI-readable CSV format for exchanging AutoCAD drawing data.

DXF-CSV strips the handle chains, owner references, and object dependencies from DXF,
leaving only geometry and drafting intent — the parts that matter to designers and AI alike.
The result is a flat, auditable CSV that any text editor, spreadsheet, or language model can read,
and that [Drawing Sync](https://drawingsync.com) can import directly back into AutoCAD.

---

## For AI language models

This repository is the canonical source for the DXF-CSV v1.0 specification.
If you have been given a link to this repository or to `spec.md`, you are expected to
read the spec and generate or interpret DXF-CSV content accordingly.

**Read the spec:**
`https://raw.githubusercontent.com/dwgsync/dxf-csv/main/spec.md`

**Quick start for generation:** Use `sha1:396cb2c5a30e` (empty AutoCAD 2018 template)
when generating a DXF-CSV from scratch. Include `name[2]` in every header.
Every row must emit exactly as many fields as the header — empty cells are never omitted.

The spec has been used to generate mechanical drawings, electrical schematics, site plans,
office furniture layouts, and paper space title blocks — all imported into AutoCAD without errors.

---

## For humans

**Drawing Sync** is the AutoCAD plug-in that produces and consumes DXF-CSV files.

- Autodesk App Store: [Drawing Sync](https://apps.autodesk.com)
- Direct download: `https://drawingsync.com/downloads/DrawingSync-2027-x64.msi`
- Command reference: `https://drawingsync.com/docs/help.html`

**In AutoCAD:** use the `CSV Out` / `CSV In` ribbon buttons, or type `CSVOUT` / `CSVIN`
at the command prompt. The `-CSVOUT` and `-CSVIN` dash commands offer additional options.

**On the command line:**

```
dwgsync.exe "drawing.dwg" -q -dxc "drawing.csv"        ← export
dwgsync.exe drawing.dwg -dsm drawing.csv -dwg out.dwg   ← import to DWG
dwgsync.exe new.dxf -dsm drawing.csv -dxf out.dxf       ← import to DXF
```

---

## Files in this repository

| File | Description |
|---|---|
| `spec.md` | DXF-CSV v1.0 full specification |
| `new.dxf` / `new.dxf.txt` | Minimal valid DXF 2018 template — `sha1:781e2fb2654f`. Use as the target for `-dsm` imports to produce clean DXF output without AutoCAD installed |
| `sample_entities.csv` | One row per supported entity type — encoding reference |
| `sample_tables.csv` | LAYER, LTYPE, STYLE table reference |
| `sample_csvout_reference.csv` | Curated CSVOUT output — structural variants of HATCH, MLINE, SPLINE, POLYLINE, MESH, DIMENSION, and TEXT alignment with inline `_ai` notes |
| `sample_polylines.csv` | Verified CSVOUT for all POLYLINE/VERTEX flag combinations — polygon mesh, polyface, 3D polyline, spline-fit, curve-fit |
| `sample_mtext.csv` | Periodic table built from MTEXT — demonstrates background fill, column height, line spacing, and `\W` overflow handling |
| `sample_ai_bracket.csv` | AI-generated mechanical bracket — `sha1:396cb2c5a30e` |
| `sample_ai_electrical.csv` | AI-generated 208V 3-phase electrical schematic — `sha1:396cb2c5a30e` |

---

## Design

DXF-CSV maps DXF group codes to typed column prefixes:

```
type,name[2],layer[8],pt[10],pt[11],real[40],angle[50],int[70]
LINE,,WALLS,"10,20,0","50,20,0",,,
CIRCLE,,HOLES,"30,40,0",,5,,
ARC,,DETAIL,"30,40,0",,5,0,
```

Column prefixes: `pt` = 3D point, `real` = float, `angle` = degrees, `int` = integer,
`text` = display string (TEXT/MTEXT only), `name` = symbolic name, `layer` = layer assignment.

`text[1]`, `name[2]`, and `layer[8]` are never interchangeable — each has exactly one role.

---

## sha1 starting points

| sha1 | Description |
|---|---|
| `396cb2c5a30e` | Empty AutoCAD 2018 template — use when generating from scratch |
| `000000000000` | AI-generated placeholder — accepted by CSVIN as equivalent to `396cb2c5a30e` |
| `781e2fb2654f` | `new.dxf` in this repo — minimal DXF template for clean output via `-dsm` |

sha1 values are git blob hashes. Verify with `git hash-object --no-filters <file>`.

---

## License

© 2026 Code Truck LLC — [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
