# DXF-CSV v1.0 Schema Specification

**Audience:** AI language models  
**Format:** Plain CSV with structured metadata  
**Purpose:** Exchange AutoCAD drawing data in a form that is easy to analyze, visualize, query, and modify — without requiring AutoCAD or DXF parsing libraries  
**Maintained by:** DrawingSync / Code Truck — https://drawingsync.com  
**Last updated:** 2026-07-12 — matches `last-updated:` clause in CSVOUT metadata

---

## Quick orientation

A DXF-CSV file is a standard CSV where every row is one DXF entity or table record. Column headers encode DXF group codes using the pattern `semanticname[code]`. A metadata string at the end of the file (after a blank separator line) provides format context. If you are reading this spec, you likely arrived here via a `dxfcsv:` clause in that metadata string.

---

## Generating a DXF-CSV file

DXF-CSV files have three sources:

- **Drawing Sync** — the flagship AutoCAD plug-in, exports from an open drawing via `CSVOUT`
- **AI directly** — an AI language model can generate a valid DXF-CSV from scratch given this spec and a design description, with no drawing required
- **Any conforming tool** — any program that reads this spec and produces correctly structured CSV

The AI path is significant: a user can describe a design in plain language, share this spec URL, and receive a complete importable CSV. This has been demonstrated with mechanical brackets, electrical schematics, site plans, office furniture, and paper space layouts — all generated as DXF-CSV without a source drawing. Use `sha1:396cb2c5a30e` (empty AutoCAD 2018 template) when generating from scratch.

Drawing Sync is available from the Autodesk App Store or by direct download:

**Direct download:** `https://drawingsync.com/downloads/DrawingSync-2027-x64.msi`  
**Default install path:** `%ProgramFiles%\Code Truck\Drawing Sync\dwgsync.exe`  
The installer and all binaries are code-signed.

### Basic usage

DXF-CSV files are created from AutoCAD and imported into AutoCAD.

Users create CSV files using the **CSV Out** button in the ribbon or by entering `CSVOUT` at the `Command:` prompt. Users can read CSV files created or modified by AI using the **CSV In** button in the ribbon or by entering `CSVIN` at the `Command:` prompt.

Dash commands `-CSVOUT` and `-CSVIN` offer additional options documented in the command reference: `https://drawingsync.com/docs/help.html`

### Command line usage

`dwgsync.exe` supports batch and headless operation without opening the AutoCAD UI. AutoCAD must be installed to process `.dwg` files. `.dxf` files can be processed without AutoCAD.

**Export CSV from a drawing:**
```
dwgsync.exe "sample.dwg" -q -dxc "sample.csv"
dwgsync.exe "sample.dxf" -q -dxs=TABLES,BLOCKS,ENTITIES -dxc "sample.csv"
```

**Import CSV into a drawing:**
```
dwgsync.exe sample.dwg -dsm sample.csv -dwg sample-out.dwg
dwgsync.exe new.dxf   -dsm sample.csv -dxf sample-out.dxf
```

| Flag | Meaning |
|---|---|
| `-q` | Quiet mode — exit after processing without opening the AutoCAD UI |
| `-dxc <path>` | Export CSV file path |
| `-dsm <path>` | Merge CSV into the drawing — assumes table and block dependencies are available to match the `sha1:` source drawing |
| `-dwg <path>` | Write output as a DWG file. AutoCAD must be installed |
| `-dxf <path>` | Write output as a DXF file. Does not require AutoCAD |

To import into a new empty drawing, the target file should not exist — Drawing Sync will create it from the standard template. The sha1 for the standard empty DXF template is `781e2fb2654f`. When importing into a new or empty drawing, use the full `-dxs` export so all required table entries are present in the CSV:

### Section filter — `-dxs`

By default all sections are exported. The `-dxs` flag limits which sections and tables are included:

```
-dxs=TABLES(LAYER),BLOCKS,ENTITIES                               ← LAYER table only; all blocks; all entities (default when omitted)
-dxs=TABLES,BLOCKS,ENTITIES                                      ← full drawing data; all tables, blocks, entities; recommended for import into drawings empty or unrelated to the `sha1:` source
-dxs=TABLES,ENTITIES                                             ← all tables; entities; no BLOCKS section
-dxs=TABLES(LTYPE),TABLES(STYLE),BLOCKS('block name')           ← LTYPE and STYLE tables; one named block
-dxs="TABLES(LTYPE),TABLES(STYLE),BLOCKS('b1'),BLOCKS('b2')"   ← two named blocks; quote when names have spaces
```

**Default behavior:** When the drawing has no BLOCK definitions or non-LAYER table entries, CSVOUT defaults to `TABLES(LAYER),BLOCKS,ENTITIES`. The LAYER table is always included. Other table types (LTYPE, STYLE, DIMSTYLE) only appear when explicitly requested. The BLOCKS section is always emitted even when empty (SECTION/ENDSEC pair only). A file exported without specifying LTYPE or STYLE may reference linetypes and styles by name without defining them — CSVIN creates or verifies them using AutoCAD defaults on import.

**Syntax rules:**
- Section names: `TABLES`, `BLOCKS`, `ENTITIES`
- A section name without parentheses includes everything in that section
- Parentheses specify a single name — repeat the keyword for each additional entry: `TABLES(LTYPE),TABLES(LAYER)` or `BLOCKS('name1'),BLOCKS('name2')`
- Multiple entries are comma-separated
- Quote the full `-dxs` value when any block name contains spaces

**For users — exporting all data:** When a drawing contains block definitions that should be included in the CSV, use **`Comma separated values ALL tables,blocks,entities (*.csv)`** in the standard file save dialog, or use the `-CSVOUT` dash command and select the All option. The default `CSVOUT` command may omit block definitions depending on the drawing. If an AI reports that INSERT rows reference blocks that have no matching BLOCK definition in the file, re-export using the All option.

**For AI — missing block definitions:** If a CSV contains INSERT rows but the BLOCKS section is empty or absent (SECTION/ENDSEC with no BLOCK rows between them), the block geometry was not included in the export. Do not attempt to infer or reconstruct the block geometry. Instead, inform the user that the file was exported without block definitions and ask them to re-export using `Comma separated values ALL tables,blocks,entities (*.csv)` or the `-CSVOUT` dash command All option.

**`-dxs=*` raw dump:** Passing `*` as the section filter exports the full DXF — HEADER, TABLES, BLOCKS, ENTITIES, and OBJECTS — with no stripping. The output is a tabular CSV representation of the raw DXF group code stream, not a DXF-CSV v1.0 file. It has no `#DXF-CSV v1.0` metadata line, includes handle columns (`handle[5]`, `id[330]`, etc.), subclass markers (`class[100]`), XDATA, and all bookkeeping that standard CSVOUT strips. Useful for DXF inspection and debugging — not suitable for CSVIN import or AI generation workflows. An AI receiving a `-dxs=*` output should treat it as raw DXF tabular data rather than a conforming DXF-CSV file.

---

## File structure

### Encoding

Files are UTF-8 encoded with BOM (EF BB BF). The BOM is emitted by default by CSVOUT and should be preserved by any tool that reads and rewrites a DXF-CSV file. Non-ASCII characters (π, ±, °, and similar) are valid in `text[1]` and other string fields. AI consumers do not require the BOM and handle UTF-8 correctly without it. Tools that strip the BOM silently (some Unix pipelines, naive concatenation) may cause non-ASCII characters to display incorrectly in Excel and other consumer applications.

CSVOUT emits `\r\n` line endings by default (Windows convention). When creating or modifying a DXF-CSV file, `\n` (LF only) is preferred — it is cleaner, avoids double-CR issues in text pipelines, and is accepted by both Excel and CSVIN.

### Layout

```
type,layer[8],pt[10],pt[11],real[40],...        ← header row (row 0)
LINE,WALLS,10.0,20.0,5.0,...                    ← data rows
...
                                                ← blank separator line
#DXF-CSV v1.0 | audience:ai | source:... | ... ← fixed metadata (always present)
#DXF-CSV-cond | space:model-and-paper | ...     ← conditional metadata (when conditions met)
#zombies:3DSOLID HELIX MESH WIPEOUT             ← zombie line (when zombies present)
```

A blank separator line — entirely empty, no trailing comma — is required between the last data row and the metadata lines. Without it, Excel and other table-aware tools treat the metadata as additional data rows and the table fails to parse cleanly; the blank line is what causes Excel to stop the table there, keeping metadata outside the formatted range. The three trailing lines that follow are all in the `type` column — all other columns on those rows are empty.

- `#DXF-CSV v1.0` — fixed clauses, always present. Split on ` | ` to parse.
- `#DXF-CSV-cond` — conditional clauses, present only when the drawing has features that require them. Split on ` | ` to parse. Absent when no conditions are met.
- `#zombies:` — space-delimited zombie type list. Absent when no zombies exist. Parse by splitting on `:` then splitting the value on spaces.

---

## Metadata string

### Fixed clauses — always present on `#DXF-CSV v1.0` line

| Clause | Meaning |
|---|---|
| `#DXF-CSV v1.0` | Format identifier and version |
| `audience:ai` | Terse encoding is intentional — written for machine consumption |
| `source:filename.dwg sha1:abc123def456` | Source drawing filename and 12-char SHA-1 content hash. Filename is unquoted when it contains no spaces. A filename containing a space is single-quoted: `source:'my file.dwg'`. A literal single quote in the filename is doubled: `source:'rich''s file.dwg'`. `source:` should always name a `.dwg` file — it describes the origin drawing, not the CSV. For AI-generated content with no source drawing use `source:generated.dwg` paired with the appropriate sha1 sentinel. |
| `codes:dxf-r12` or `codes:dxf-2018` | DXF version of the source export. `dxf-r12` = AutoCAD Release 12 entity set. `dxf-2018` = AutoCAD 2018 format. Future versions follow the same pattern |
| `dxfcsv:'https://drawingsync.com/dxfcsv/v1.0/spec.md'` | This document |
| `last-updated:2026-07-12` | Build date of the CSVOUT binary that produced this file — set at compile time. This date also identifies the spec version: a CSV carrying `last-updated:2026-07-12` was produced by the build that corresponds to this edition of the spec. When the spec is updated, both this example date and the CSVOUT binary version advance together |
| `units:xx` | Drawing units — see Units section below |
| `coords:xy z-absent=truly-2D` or `coords:xyz z-absent=truly-2D` | All coordinates are raw drawing units. `xy` = all entities are 2D, no Z values present. `xyz` = 3D or mixed drawing, Z present on some or all entities. Z absent on a point means genuinely 2D — not Z=0. See Coordinates section |
| `design:r12-extended` | Post-R12 entities treated as R12-extended primitives. Handles, owners, subclass markers, CLASSES, OBJECTS stripped. See Design principle section |
| `color:aci absent=BYLAYER` | Color values are AutoCAD Color Index integers; absent cell means BYLAYER |
| `missing:linetype=BYLAYER color=BYBLOCK lwt=-3` | Default values suppressed — absent linetype means BYLAYER, absent color means BYBLOCK, absent lineweight means DEFAULT (-3) |

### Conditional clauses — present on `#DXF-CSV-cond` line when conditions are met

| Clause | Present when |
|---|---|
| `blocks:inline` | Block definitions are present in this export — inline between BLOCK/ENDBLK rows. Absent when the BLOCKS section was excluded via `-dxs` or the drawing has no block definitions |
| `space:model-and-paper` | File contains both model-space and paper-space entities |
| `ext[210]:ocs-normal absent=0 0 1` | Any entity has a non-default OCS extrusion normal — `ext[210]` column is present |
| `mesh:polyline` | 3D polygon mesh POLYLINEs present (legacy mesh, `int[70]` bit 4 set). `int[71]` and `int[72]` give M×N mesh dimensions on each POLYLINE row |
| `mesh:subdiv` | MESH entities present (subdivision mesh, post-2010 smooth mesh entity) |
| `attrib:tag=name[2]` | ATTRIB or ATTDEF entities present — tag identifier is in `name[2]` |
| `paper[67]:paper-space` | File contains paper-space entities — `paper[67]=1` on entities in the default layout (`Layout1`), `paper[67]=2` and above for additional layouts |
| `angle[51]:arc-end or text-oblique` | ARC entities or oblique TEXT present |
| `bulge:tan(theta/4) in real[42]` | Bulge values present on VERTEX or LWPOLYLINE rows |
| `seq[66]:sequence-follows` | Any POLYLINE or INSERT has a following VERTEX or ATTRIB sequence terminated by SEQEND |
| `truecolor:rgb[420]` | Any entity uses true color (group 420) — packed 24-bit RGB integer overriding ACI color |
| `url:'https://github.com/org/repo.git'` | Source drawing is in a known git repository; `sha1` doubles as git blob hash — retrieve with `git cat-file blob <sha1>` |

### Zombie line — present only when zombie entities exist

```
#zombies:TYPE1 TYPE2 TYPE3
```

Space-delimited, alphabetically sorted list of entity type names that exported as zombies — entities whose geometry requires an object enabler that was not loaded. A zombie row will have no geometry columns but may have non-geometric properties (layer, color, linetype, style, paper space). Do not treat zombie rows as errors. See Design principle section.

---

## Design principle

DXF-CSV applies a consistent filter to all DXF input regardless of source version:

**Keep** — drafting semantics: geometry, layer, linetype, color, block structure, text content, dimension definition points, entity type.

**Strip** — AutoCAD bookkeeping: handles (group 5), owner chains (330/360), group 100 subclass markers, CLASSES section, OBJECTS section, extension dictionaries, reactors, XDATA.

**Rationale:** Group 100 subclass markers are redundant with the `type` column. Handles and owner chains serve AutoCAD's internal object graph for reactor/notification dispatch — they have no drafting meaning. The CLASSES and OBJECTS sections register custom object types and non-graphical persistent objects (plot settings, material references, etc.) that are not part of drawing geometry.

Post-R12 entities (LWPOLYLINE, SPLINE, MTEXT, etc.) are included where present, treated as if they were added to R12's orthogonal data model — each entity's properties are independent, with no side-effect dependencies on other objects. This means no observers or reactors are needed to interpret the data.

### Uniform export environment

CSVOUT runs AutoCAD with object enablers deliberately disabled. Object enablers are `.dbx` modules that load custom object types into the drawing database — Civil 3D, Architecture, MEP, and other vertical products each ship their own. When enablers are active, custom objects may serialize geometry in undocumented, enabler-dependent formats or refuse to serialize at all. Disabling enablers produces a **uniform environment** across all drawing types. Every drawing — plain AutoCAD, Civil 3D, Architecture — exports through the same code path. Entities whose geometry requires an enabler appear as **zombie entities**: the type name and non-geometric properties (layer, color, linetype, style, paper space) are preserved, but all geometry columns are absent.

Zombies are placeholders, not errors. The entity exists in the drawing with its correct layer and properties. On round-trip import, nothing is lost — the original DWG (identified by `source:` and `sha1:`) holds the complete object, and the importer restores it from there rather than from the CSV geometry.

The zombie list reflects which enablers were absent in the specific export session, not which objects are "third-party." Any entity type — including plain AutoCAD entities — can appear as a zombie when its handler is absent from the running AutoCAD session.

**Single-entity-type column suppression:** Columns that would be populated by only one entity type and carry no cross-entity semantic value are suppressed — the column is not added to the header even if the group code is present in the source DXF. This keeps the column set focused on broadly useful data. Example: `char[281]` (PDFUNDERLAY contrast) and `char[282]` (PDFUNDERLAY fade) are suppressed when PDFUNDERLAY is the only entity that would populate them. The entity is still exported — only the single-entity-type columns are dropped.

**Implication for import:** CSVIN operates in two modes depending on the target drawing:

- **Modify mode** — the target is the original source drawing identified by `source:` and `sha1:`. CSVIN verifies the hash matches the open drawing and updates changed entities in place via `acdbEntMake` / `acdbEntMod`. No DXF is generated or consumed.
- **Create mode** — the target is a known published template (e.g. `sha1:781e2fb2654f`, `new.dxf`). CSVIN merges the CSV into the template to produce a new DXF or DWG. The sha1 is a public constant — any AI generating for this sha1 knows exactly what tables and handles are present. The resulting DXF from Drawing Sync's own writer contains only drafting content; the OBJECTS section remains minimal, unlike AutoCAD-written DXF which appends thousands of lines of plot settings and render data.

In both modes the CSV carries geometry and drafting intent; bookkeeping is never round-tripped.

**Derived counts are absent:** Counts that are fully redundant with their data columns are omitted from the CSV. CSVIN derives them by counting fields — the count column would add no information. This applies to:
- SPLINE: `int[72]` (knot count derived from `real[40]`), `int[73]` (control point count from `pt[10]`), `int[74]` (fit point count from `pt[11]`)
- MESH: `int[92]` (vertex count from `pt[10]`), `int[95]` (crease count, always equal to `int[94]`)

AI generators must not emit these columns. A consumer reading a DXF-CSV file should not expect them. This is a design rule, not a per-entity exception.

**sha1 as drawing state contract:** The `sha1:` clause is not just documentation — it is a contract between the CSV and the drawing state. CSVIN verifies the hash before importing; if the open drawing doesn't match, import is rejected. This mechanism supports a library of known starting points. An AI generating content for `sha1:396cb2c5a30e` (empty AutoCAD 2018 template) makes no assumptions about pre-existing blocks, layers, or styles — it must define everything it uses. An AI generating content for a domain-specific template sha1 can reference blocks, layers, dimstyles, and text styles that already exist in that drawing without redefining them, keeping the generated CSV lean and focused on design intent rather than infrastructure. The sha1 is the key; the drawing is the state. One exception to strict matching: an all-zero sha1 (`000000000000`) is treated as equivalent to `396cb2c5a30e` — AI consumers generating from scratch with no real source drawing commonly emit this placeholder rather than looking up the real hash, and CSVIN accepts it with the same "no pre-existing references" assumption.

**Open library:** Anyone can define a template drawing, publish it, and document its sha1. No central registry or approval is required — if the sha1 matches the open drawing, CSVIN imports cleanly and the expectations encoded in that template are guaranteed. Domain communities can maintain their own templates: architectural (standard layer names, door and window blocks, annotation styles), electrical (symbol libraries, IEC or NFPA layer conventions), civil (survey layers, coordinate systems), mechanical (ASME title blocks, GD&T styles). An AI targeting a known template sha1 can skip all table definitions and generate only entities — the smallest possible CSV for the most complete result.

Currently published starting points:

| sha1 | Description |
|---|---|
| `396cb2c5a30e` | Empty AutoCAD 2018 template (acad.dwt) — no blocks, no named layers beyond `0`, no styles beyond `Standard`. Use when generating from scratch for import into AutoCAD. |
| `000000000000` | AI-generated placeholder, equivalent to `396cb2c5a30e` — means "start from the latest AutoCAD template" with no references beyond the standard ones. AI consumers without a real source drawing commonly emit this all-zero sha1 rather than `396cb2c5a30e` itself; CSVIN treats it the same way: no pre-existing blocks, layers, or styles may be assumed beyond `0` and `Standard`. |
| `781e2fb2654f` | `new.dxf` — Drawing Sync minimal DXF template, available at `https://drawingsync.com/dxfcsv/v1.0/new.dxf` or `https://drawingsync.com/dxfcsv/v1.0/new.dxf.txt` (text-accessible alias for browsers and AI fetch). AC1032 (DXF 2018), 308 lines. Contains the minimum valid structure: `*Model_Space` and `*Paper_Space` block records with handles, ByBlock/ByLayer/Continuous linetypes, layer `0`, Standard style and dimstyle, ACAD appid. OBJECTS section is a single empty DICTIONARY — no AutoCAD bookkeeping bloat. Use with `-dsm` to produce clean DXF output independent of AutoCAD. |

---

## Column headers

Headers follow the pattern `semanticname[code]` where `code` is the DXF group code number. All group codes are **entity-scoped** — the same code number means different things on different entity types, exactly as in the DXF specification. When in doubt, look up the code against the `type` column value, not the column header.

**The column set in any given file reflects only the group codes present in that export.** Columns are added to the header when a group code appears in the data — not when a section or table is included. A file with no arc entities will have no `angle[51]` column. A file exported without DIMSTYLE will have none of the dimvar columns. Always read the header row to determine the column set for that file.

**`name[2]` is required in every valid DXF-CSV file.** Every file contains SECTION, ENDSEC, LAYER, and LTYPE rows — all of which use `name[2]`. Omitting `name[2]` from the header leaves these rows nameless and the file unimportable. When generating a DXF-CSV file, always include `name[2]` in the header regardless of what other columns are present.

### Column prefix reference

| Prefix | DXF resbuf type | Meaning |
|---|---|---|
| `type` | RT_STR | Entity or record type (group code 0) |
| `text` | RT_STR | String value |
| `name` | RT_STR | Name string (block, layer, style, etc.) |
| `pt` | RT_3DPOINT | 2D or 3D coordinate — `x,y` or `x,y,z` |
| `real` | RT_REAL | Floating-point scalar |
| `angle` | RT_REAL | Angle in degrees |
| `int` | RT_SHORT | 16-bit signed integer |
| `long` | RT_LONG | 32-bit signed integer |
| `char` | RTCHAR | Single-byte integer (0–255). DXF group codes 280–289 — used for flags, boolean-like values, and small counts |
| `seq` | RT_SHORT | Structural sequence flag (group code 66) |
| `paper` | RT_SHORT | Paper-space flag (group code 67) |
| `color` | RT_SHORT | ACI color index (group code 62) |
| `lwt` | RT_SHORT | Lineweight (group code 370) |
| `ext` | RT_3DPOINT | OCS extrusion normal vector (group code 210) |
| `elev` | RT_REAL | Elevation (group code 38) |
| `thick` | RT_REAL | Thickness (group code 39) |
| `style` | RT_STR | Style name (group code 7) |
| `linetype` | RT_STR | Linetype name (group code 6) |
| `variable` | RT_STR | Header variable string (group code 9) |

### Standard columns

| Header | Code | Meaning |
|---|---|---|
| `type` | 0 | Entity type or record type. Primary key of every row. |
| `text[1]` | 1 | String value — TEXT content, ATTRIB value, DIMENSION override text. May contain MTEXT inline formatting codes (`\A1;`, `{\H...}`, etc.) when the source entity carried formatted text — treat the same as MTEXT content |
| `name[2]` | 2 | Name — block name (INSERT/BLOCK), layer name (LAYER), section/table name, ATTRIB/ATTDEF tag |
| `text[3]` | 3 | Entity-scoped string. On BLOCK rows: absent when equal to `name[2]`. On STYLE rows: font or shape filename (e.g. `txt`, `romans.shx`, `ltypeshp.shx`) when it differs from `name[2]` — absent when it would duplicate the style name. On LTYPE rows: human-readable description string (e.g. `Solid line`, `Dashed (.5x) _ _ _ _`) as shown in the AutoCAD linetype dialog |
| `linetype[6]` | 6 | Linetype name. Absent = BYLAYER |
| `style[7]` | 7 | Text style name. References STYLE table |
| `layer[8]` | 8 | Layer name. Present on all geometry entities |
| `pt[10]` | 10 | Primary point. Single `x,y` or `x,y,z` for most entities. Space-delimited vertex list for LWPOLYLINE |
| `pt[11]` | 11 | Second point — LINE end, 3DFACE corner 2, TEXT alignment point, DIMENSION pt 2 |
| `pt[12]` | 12 | Third point — 3DFACE/SOLID corner 3. On SPLINE: start tangent vector (independently optional) |
| `pt[13]` | 13 | Fourth point — 3DFACE/SOLID corner 4, DIMENSION pt 4. On SPLINE: end tangent vector (independently optional) |
| `pt[14]` | 14 | Fifth point — DIMENSION pt 5 (first extension line start) |
| `pt[15]` | 15 | Fifth point — entity-scoped. Absent when no entity in the export uses this code |
| `pt[16]` | 16 | Sixth point — entity-scoped. Absent when no entity in the export uses this code |
| `elev[38]` | 38 | Elevation — Z offset applied to all vertices of a flat entity (LWPOLYLINE, etc.) at draw time, rather than encoding Z in each vertex. Absent = 0. Combine with `thick[39]` for 2.5D extrusion: a LINE at z=0 with `elev[38]=2.5` draws at z=2.5; add `thick[39]=2.5` to extrude it 2.5 units further in Z |
| `thick[39]` | 39 | Extrusion thickness — Z depth applied to flat entities. Absent = 0. A LINE, ARC, or LWPOLYLINE with `thick[39]` becomes a surface extruded in the Z direction. Combined with `elev[38]`, these two fields give flat entities a 2.5D presence without requiring 3D entities |
| `trans[440]` | 440 | Entity transparency. Packed integer: top byte `0x02` = entity-level transparency type; lower 3 bytes = raw transparency value where 0 = fully opaque and 255 = fully transparent. Transparency percent ≈ `(lower_byte / 255) × 100`. Example: `0x02000026` = 38/255 ≈ 15% transparent. Absent = opaque. LAYER transparency uses a similar encoding |
| `real[40]` | 40 | Floating scalar — radius (CIRCLE/ARC), text height (TEXT/MTEXT), start width (POLYLINE), overall scale (MLINE) |
| `real[41]` | 41 | Floating scalar — x-scale (INSERT), end width (POLYLINE), text width factor (TEXT). On MTEXT: defined width (reference rectangle width, 0=undefined). On MLINE: element parameters comma-delimited |
| `real[42]` | 42 | Floating scalar — bulge (VERTEX/LWPOLYLINE), y-scale (INSERT). On MTEXT: actual height (AutoCAD-computed, read-only) |
| `real[43]` | 43 | Floating scalar — z-scale (INSERT), constant width (LWPOLYLINE). On MTEXT: actual width (AutoCAD-computed, read-only) |
| `angle[50]` | 50 | Angle in degrees — start angle (ARC), rotation (INSERT/TEXT), POINT display angle |
| `angle[51]` | 51 | Angle in degrees — end angle (ARC), oblique angle (TEXT) |
| `angle[53]` | 53 | Angle in degrees — entity-scoped. On HATCH: hatch pattern angle |
| `color[62]` | 62 | ACI color index. Absent = BYLAYER. Negative = layer is frozen/off (use absolute value for color) |
| `lwt[370]` | 370 | Lineweight in hundredths of a mm. Absent = DEFAULT (-3). Common values: 0=hairline, 5,9,13,15,18,20,25,30,35,40,50,53,60,70,80,90,100,106,120,140,158,200,211. Special: -1=BYLAYER, -2=BYBLOCK |
| `plotst[380]` | 380 | Plot style index — entity-level plot style assignment. Integer enumerator. Absent = BYLAYER. Passed through as-is; exact enumeration values are DXF-internal |
| `long[420]` | 420 | True color as packed 24-bit RGB integer: `(R << 16) | (G << 8) | B`. When present, overrides `color[62]` for display. Absent = use `color[62]` (ACI). Present only when `truecolor:rgb[420]` is in the conditional metadata. Valid on both entity rows and LAYER table rows — a LAYER row with `long[420]` sets the layer's true-color display independent of its ACI `color[62]` |
| `seq[66]` | 66 | Sequence-follows flag. Always 1 when present. On POLYLINE rows: **always required** — indicates VERTEX rows follow, terminated by SEQEND. On INSERT rows: required when ATTRIB rows follow, terminated by SEQEND; omit when no ATTRIBs follow. In both cases SEQEND will always be present after the sequence. Retained as a structural aid — without it, a reader encountering an INSERT would require read-ahead to determine whether ATTRIB rows follow. |
| `paper[67]` | 67 | Paper-space layout index. Value identifies which paper space layout the entity belongs to: `1` = first layout (`*Paper_Space`, default `Layout1`), `2` = second layout (`*Paper_Space0`), `3` = third (`*Paper_Space1`), and so on — the `*Paper_Space` BLOCK_RECORD name suffix increments as `0`, `1`, `2`... for layouts beyond the first. Absent on model-space entities. Emitted on all paper-space entities including VIEWPORTs. CSVIN import planned: entities with `paper[67]=2` and above are the target (multi-layout support) |
| `int[68]` | 68 | VIEWPORT status flags |
| `int[69]` | 69 | VIEWPORT ID |
| `int[70]` | 70 | Integer flags — entity-scoped. LAYER: 1=frozen, 2=frozen-in-new-viewports, 4=locked, 16=xref-dependent (layer from an attached xref — appears in LAYER table but entities never reference it directly), 64=used. LWPOLYLINE: bit 0 (1)=closed, bit 7 (128)=plinegen (linetype generated continuously across all vertices rather than restarting per segment). POLYLINE: see mesh flags. VERTEX: see vertex flags. BLOCK: 0=regular, 1=anonymous (`*`-prefixed), 2=has-attributes (required when block contains ATTDEFs), 4=xref block definition (external reference — geometry comes from the external file, entities inside not present in CSVOUT), 8=xref overlaid. MLINE: bit 0=closed, bit 1=suppress start caps, bit 2=suppress end caps |
| `int[71]` | 71 | Integer — POLYLINE mesh M vertex count, TEXT generation flags. On MTEXT: attachment point (1=TL, 2=TC, 3=TR, 4=ML, 5=MC, 6=MR, 7=BL, 8=BC, 9=BR) |
| `int[72]` | 72 | Integer — POLYLINE mesh N vertex count, TEXT/ATTRIB horizontal justification. On MTEXT: drawing direction (1=left-to-right, 3=top-to-bottom, 5=by style) |
| `int[73]` | 73 | Integer — TEXT vertical justification (0=baseline 1=bottom 2=middle 3=top). On MTEXT: line spacing style (1=at least, 2=exactly) |
| `int[74]` | 74 | Integer — ATTDEF/ATTRIB vertical justification. On MLINE: per-element parameter counts comma-delimited (absent when all elements have 2 parameters) |
| `real[44]` | 44 | Floating scalar — entity-scoped. On INSERT (array): column spacing. On SPLINE: fit tolerance. On MTEXT: line spacing factor (1.0=single, absent=not set) |
| `real[45]` | 45 | Floating scalar — entity-scoped. On INSERT (array): row spacing. On HATCH: pattern line offset X component. On MTEXT: background fill scale factor (~1.0–3.0, 1.5 typical) — only present as part of the background fill tail, see `long[90]` |
| `real[46]`–`real[48]` | 46–48 | Floating scalar — entity-scoped. On HATCH: `real[46]` = pattern line offset Y component. On MTEXT: `real[46]` = defined column height (legacy field, 0/absent when not used). On DIMSTYLE rows: dimension variables |
| `real[48]` | 48 | Linetype scale — entity-level override of the global LTSCALE. Absent = use global scale |
| `real[49]` | 49 | LTYPE element data — comma-delimited list of dash/dot/gap lengths for non-CONTINUOUS linetypes. Positive = dash length, negative = gap length, zero = dot |
| `int[75]`–`int[79]` | 75–79 | Entity-scoped integers. On POLYLINE: `int[75]` = smooth surface type (0=none, 5=quadratic B-spline, 6=cubic B-spline, 8=Bezier). On HATCH: `int[75]`=pattern type, `int[76]`=associativity, `int[77]`=hatch style (0=normal, 1=outer, 2=ignore), `int[78]`=pattern line count, `int[79]`=pixel size. On DIMSTYLE rows: dimension style variables |
| `real[141]` | 141 | Floating scalar — entity-scoped. On ACAD_TABLE: row heights comma-delimited (nROW values). On DIMSTYLE: `real[141]`–`real[147]` dimension variables |
| `real[142]` | 142 | Floating scalar — entity-scoped. On ACAD_TABLE: column widths comma-delimited (nCOL values) |
| `int[170]` | 170 | Integer — entity-scoped. On DIMSTYLE: extended dimension variable. On MULTILEADER: combined leader type — 2 fields (CONTEXT_DATA + common): 1=straight, 2=spline. Absent when both fields are default (`1,1`) |
| `int[171]`–`int[174]` | 171–174 | DIMSTYLE integer variables (extended range) — entity-scoped, only on DIMSTYLE rows |
| `int[175]` | 175 | Integer — entity-scoped. On MULTILEADER: combined text attachment — 2 fields (CONTEXT_DATA + common). Absent when default (`1,0`) |
| `int[176]`–`int[178]` | 176–178 | DIMSTYLE integer variables (extended range) — entity-scoped, only on DIMSTYLE rows |
| `int[179]` | 179 | Integer — entity-scoped. On MULTILEADER: text angle type. Absent when default |
| `ext[210]` | 210 | OCS extrusion normal vector — `x,y,z`. Absent = default `0,0,1` (WCS). Applies to SOLID, CIRCLE, INSERT, and other entities in a non-WCS plane |
| `long[90]` | 90 | Long integer — entity-scoped. On MESH: combined face and edge data (see MESH entity). On MTEXT: background fill flag — triggers a trailing fill tail (`int[63]` fill color, `real[45]` fill scale factor) when set to 1, 3, 16, or 17. When set to 2 (use drawing background color), only `long[90]` itself is present — the fill tail is rejected by `entmake` in that case and must not be emitted. Absent or 0 = no background fill. On ACAD_TABLE: combined column — field[0]=table flags (typically 22), field[1..N]=per-cell value flags (4=has content, 0=empty), where N=nROW×nCOL |
| `long[91]` | 91 | Long integer — entity-scoped. On MESH: subdivision level (absent when 0). On ACAD_TABLE: combined column — field[0]=nROW (row count), field[1..N]=per-cell merged-row-span (262144=no merge) |
| `long[93]` | 93 | Long integer — entity-scoped. On MESH: face data count. On ACAD_TABLE: combined column — field[0]=override count, field[1..N]=per-cell value type (6=content cell, 7=virtual/empty cell) |
| `long[94]` | 94 | Long integer — entity-scoped. On MESH: crease edge count |
| `real[140]` | 140 | Floating scalar — entity-scoped. On MESH: crease values comma-delimited, one per edge (`int[94]` values). Absent when all creases are 0.0 (flat mesh). On DIMSTYLE: `real[140]`–`real[147]` are floating-point dimension variables |
| `int[63]` | 63 | Entity-scoped. On HATCH: fill color override (background color), comma-delimited when multiple loops have different fill colors |
| `real[47]` | 47 | Entity-scoped. On HATCH: pattern line minimum dash length (pixel size). Suppressed when equal to default — this is a display hint, not geometry |
| `char[271]` | 271 | Single-byte integer — entity-scoped. On DIMSTYLE: DIMDEC (decimal places for primary units) |
| `char[272]` | 272 | Single-byte integer — entity-scoped. On DIMSTYLE: DIMTDEC (decimal places for tolerance) |
| `char[280]`–`char[289]` | 280–289 | Single-byte integers (0–255, RTCHAR). Entity-scoped. On PDFUNDERLAY: `char[281]`=contrast (0–100), `char[282]`=fade (0–80). On HELIX: `char[280]`=handedness (0=left, 1=right). On DIMSTYLE: various boolean-like flags. On ACAD_TABLE: `char[280]`=combined column (field[0]=blockref shadow always 0, field[1]=AcDbTable shadow — absent when 0); `char[281]`=table style override flag (absent when 0). On ATTRIB: `char[280]`=combined 2 fields (field[0]=blockref shadow always 0, field[1]=lock/duplicate flag) |
| `bool[291]` | 291 | Boolean (0/1) — entity-scoped. On MULTILEADER: has-dogleg combined — 3 fields (CONTEXT_DATA, LEADER{, common). Absent when all fields at default (`0,1,1`) |
| `bool[292]` | 292 | Boolean (0/1) — entity-scoped. On MULTILEADER: has-text-direction combined — 2 fields (CONTEXT_DATA, common). Absent when default (`0,0`) |
| `int[95]` | 95 | Integer — entity-scoped. On MULTILEADER: arrow style index (absent when default 1). On MESH: crease count — **absent from CSV**, derived by CSVIN from `long[94]` |
| `string[304]` | 304 | String — entity-scoped. On MULTILEADER: mtext text content. Absent when leaderless/compact. DXF sequence markers `LEADER_LINE{` and `}` that share code 304 are skeleton literals injected by CSVIN — never appear as CSV values |
| `long[92]` | 92 | Long integer — entity-scoped. On ACAD_TABLE: nCOL (column count, single value) |
| `long[421]` | 421 | Long integer — entity-scoped. On HATCH gradient: packed 24-bit RGB color(s) comma-delimited (one per gradient color entry). Same bit layout as `long[420]`: `(R << 16) | (G << 8) | B`. Present only on gradient fill hatches. On MTEXT: packed 24-bit RGB override for background fill color — same relationship to `int[63]` as `long[420]` has to `color[62]`: `int[63]` is always present when fill is active (ACI approximation), `long[421]` is present only when the fill color is a true-color RGB value, absent when fill color is an ACI. When present, `long[421]` overrides `int[63]` for display. On ACAD_TABLE: per-cell fill true-color override (ncell comma-delimited, absent when -1 for all cells) |
| `string[302]` | 302 | String — entity-scoped. On ACAD_TABLE: per-cell text content (ncell tab-delimited fields including empty strings for virtual/empty cells). This is the structural cell-content column used by CSVIN; `text[1]` carries only unique non-empty values for AI readability |

**DIMSTYLE group code range:** DIMSTYLE rows use a wide and evolving range of group codes across `real`, `int`, `char`, and `pt` prefixes. The codes listed above cover the most common variables but AutoCAD adds dimension variables with each release. A DIMSTYLE row may contain additional valid codes (e.g. `char[277]` DIMUNIT, `char[280]` per-object linetype flag, `pt[213]` leader direction vector) not individually documented here. All are valid — read the column header and treat any unknown DIMSTYLE code as a dimension variable to preserve round-trip.

### Name-only table rows

Any table row (LAYER, LTYPE, STYLE, DIMSTYLE) may appear with only `name[2]` populated and all other columns absent. This is a valid reference — it declares the name exists and CSVIN will create or verify the entry using AutoCAD defaults. This pattern is common in AI-generated CSV when the consumer wants to reference a standard AutoCAD resource (e.g. `DIMSTYLE name[2]=ISO-25`, `LTYPE name[2]=CENTER`) without specifying every parameter.

---

## Coordinates

All coordinates are raw drawing units — no conversion applied. The `units:` clause identifies the unit system.

**Single-point entities** (LINE, CIRCLE, ARC, INSERT, TEXT, POINT): `pt[10]` contains one coordinate as `x,y` or `x,y,z`.

**Multi-point entities** (3DFACE, SOLID, TRACE, DIMENSION): use `pt[10]` through `pt[14]`, one coordinate per column.

**LWPOLYLINE vertices**: `pt[10]` contains all vertices space-delimited — `x1,y1 x2,y2 x3,y3`. Z is absent for 2D polylines; elevation is in `elev[38]` (check file columns).

**POLYLINE/VERTEX**: vertices are on individual VERTEX rows following the POLYLINE row, terminated by SEQEND. Each VERTEX has one coordinate in `pt[10]`. The POLYLINE row itself carries only elevation in `pt[10]` — X and Y are always zero and carry no meaning.

**Z-absent rule:** CSVOUT follows DXF conventions — flat entity types (LWPOLYLINE, 2D ARC, etc.) do not include a Z component; 3D entity types (POLYLINE/VERTEX, INSERT, etc.) always include Z. A missing Z means the entity type is inherently flat. Z=0 means the entity type carries Z and it happens to be zero. Consumers should treat both as equivalent for flat geometry — `5,15` and `5,15,0` mean the same thing in practice. The `coords:xy` vs `coords:xyz` metadata clause simply reflects which is present in this file.

---

## Bulge

Bulge encodes arc segments within polylines. Formula: `bulge = tan(included_angle / 4)`.

- `0` = straight segment  
- `1` = semicircle (90° included angle × 4 = 360°... actually a quarter circle: included = 4×atan(1) = 180°)
- Positive = arc bows left of the direction of travel (CCW)
- Negative = arc bows right (CW)
- Very large absolute values (e.g. 437, 39073) indicate near-complete circles where the two endpoint vertices are very close together

To recover arc geometry from bulge `b` between points P1 and P2:
```
chord   = distance(P1, P2)
radius  = chord / (2 * sin(2 * atan(|b|)))
sagitta = chord/2 * |b|
```

**In LWPOLYLINE:** bulge values are comma-delimited in `real[42]`, one per vertex, in the same order as the space-delimited vertices in `pt[10]`.

**In VERTEX rows:** bulge is a single value in `real[42]` on the VERTEX row it applies to.

---

## Mesh polylines

When POLYLINE `int[70]` has bit 4 set (value & 16 != 0), the entity is a 3D polygon mesh:

- `int[71]` = M vertex count (rows)
- `int[72]` = N vertex count (columns)  
- Total vertices = M × N, on the following VERTEX rows
- VERTEX `int[70]` = 64 means polygon mesh vertex
- VERTEX `int[70]` = 32 means mesh vertex added by AutoCAD for curve fitting
- POLYLINE `int[70]` bit 5 (value & 32) = closed in M direction
- POLYLINE `int[70]` bit 6 (value & 64) = closed in N direction

---

## Block structure

Block definitions are inline in the CSV between BLOCK and ENDBLK sentinel rows. They are not separate files.

```
BLOCK   name[2]='WIDGET'   layer[8]='0'   pt[10]='0,0,0'    ← definition starts
LINE    ...                                                    ← block geometry
CIRCLE  ...
ENDBLK                                                        ← definition ends
...
INSERT  name[2]='WIDGET'   pt[10]='100,200,0'  angle[50]=45  ← placement in model space
```

To render a placed block: find the BLOCK definition matching `name[2]` on the INSERT row, transform its geometry by the INSERT's position (`pt[10]`), x/y/z scale (`real[41]`/`real[42]`/`real[43]`, absent = 1.0), and rotation angle (`angle[50]`, absent = 0).

### Block strip rules

Two categories of blocks are stripped from the output and never appear as BLOCK/ENDBLK definitions:

**Anonymous blocks** — any block whose name begins with `*`. These are AutoCAD-managed internal blocks. Examples: `*Model_Space`, `*Paper_Space`, `*D12` (dimension rendered geometry), `*U333` (utility blocks).

**Empty blocks** — any block containing no geometry rows between BLOCK and ENDBLK, regardless of name. This includes user-defined blocks whose geometry has been deleted but whose definition was not cleaned up.

### Anonymous block inclusion and `-dxs` interaction

By default, anonymous blocks are stripped. This includes `*D##` dimension geometry blocks, which AutoCAD maintains as a dependent cache of rendered dimension primitives. The DIMENSION entity referencing such a block remains in the export — the `name[2]` field will reference a block definition that is not present. On import, the importer recreates the anonymous geometry from the dimension parameters rather than restoring the cached primitives.

The `-dxs` flag controls which sections are exported. If the user excludes the BLOCKS section entirely, or specifies named blocks only, no block definitions will be present regardless of the anonymous strip rule. In all cases, an INSERT or DIMENSION referencing an absent block definition is valid — treat as a zero-geometry placement, not an error.

Anonymous block inclusion is available as an export option — when included, anonymous blocks appear as BLOCK/ENDBLK pairs with `*`-prefixed names.

### INSERT with no matching BLOCK definition

An INSERT or DIMENSION row whose `name[2]` has no matching BLOCK definition in the file is valid — the referenced block was stripped (anonymous or empty) or excluded via `-dxs`. Treat it as a zero-geometry placement at the specified position, scale, and rotation. Do not treat it as an error.

### BLOCKS section states

The presence or absence of the BLOCKS section in the file is meaningful:

- **BLOCKS section absent** — the user excluded blocks from this export
- **BLOCKS section present, empty** (`SECTION`/`ENDSEC` with no BLOCK rows between) — blocks were exported but no user-defined block definitions exist in the drawing
- **BLOCKS section present, with BLOCK/ENDBLK pairs** — user-defined block definitions are included

Note: AutoCAD always maintains at least `*Model_Space` internally, so the BLOCKS section is never truly empty at the DWG level. An empty BLOCKS section in the CSV means all definitions were stripped (anonymous or empty).

---

## Structural sentinel rows

SECTION, ENDSEC, BLOCK, ENDBLK, and SEQEND are structural sentinels. They carry no entity properties — all columns are empty except `type` (and `name[2]` on SECTION rows). Any properties that may exist on these rows in the source DXF are bookkeeping artifacts and are stripped on export.

`TABLE` and `ENDTAB` are never emitted — they exist in DXF to wrap individual table sections but carry no information beyond what the enclosing SECTION row already provides.

### BLOCK and INSERT conventions

**`BLOCK` `int[70]` flags:** 0 = regular block (or omit), 1 = anonymous (`*`-prefixed, AutoCAD-managed), 2 = has attributes (required when block contains ATTDEFs), 4 = xref block definition (external reference — the block's geometry lives in the external DWG file; no entities will appear inside this BLOCK/ENDBLK pair in CSVOUT), 8 = xref overlaid. Must be set to 2 when the block contains ATTDEFs — this signals AutoCAD to prompt for attribute values on INSERT. Omit or use 0 for blocks without ATTDEFs.

**Color inside block definitions:** Entities inside a BLOCK should use `color[62]=0` (BYBLOCK) so the INSERT's own layer color propagates into the block geometry at reference time. Entities with a specific ACI or true-color override inside a block will always display that color regardless of INSERT layer — use BYBLOCK for reusable symbols and specific colors only when the block is intentionally hardcoded.

**Nested blocks:** A BLOCK definition may contain INSERT rows referencing other blocks — nesting is supported. The nested INSERT appears as a normal INSERT row within the BLOCK/ENDBLK pair.

**`INSERT` `seq[66]`:** Set to 1 when ATTRIB rows follow the INSERT before SEQEND. When absent or 0, no ATTRIB rows follow and no SEQEND is expected for that INSERT.

**`ATTDEF`:** Defines an attribute template inside a BLOCK. `text[1]` = default value, `name[2]` = tag (matched against ATTRIB `name[2]` on INSERT), `text[3]` = prompt string. `int[70]` flags: 0=visible, 1=invisible, 2=constant, 4=verify, 8=preset.

**`ATTRIB`:** Carries a live attribute value on a specific INSERT instance. Mirrors ATTDEF structure — `name[2]` = tag, `text[1]` = actual value, `real[40]` = text height, `pt[10]` = text position. Must be followed by SEQEND (or another ATTRIB then SEQEND) when `seq[66]=1` is set on the INSERT.

`SECTION`/`ENDSEC` for TABLES and BLOCKS appear only when the entire section is included via `-dxs`:
- `-dxs=TABLES` → `SECTION name[2]=TABLES` / `ENDSEC` wrap all table rows
- `-dxs=TABLES(LAYER),TABLES(LTYPE)` → no SECTION/ENDSEC; LAYER and LTYPE rows appear directly
- `-dxs=BLOCKS` → `SECTION name[2]=BLOCKS` / `ENDSEC` wrap all block definitions
- `-dxs=BLOCKS('myblock')` → no SECTION/ENDSEC; BLOCK/ENDBLK for that block appears directly

SEQEND terminates a POLYLINE/VERTEX sequence or an INSERT/ATTRIB sequence. Its presence is preserved for human readability — a reader can always infer termination from context, but the explicit row is kept because people look for it.

---

## ATTRIB / ATTDEF

ATTDEF defines an attribute template inside a block definition. ATTRIB is the filled-in instance attached to an INSERT in model space.

- `name[2]` = tag name (the attribute identifier, e.g. `PART_NUMBER`)
- `text[1]` = value on ATTRIB rows (the actual data, e.g. `WD-4412`)
- `int[70]` = attribute flags: 0=visible, 1=invisible, 2=constant, 4=verify, 8=preset
- `int[72]` = horizontal justification (same as TEXT)

ATTRIB rows immediately follow their parent INSERT row, before SEQEND. An INSERT and its following ATTRIB rows always share the same space — both model or both paper. `paper[67]` will be consistent across the INSERT and all its ATTRIBs.

---

## Paper space

When `space:model-and-paper` is in the metadata, the file contains both model-space and paper-space entities.

- `paper[67]=1` on an entity means it lives in the default paper space layout (`Layout1`, `*Paper_Space` block record). `paper[67]=2` = second layout (`*Paper_Space0`), `paper[67]=3` = third (`*Paper_Space1`), and so on
- VIEWPORT entities define the viewports on the paper layout
- VIEWPORT `real[40]` and `real[41]` are viewport width and height in paper units
- VIEWPORT `int[68]` and `int[69]` are status flags and viewport ID
- VIEWPORT entities are always paper space entities — `paper[67]=1` is emitted on VIEWPORT rows.
- **Viewport layer overrides** (per-viewport layer freeze, color, linetype) are not exported — these are stored as handle lists in extended data and do not survive round-trip. The user sets them manually after import.

**VIEWPORT creation workflow:** CSVIN cannot create VIEWPORT entities (`acdbEntMake` limitation — permanent). For AI-assisted paper space layout, place a LWPOLYLINE rectangle in paper space (`paper[67]=1`) at the desired viewport position and size, with a TEXT entity on layer `_ai` describing the view you want (orientation, scale, layers to freeze). The user creates the actual viewport manually using the rectangle as a guide, then adjusts layer visibility to match the directive. Example:

```
LWPOLYLINE, _ai, "2.5,1.5 14.5,1.5 14.5,9.5 2.5,9.5", paper[67]=1  ← viewport boundary guide
TEXT, _ai, "Create viewport here: 3D isometric view, freeze layers: floorplan centerline labels", paper[67]=1, pt[10]="2.5,9.6"
```

---

## Units reference

| Value in `units:` | Drawing unit |
|---|---|
| `unitless` | No unit system defined (code 0) |
| `in` | Inches |
| `ft` | Feet |
| `mi` | Miles |
| `mm` | Millimeters |
| `cm` | Centimeters |
| `m` | Meters |
| `km` | Kilometers |
| `mil` | Mils (thou, 1/1000 inch) |
| `yd` | Yards |
| `us-ft` | US Survey feet |
| `us-mi` | US Survey miles |
| `unspecified` | `$INSUNITS` absent from source DWG header |

Less common DXF unit codes (`uin`, `angstrom`, `nm`, `um`, `dm`, `dam`, `hm`, `Gm`, `AU`, `ly`, `pc`, `us-in`, `us-yd`) are passed through using the same short-name pattern — see the DXF specification for the full enumeration.

---

## ACI color reference

AutoCAD Color Index (ACI) — integer 1-255. Common values:

| ACI | Color |
|---|---|
| 1 | Red |
| 2 | Yellow |
| 3 | Green |
| 4 | Cyan |
| 5 | Blue |
| 6 | Magenta |
| 7 | White / Black (display-dependent) |
| 8-255 | Extended palette |
| absent | BYLAYER |
| 0 | BYBLOCK |
| 256 | BYLAYER (explicit) |

Negative ACI in the LAYER table means the layer is frozen or off. Use `abs(value)` for the display color.

---

## Entity quick reference

| Type | Key columns | Notes |
|---|---|---|
| `LINE` | `pt[10]` start, `pt[11]` end | |
| `CIRCLE` | `pt[10]` center, `real[40]` radius | |
| `ARC` | `pt[10]` center, `real[40]` radius, `angle[50]` start, `angle[51]` end | Angles in degrees CCW from X axis |
| `POINT` | `pt[10]` location, `angle[50]` display angle | |
| `TEXT` | `pt[10]` insertion point, `real[40]` height, `text[1]` string, `int[72]` horizontal justification, `int[73]` vertical justification | `int[72]`: 0=left (absent), 1=center, 2=right, 3=aligned, 4=middle, 5=fit. `int[73]`: 0=baseline (absent), 1=bottom, 2=middle, 3=top. **Default (left/baseline):** set `pt[10]` to the insertion point; omit `pt[11]` and both justification flags. **Any other justification:** set `pt[11]` to the anchor point (the center, right edge, top, etc. — wherever the text should align to); omit `pt[10]`. CSVIN derives `pt[10]` from `pt[11]` automatically before calling `entmake`. The `pt[10]` values seen in CSVOUT output are AutoCAD's cached fast-render positions — do not attempt to compute or reproduce them. |
| `MTEXT` | `pt[10]` insertion point, `real[40]` text height, `real[41]` defined width (reference rectangle width, 0=undefined/no wrap), `text[1]` content, `int[71]` attachment point, `angle[50]` rotation | `\P` = newline in content. Other inline codes (`\A1;`, `{\H...}`, etc.) preserved as-is. `real[44]` = line spacing factor (1.0=single, absent=default/none set). `real[46]` = defined column height (0=variable/none — legacy XDATA-era field, distinct from any column geometry on the now-stripped embedded object). `int[72]` = drawing direction (1=LR, 3=TB, 5=by style). `int[73]` = line spacing style (1=at least, 2=exactly). `long[90]` = background fill flag (1/3/16/17 = filled, with trailing `int[63]` fill color and `real[45]` fill scale factor; 2 = use drawing background, `long[90]` only — no trailing fields; absent/0 = no fill). `long[421]` = RGB background fill color override — same relationship to `int[63]` as `long[420]` has to `color[62]`: `int[63]` is always present when fill is active, `long[421]` is present only when the fill color is a true-color RGB value and absent when it is an ACI; overrides `int[63]` for display when present. `real[45]` is **only** present as part of the fill tail — it is not a general-purpose defined-height field. DXF code 101 (`Embedded Object`) and the codes unique to that block (`pt[11]`, `real[42]`, `real[43]`, `int[70]`, `int[74]`) are stripped — `entmake` does not support them. The remaining shared codes (`pt[10]`, `real[41]`, `real[44]`, `int[71]`, `int[72]`, `int[73]`) are emitted once, flattened — not duplicated for the main entity vs. the stripped embedded object. |
| `INSERT` | `pt[10]` position, `name[2]` block name, `real[41/42/43]` scale, `angle[50]` rotation | Absent scale = 1.0. If no matching BLOCK definition exists, the block was stripped — treat as zero-geometry insertion |
| `MINSERT` | same as INSERT plus `real[44]` column spacing, `real[45]` row spacing, `int[70]` column count, `int[71]` row count | Rectangular array of block insertions. **Exported as `INSERT` type** — there is no separate MINSERT entity type in the CSV. Distinguish from a plain INSERT by the presence of `real[44]`/`real[45]`. `int[70]` and `int[71]` carry column and row counts in this context — distinct from their flag meanings on other entity types |
| `ATTRIB` | follows INSERT, `name[2]` tag, `text[1]` value | Always same space (model or paper) as parent INSERT |
| `ATTDEF` | inside BLOCK definition, `name[2]` tag | |
| `3DFACE` | `pt[10]`-`pt[13]` four corners | `int[70]` = edge visibility bitmask (bit N hides edge N) |
| `SOLID` | `pt[10]`-`pt[13]` four corners | Note: pt[12] and pt[13] are swapped vs 3DFACE in DXF spec |
| `TRACE` | `pt[10]`-`pt[13]` four corners | Same geometry as SOLID, legacy entity |
| `POLYLINE` | `pt[10]` elevation only (X and Y always zero), followed by VERTEX rows, terminated by SEQEND | `int[70]` flags: 1=closed-M, 2=curve-fit, 4=spline-fit, 8=3D, 16=polygon-mesh, 32=closed-N, 64=closed-N (polygon mesh). Flags combine — `int[70]=17` is polygon-mesh + closed-M. `int[70]=128` (continuous linetype pattern) applies to 2D polylines only — ignored on 3D polylines (`int[70]=8`). `seq[66]=1` always present. `real[40]` default start width, `real[41]` default end width — absent when zero. `int[75]` smooth surface type: 0=none, 5=quadratic B-spline, 6=cubic B-spline, 8=Bezier. **Polygon mesh:** when `int[70]` has bit 16 set, `int[71]`=M vertex count and `int[72]`=N vertex count are required — must be present for DXF export. See sample_polylines.csv. |
| `VERTEX` | `pt[10]` coordinate, `real[40]` start width, `real[41]` end width, `real[42]` bulge | `int[70]` vertex flags: 1=curve-fit generated vertex, 2=tangent defined (`angle[50]` carries tangent direction), 8=spline frame control point, 16=spline fit point, 18=spline fit point with tangent (16+2 — used for spline-fit insert/anchor vertex), 32=3D polyline vertex OR 3D polygon mesh vertex, 64=polygon mesh vertex closed-N, 128=face index vertex (`int[71/72/73]` are vertex indices). **Note:** `int[70]=4` on a VERTEX is not valid DXF — never emit it. `real[40]`/`real[41]` absent when they match the POLYLINE default widths. **Curve-fit POLYLINEs (`int[70]=2` on POLYLINE):** CSVOUT strictly interleaves original control vertices (int[70]=2, with `angle[50]` tangent) and generated curve-fit vertices (int[70]=1, with `real[42]` bulge). Last original vertex has no bulge. For AI generation, emit only the original control vertices (int[70]=2) — AutoCAD regenerates the fit. **Spline-fit POLYLINEs (`int[70]=4` on POLYLINE, `int[75]=6` for cubic):** vertex order is strict: (1) insert/anchor vertex (int[70]=18, coordinates of first fit point, carries `angle[50]` tangent), (2) all generated B-spline control vertices (int[70]=8), (3) all fit-point vertices (int[70]=16). For AI generation use SPLINESEGS=2: for N fit points generate `2×(N-1)+1` control points (int[70]=8), followed by N fit points (int[70]=16). Example for 5 fit points: `insert(18) → ctrl×9(8) → fit×5(16) → SEQEND`. See `sample_polylines.csv` for complete verified examples of all POLYLINE/VERTEX flag combinations. |
| `LWPOLYLINE` | `pt[10]` all vertices space-delimited, `real[42]` bulge values comma-delimited | Post-R12, treated as R12-extended. `elev[38]` = elevation when non-zero. `thick[39]` = thickness when non-zero. `int[70]` = 1 when closed, absent when open |
| `MESH` | `pt[10]` vertices space-delimited, `long[90]` face and edge data, `long[93]` face data count, `long[94]` crease edge count, `real[140]` crease values comma-delimited | Subdivision mesh (post-2010). `long[91]` = subdivision level (absent when 0). `long[90]` contains two sections combined: face section (`long[93]` values — face vertex counts followed by vertex indices) then edge section (`long[94]×2` values — vertex index pairs per crease edge). Total `long[90]` field count = `long[93]` + `long[94]×2`. A fixed `(90 . 0)` sentinel is always appended by CSVIN after the edge section — it is not a CSV value. `int[92]` (vertex count) and `int[95]` (crease count, always equal to `int[94]`) are **absent from the CSV** — derived by CSVIN from `pt[10]` field count and `long[94]` respectively. `real[140]` absent when all creases are 0.0 — omit for flat meshes. When present, field count must equal `long[94]`. |
| `MLINE` | `name[2]` style name, `pt[10]` start point, `pt[11]` vertex coordinates space-delimited, `pt[12]` segment direction unit vectors space-delimited, `pt[13]` miter direction unit vectors space-delimited, `real[40]` overall scale, `real[41]` element parameters comma-delimited | Multiline entity — parallel lines drawn along a path using an MLINESTYLE definition. `name[2]` = MLINESTYLE name (style geometry is embedded in the export rather than referencing external style data). `pt[10]` = origin/start point (DXF code 10, always equals first point in `pt[11]`). `pt[11]` = all vertices of the path, space-delimited (DXF code 11, one per vertex). `pt[12]` = segment direction unit vectors, space-delimited, one per vertex (DXF code 12). `pt[13]` = miter direction unit vectors, space-delimited, one per vertex (DXF code 13). `real[40]` = overall scale factor applied to all element offsets. `real[41]` = element parameter flat list, comma-delimited (DXF code 41). For a 2-element style with N vertices, each vertex contributes 2 × (offset, trailing_gap) parameter groups; for non-uniform element parameter counts see `int[74]`. `int[70]` = flags: bit 0=closed, bit 1=suppress start caps, bit 2=suppress end caps (absent=open with caps). `int[74]` = per-element parameter counts comma-delimited (DXF code 74) — absent when all elements have exactly 2 parameters each at all vertices. CSVIN support: imports against the `Standard` MLINESTYLE — the named style must exist in the target drawing or CSVIN substitutes Standard. |
| `SEQEND` | terminates POLYLINE/INSERT+ATTRIB sequence | No properties — type column only |
| `DIMENSION` | `pt[10]` dim line point, `pt[11]` text midpoint, `pt[13]` definition pt 1, `pt[14]` definition pt 2, `pt[15]` definition pt 3 (arc/angular), `pt[16]` arc point (angular), `real[40]` leader length (certain types), `real[144]` actual measurement value (derived, read-only), `text[3]` dimstyle name, `angle[50]` rotation angle | `int[70]` low 4 bits = dimension type: 0=linear-rotated, 1=aligned, 2=angular, 3=diameter, 4=radius, 5=angular-3pt, 6=ordinate. Bit 32 = unique block reference — always set in practice; default value of `int[70]` when absent is 32. Bit 64 = ordinate type. Bit 128 = text user-positioned. Common values: 32=linear, 33=aligned, 34=angular, 35=radius, 36=ordinate, 37=angular-3pt. `name[2]` references a `*D##` anonymous block — stripped by default, no matching BLOCK definition will be present. `char[271]`/`char[272]` may appear as per-entity style overrides (decimal places). `real[144]` is the AutoCAD-computed measurement — present but not required for CSVIN import. See Block strip rules. |
| `SPLINE` | `pt[10]` control points space-delimited, `pt[11]` fit points space-delimited, `pt[12]` start tangent vector (optional), `pt[13]` end tangent vector (optional), `real[40]` knot vector comma-delimited, `real[41]` weights comma-delimited (rational splines only), `real[42]` knot tolerance, `real[43]` control point tolerance, `real[44]` fit tolerance (present only when fit points exist) | `int[70]` flags (bits 0–4 only — upper bits stripped by CSVOUT): 1=closed, 2=periodic, 4=rational, 8=planar, 16=linear. AI generators should only set bits 0–4. `int[71]` = degree. `int[72]` (knot count), `int[73]` (control point count), `int[74]` (fit point count) are **absent from the CSV** — they are derived by CSVIN from field counts of `real[40]`, `pt[10]`, `pt[11]` respectively. Do not emit them. `pt[12]` and `pt[13]` are **independently optional** — either may be present without the other, and both are absent in the column header unless at least one spline in the drawing has that tangent. `real[44]` present only when `int[74] > 0`. All data packed inline — no continuation rows. |
| `ELLIPSE` | `pt[10]` center, `pt[11]` major axis vector, `real[40]` minor/major axis ratio, `real[42]` end parametric angle | End angle in radians: 2π = full ellipse. `pt[11]` magnitude = major axis length, direction = major axis orientation. `real[40]` is the ratio of minor to major axis (0 to 1), not an absolute length. |
| `HATCH` | `name[2]` pattern name, `pt[10]` boundary loop vertices (space-delimited `x,y` pairs per loop, loops separated by `0.0,0.0,0.0` sentinel), `pt[11]` spline control points for curved boundary edges, `real[40]` circle boundary radius (comma-delimited when multiple circle boundaries), `real[41]` pattern scale, `real[42]` boundary edge bulge values (comma-delimited), `real[43]` pattern line X origins (comma-delimited per line), `real[44]` pattern line Y origins, `real[45]` pattern line delta-X, `real[46]` pattern line delta-Y, `real[49]` dash/gap lengths per pattern line (comma-delimited), `angle[50]` circle boundary start angle (comma-delimited when multiple circles), `angle[51]` circle boundary end angle — 360=full circle (comma-delimited when multiple circles), `angle[52]` overall hatch rotation in degrees, `angle[53]` per-pattern-line angles (comma-delimited), `int[70]` associativity (1=associative), `int[71]` dual/double hatch flag, `int[72]` edge types or polyline closed flag (see notes), `int[73]` complete circle flag (1=full circle when `int[72]=2`), `int[75]` hatch style (0=normal, 1=outer, 2=ignore), `int[76]` pattern type (0=user-defined, 1=predefined, 2=custom), `int[77]` island detection style, `int[78]` number of pattern lines, `int[79]` boundary loop edge type flags (comma-delimited), `long[91]` number of boundary loops, `long[92]` loop type flags (comma-delimited per loop), `long[93]` edge count per loop (comma-delimited), `long[98]` source boundary object count (absent when not associative), `trans[440]` transparency, `long[450]`–`long[453]` gradient fill data, `real[460]`–`real[462]` gradient angles/shift | `name[2]` is the pattern name — `ANSI31`, `SOLID`, `BRICK`, `_USER`, `_U` (user-defined short form), `FP_*` (custom floor plan patterns), or other named patterns. For `SOLID` fill: `int[76]=1`, no pattern line data. **`long[92]` loop type flags** (comma-delimited, one per loop): bit 0 (1)=external boundary, bit 1 (2)=polyline boundary, bit 2 (4)=outermost, bit 3 (8)=textbox island, bit 4 (16)=hole/island. Common values: 1=external edge-type, 5=external+polyline, 7=external+polyline+outermost, 17=external+circle-type, 24=island (bits 3+4). **`int[72]` has two roles** depending on `long[92]`: for polyline boundaries (long[92] bit 1 set), `int[72]` is the polyline closed flag (0=open, 1=closed) — one value per polyline loop, comma-delimited; for edge boundaries (long[92] bit 1 clear), `int[72]` is per-edge type (1=line, 2=arc/circle, 3=ellipse, 4=spline) — comma-delimited across all edges of all edge-type loops. Circle boundaries: `int[72]=2`, `real[40]`=radius, `angle[50]`=start, `angle[51]`=end (360=full circle), `int[73]=1` — when multiple circle boundaries exist, `real[40]`, `angle[50]`, and `angle[51]` are all comma-delimited. `angle[52]` is the overall hatch rotation and may be any value. Multiple boundary loops separated by `0.0,0.0,0.0` sentinel in `pt[10]`. All comma-delimited fields are parallel arrays indexed by pattern line or boundary edge. For polyline boundaries (`long[92]` bit 1 set): `pt[10]` = `origin(x,y,z)` + boundary vertices (2D) + trailing pair equal to `real[43]`,`real[44]`; `pt[11]` = same vertices starting from second vertex. **AI generation:** set `real[43]`/`real[44]` and the `pt[10]` trailing pair to `0.0` — the hatch draws correctly and AutoCAD recomputes the exact seed on regeneration. `real[45]`/`real[46]` are computable: `(-cos(angle)×spacing×scale, sin(angle)×spacing×scale)`. `int[72]` is required even for SOLID fill polyline boundaries (as the closed-polyline flag). **Gradient fill:** when `long[450]=1`, the hatch is a gradient fill. `long[452]` = 1 for one-color gradient, 2 for two-color. `long[453]` = number of gradient color entries. `long[421]` = packed RGB color(s) for gradient (comma-delimited when multiple). `real[462]` = shade/tint factor (0.0–1.0). `real[463]` = gradient taper values comma-delimited (one per color). `string[470]` = gradient type name (`LINEAR`, `HEMISPHERICAL`, `CURVED`, `INVLIN`, `INVSPH`, `INVCUR`, `SPHER`, `HEMI`, `CURV`). CSVIN handles group-47 (pixel size threshold) internally from flags — do not emit `real[47]`. |
| `RAY` | `pt[10]` origin, `pt[11]` unit direction vector | Extends infinitely in one direction from origin. `pt[11]` is a direction vector — need not be normalized. AutoCAD normalizes internally on creation |
| `XLINE` | `pt[10]` point on line, `pt[11]` unit direction vector | Extends infinitely in both directions (construction line). `pt[11]` is a direction vector — need not be normalized |
| `LEADER` | `pt[10]` vertices space-delimited, `real[40]` arrowhead height, `real[41]` annotation width, `text[3]` dimstyle name, `int[73]` annotation type | `int[73]`: 0=none/MTEXT follows, 1=tolerance, 2=block reference. `pt[213]` = normal vector override — present when the leader is not in the WCS XY plane |
| `TOLERANCE` | `pt[10]` insertion point, `text[1]` tolerance string, `text[3]` dimstyle name | Tolerance string uses `%%v` control codes for geometric tolerance symbols and `^J` as line separator |
| `ACAD_TABLE` | `pt[10]` insertion point, `pt[11]` direction vector, `long[91]` nROW (row count — field[0] only for simple tables), `long[92]` nCOL (column count), `real[141]` row heights comma-delimited (nROW values), `real[142]` column widths comma-delimited (nCOL values), `string[302]` per-cell text (ncell tab-delimited fields including empty strings), `text[1]` unique non-empty cell text (tab-delimited, for AI readability), `long[90]` combined: field[0]=table flags (typically 22), field[1..N]=per-cell value flags (4=content, 0=empty), `long[93]` combined: field[0]=override count, field[1..N]=per-cell value type (6=content, 7=virtual/empty) | nROW × nCOL = total cell count (ncell). `name[2]` is never emitted — the anonymous `*T##` block is auto-assigned by CSVIN and by direct-to-DXF export. `string[302]` is the structural cell-content column: exactly ncell tab-delimited fields, empty string for empty/virtual cells. `text[1]` is redundant for CSVIN but present for AI readability. For a minimal plain table, only `long[91]` (field[0]=nROW), `long[92]`, `real[141]`, `real[142]`, `string[302]`, `long[90]`, and `long[93]` are required — per-cell style columns may be omitted entirely and CSVIN produces a functional default-styled table. Per-cell style overrides (`int[63]`, `int[64]`, `real[140]`, `long[421]`, `long[422]`, `real[145]`, etc.) are exported as padded ncell comma-delimited columns — see ACAD_TABLE extended columns. Empty tables (nROW=0 or nCOL=0) are valid — only table-level columns present. `long[91]` field[0] is sufficient for AI generation of simple tables (no merged cells); the full ncell per-cell merged-row-span values are present in CSVOUT when cells are merged. |
| `MULTILEADER` | `string[304]` mtext content, `pt[10]` all points space-delimited (content base, last leader pt, vertices × nlines, color point), `pt[11]` normals space-delimited (mtext normal + dogleg direction; absent when both at default), `pt[12]` mtext location, `real[40]` combined: content scale + dogleg length, `long[90]` combined property override flags (3 fields), `long[91]` combined text-width flags (absent when all sentinel/default), `int[170]` combined leader type — 2 fields (absent when default `1,1`), `int[175]` combined text attachment — 2 fields (absent when default `1,0`), `int[179]` text angle type (absent when default), `bool[291]` has-dogleg flags — 3 fields (absent when default `0,1,1`), `bool[292]` has-text-direction flags — 2 fields (absent when default `0,0`), `real[141]` text height scale (absent when 0.0), `int[95]` arrow style (absent when default 1) | `pt[10]` field count n10 drives geometry: n10=0 → leaderless/compact; n10≥4 → nlines = n10−3 (field[0]=content base, field[1]=last leader pt, field[2..N-2]=per-line vertices, field[N-1]=color point `1.0,1.0,1.0`). All combined columns have fixed field counts from the DXF sequence structure, not from nlines. `long[90]` field[0] is always `-1073741824` (sentinel), field[1] is always `0`, field[2] is the property override bitmask: `279552`=straight, `279553`=spline, `312704`=four-leader. `long[91]` standard: 3 fields (`CONTEXT_DATA`, `LEADER_index`, `common`); four-leader: 6 fields. `string[304]` absent when leaderless. For spline leader: set `long[90]` field[2] to `279553` and add `int[170]: 1,2`. For right-aligned text: add `int[175]: 1,2` and `int[179]: 3`. For multiple leader lines: add more vertex points to `pt[10]` between last-leader-pt and color-point. |
| `PDFUNDERLAY` | `pt[10]` insertion, `real[41/42/43]` uniform scale, `char[281]` contrast (0–100), `char[282]` fade (0–80) | Undocumented entity — PDF reference underlay attached to the drawing. Not supported by CSVIN. Non-geometric properties (layer, linetype) preserved for analysis. `char[281]`/`char[282]` columns will be absent when only PDFUNDERLAY uses them — suppressed per single-entity-type column policy. |
| `BLOCK` | `name[2]` block name, `pt[10]` base point | Anonymous (`*`-prefix) and empty blocks are stripped by default. See Block structure section |
| `ENDBLK` | closes BLOCK definition | No properties — type column only |
| `LAYER` | `name[2]` name, `linetype[6]`, `color[62]`, `int[70]` flags | `int[70]`: 1=frozen, 2=frozen-by-default, 4=locked. Reserved layer names: `_ai` and `_claude` are reserved for AI directives — CSVIN ignores all entities on these layers. `Defpoints` is AutoCAD's dimension point layer — suppress for visualization. |
| `LTYPE` | `name[2]` name, `text[3]` description, `real[40]` total pattern length, `real[49]` element lengths comma-delimited, `int[70]` flags, `int[72]` alignment (always 65=ASCII 'A'), `int[73]` element count | `text[3]` is the human-readable description shown in the linetype dialog. `real[49]` absent = CONTINUOUS (no elements). Positive element = dash, negative = gap, zero = dot. `ByBlock` and `ByLayer` are AutoCAD built-in linetypes — they appear as name-only rows when the LTYPE table is exported. |
| `STYLE` | `name[2]` name, `text[3]` font/shape filename (when differs from name), `real[40]` fixed height (0=variable), `real[41]` width factor, `real[42]` oblique angle, `angle[50]` last-used height, `int[70]` flags, `int[71]` text generation flags | `int[70]` bit 1 = shape file, bit 4 = vertical. `int[71]` bit 2 = mirror X, bit 4 = mirror Y. A STYLE row with empty `name[2]` is the anonymous shape file style AutoCAD creates for linetype shape references — `text[3]` will be a `.shx` filename (e.g. `ltypeshp.shx`). |
| `DIMSTYLE` | `name[2]` name, `real[40]`–`real[48]` dimension variables, `int[70]`–`int[78]` dimension flags, `real[140]`–`real[147]` extended reals, `int[170]`–`int[178]` extended flags | Full R12 dimvar set. See DXF R12 spec for individual variable meanings (DIMSCALE=40, DIMASZ=41, DIMEXO=42, etc.). A DIMSTYLE row with only `name[2]` is valid — all other columns absent means use AutoCAD defaults for that style |
| `SECTION` / `ENDSEC` | `name[2]` section name | Present only when the full section is included via `-dxs`. Absent when individual tables or blocks are selected by name. See Structural sentinel rows section |

---

## Typical analysis workflow for AI

1. Read the trailing metadata lines after the blank separator. Parse `#DXF-CSV v1.0` fixed clauses and `#DXF-CSV-cond` conditional clauses by splitting on ` | `. If a `#zombies:` line is present, split on `:` then spaces to build a zombie type set.
2. Load header row to get the column set for this file. Not all columns are present in all files — columns reflect only the group codes present in this export.
3. Filter structural rows: `type` in (`SECTION`, `ENDSEC`, `BLOCK`, `ENDBLK`, `SEQEND`) — keep for block resolution but exclude from geometry analysis.
4. Load LAYER table rows to build a layer→color and layer→linetype map.
5. For geometry work: resolve INSERT rows by finding matching BLOCK definitions and transforming their entity coordinates. INSERT rows with no matching BLOCK definition are valid — the block was stripped.
6. For visualization: suppress display-only layers (DEFPOINTS, NPLT-suffixed names, ASHADE, SCRN-suffixed names). AutoCAD system layers with `*ADSK_`-prefixed names (e.g. `*ADSK_SYSTEM_LIGHTS`) may also appear — these contain non-graphical objects and can be suppressed.
7. If a `#zombies:` line is present, build a set of zombie type names. Rows matching these types have no geometry — skip them for geometry analysis but retain them for layer/property inventory.
8. Z coordinates present = 3D drawing. Z absent on all entities = 2D plan. Mixed = 2.5D (flat entities at various elevations).
9. Check for TEXT entities on layer `_ai` or `_claude` — these are AI directives. The `pt[10]` coordinate is the anchor for any generated output. See AI directives section.

---

## AI directives

A TEXT or MTEXT entity on layer `_ai` is an AI directive — an instruction to an AI consumer rather than drafting geometry. CSVIN ignores all entities on `_ai`. The `pt[10]` coordinate is the anchor point for any generated output.

```
MTEXT, "palette: scan attached PDF for annotations, street names, elevation points, area tables. columns by type. topo sorted high to low. text height 30.", _ai, 3000,2100
```

TEXT is supported for short directives. MTEXT is preferred — it allows longer instructions without truncation and is more visible in AutoCAD. The directive text is plain English. There is no required syntax — write what you want the AI to do. The coordinate tells the AI where to place any generated content in the drawing coordinate space.

**Layer `_claude`** is also supported and carries the same meaning. Use `_claude` when the task requires Claude specifically — it signals to the user that the directive was authored for or by Claude, and that another AI may not produce equivalent results.

**Returned CSV — additive by default:** A CSV returned in response to a directive should contain only the entities the AI added or modified — not a copy of the source drawing. This keeps generated files small and makes import clean with no risk of overwriting existing geometry. If the task requires modifying existing entities, the user should say so explicitly in the directive.

**Required columns:** Always include `name[2]` in the header. Every DXF-CSV file contains SECTION, LAYER, and LTYPE rows — all require `name[2]`. A file missing `name[2]` from the header will have nameless structural rows and will fail to import.

**Column role disambiguation — never interchange these three:**
- `text[1]` carries the display string for TEXT and MTEXT entities, the value for ATTRIB entities, and override text for DIMENSION entities. It has no meaning on any other row type.
- `name[2]` carries the symbolic name for table entries (LAYER, LTYPE, STYLE, DIMSTYLE, SECTION) and for INSERT block references and HATCH pattern references. It is never a display string and never a layer assignment.
- `layer[8]` carries the layer assignment for all geometry entities. It is always a layer name string. It is never a display string and never a symbolic name for anything other than the layer the entity lives on.

A TEXT entity has `text[1]` (what it says) and `layer[8]` (which layer it's on) and no `name[2]`. A LAYER row has `name[2]` (the layer name) and no `text[1]` or `layer[8]`. An INSERT has `name[2]` (block name), `layer[8]` (layer it lives on), and no `text[1]`. A HATCH has `name[2]` (pattern name like `ANSI31`) and `layer[8]` (layer it lives on) and no `text[1]`.

**Column minimization:** Include only columns needed by the entities actually present in the file. Unused columns — where every row has an empty value — add width without value and increase the risk of row misalignment. If no entity in the file uses `pt[12]`, omit `pt[12]` from the header entirely.

**Row alignment:** Every row must emit exactly as many comma-separated fields as the header row. Empty cells are never omitted — a row with 15 header columns must always produce 15 fields, using empty strings for unused positions. A single short row will shift all subsequent columns and corrupt the import.

**Text height and style:** Use the directive entity's own `real[40]` (text height) as the reference size for any generated text content. If `style[7]` is present on the directive, use the same style. This ensures generated text reads at the correct scale for the drawing without requiring a follow-up correction.

**Scope:** AI directives work best for content that can be extracted from text — annotation palettes, area tables, label sets, layer organization. Geometry tracing requires a human with an underlay.

### Filled geometry — LWPOLYLINE and SOLID

For simple filled shapes, LWPOLYLINE with per-vertex width is preferred over HATCH. HATCH is powerful but requires boundary loops, island logic, and pattern parameters — LWPOLYLINE fill requires only geometry the AI already knows.

**Filled rectangle:** Two-vertex closed LWPOLYLINE, horizontal segment at the vertical center of the rectangle, `real[40]` and `real[41]` both set to the rectangle height on the first vertex and `0.0` on the second. The segment draws filled at the width specified by the first vertex. `int[70]=1` (closed).
```
LWPOLYLINE, layer, "x0,y_center x1,y_center", real[40]="h,0.0", real[41]="h,0.0", int[70]=1
```

**Filled disc (donut):** Two-vertex closed LWPOLYLINE with `real[42]=1.0,1.0` (bulge = full semicircle on each segment) and `real[43]` = diameter as constant width. Place vertices at left and right of the diameter. Both `real[40]` and `real[41]` should match `real[43]` on both vertices when using per-vertex width instead of constant width — mismatched second-vertex width produces an unfilled arc.
```
LWPOLYLINE, layer, "cx-r,cy cx+r,cy", real[42]="1.0,1.0", real[43]=diameter, int[70]=1
```

**Tapered triangle:** Single open segment from apex to base midpoint, `real[40]` = full height at apex vertex, `0.0` at base vertex — produces a filled triangle pointing toward the apex. `int[70]=0` (open — closing a tapered segment creates a spike artifact).

**Variable-width shapes (diamonds, chevrons):** POLYLINE with per-vertex `real[40]` (start width) and `real[41]` (end width) on each VERTEX row. Each segment draws at the width specified by its start vertex, transitioning to the end width. `seq[66]=1` required on the POLYLINE row. SEQEND terminates the sequence.

**SOLID entity:** Four-corner filled quadrilateral — good for simple rectangles and trapezoids. Corner order is non-intuitive: `pt[10]`=BL, `pt[11]`=BR, `pt[12]`=TL, `pt[13]`=TR (not sequential around the perimeter — `pt[12]` and `pt[13]` are swapped relative to 3DFACE). Incorrect order produces two triangles instead of a filled quad.

**Draw order:** Entities render in CSV row order within a block or model space section — later rows draw on top of earlier rows. Place background fills first, then overlapping geometry, then text. This is the only draw-order control available without issuing AutoCAD's DRAWORDER command after import.

**Color choices for fills:**
- ACI 7 (white/black) renders as black on white paper and white on a dark background — display-dependent. Avoid for fills where background color matters.
- ACI 250 is near-black regardless of background — use instead of ACI 8 (dark grey) or ACI 7 when black fill is intended.
- `long[420]=16711422` (RGB 254,254,254) renders as near-white on any background. Preferable to ACI 7 when a white fill must stay white on paper.
- `trans[440]` transparency allows layering — a semi-transparent fill over geometry lets both show through. Common value: `33554559` ≈ 50% transparent.

Three reference files are published alongside this spec at `https://drawingsync.com/dxfcsv/v1.0/`:

**`sample_entities.csv`** — `https://drawingsync.com/dxfcsv/v1.0/sample_entities.csv` — one correct row per supported entity type. Use as an encoding reference when reading or generating entity rows. Covers LINE, CIRCLE, ARC, POINT, TEXT, MTEXT, LWPOLYLINE, POLYLINE/VERTEX, SPLINE, ELLIPSE, 3DFACE, SOLID, INSERT, MESH, HELIX, and others. Each row uses realistic values with all required columns populated.

**`sample_tables.csv`** — `https://drawingsync.com/dxfcsv/v1.0/sample_tables.csv` — fully populated LAYER, LTYPE, and STYLE table sections. Use as a reference for table row structure, column usage, and realistic values. Includes named linetypes (CENTER, HIDDEN, DASHED, PHANTOM), layers with lineweight and color, and text styles with shape file references.

**`sample_ai_bracket.csv`** — `https://drawingsync.com/dxfcsv/v1.0/sample_ai_bracket.csv` — a complete minimal mechanical drawing authored by an AI consumer following this spec. Demonstrates correct AI authoring patterns: layer definitions before entity rows, correct column usage, proper default suppression, realistic geometry. `sha1:396cb2c5a30e` identifies the source as an empty AutoCAD 2018 template — the standard sha1 for AI-generated content not derived from an existing drawing.

**`sample_ai_electrical.csv`** — `https://drawingsync.com/dxfcsv/v1.0/sample_ai_electrical.csv` — a complete schematic drawing: 208V three-phase delta water heater with three heating zones, thermostats, contactors, and ground symbol. Demonstrates electrical/schematic layer vocabulary (`power`, `bus`, `control`, `elements`, `labels`, `ground`), DASHED linetype for routed conductors, centered TEXT justification for component labels, and LWPOLYLINE for component boxes. Also includes `_ai` layer test entities demonstrating CIRCLE, open and closed LWPOLYLINE, DASHED LINE, and centered TEXT. Use alongside `sample_ai_bracket.csv` for schematic domain authoring.

**`sample_polylines.csv`** — `https://drawingsync.com/dxfcsv/v1.0/sample_polylines.csv` — verified CSVOUT output covering all POLYLINE/VERTEX flag combinations not present in typical drawings: real-world 20×7 polygon mesh (`int[70]=17`, `int[71]=20`, `int[72]=7`), closed-M/N mesh (`int[70]=48`), polyface mesh (`int[70]=64`), 3D polyline (`int[70]=8`) with `int[70]=32` vertices, spline-fit (`int[70]=4`) with correct insert(18)→ctrl(8)→fit(16) ordering, and curve-fit (`int[70]=2`) with interleaved tangent vertices. Five `_ai` layer notes cover: vertex ordering rules, PLINETYPE/LWPOLYLINE conversion, M×N vertex count requirement, int[70]=128 flag collision between POLYLINE and LWPOLYLINE, and guidance to prefer LWPOLYLINE for simple 2D work. Use as the reference for any POLYLINE generation — these sequences are not easy to produce correctly from the spec alone.

**`sample_mtext.csv`** — `https://drawingsync.com/dxfcsv/v1.0/sample_mtext.csv` — a complete periodic table of the elements, built almost entirely from multi-line MTEXT cells. Demonstrates background fill (`long[90]`, `int[63]`, `real[45]`), defined column height (`real[46]`), tightened line spacing (`real[44]=0.35`) for fitting three-line content in a fixed-height cell, middle-center attachment (`int[71]=5`), and `\W` width-factor scaling for long element names that would otherwise overflow a narrow cell. Five `_ai` layer TEXT notes cover: the `long[90]` fill-tail structural exception (value 2 vs. 1/3/16/17, see the `real[45]` finding in the 2026-06-19 changelog entry), the distinction between `^J` soft return (stays inside the current paragraph, responds cleanly to `\H` height scaling) and `\P` paragraph break (governed by AutoCAD's looser paragraph spacing model) — `^J` is what makes a tight fixed-height multi-line cell work, cell geometry conventions (10×10 unit cells, center-point insertion), and the `\W` graceful-compression pattern for overflow text. Use as the reference for any multi-line MTEXT content, background fill, or column-height work — the formatting interactions here are not obvious from the spec alone.

**`sample_csvout_reference.csv`** — `https://drawingsync.com/dxfcsv/v1.0/sample_csvout_reference.csv` — curated, hand-verified CSVOUT output covering structural variants of the most complex entity types: HATCH (19 examples covering FP_* floor-plan patterns, ESCHER, ANSI31, SOLID plain/gradient with one-color and two-color gradients, circle boundaries, islands, and mixed arc+line edge boundaries), MLINE (7 examples with `int[70]` flag variants), SPLINE (12 examples covering open/closed, fit-point-only, rational, and start/end tangent combinations), POLYLINE (9 examples including polygon mesh and polyface), MESH (8 examples from minimal tetrahedra to a torus with subdivision and crease values), DIMENSION (11 examples per type), TEXT alignment (all `int[72]`/`int[73]` combinations including multi-script), ACAD_TABLE (structural variants including merged cells, per-cell style overrides, and empty tables), and MULTILEADER (7 variants: default, compact/leaderless, straight, spline, right-aligned, four-leader, text-width-override). `_ai` layer TEXT rows are embedded throughout explaining the generation rules inline. Use as the primary reference when encoding any of these entity types for CSVIN.

---

## DXF-CSV v2018

The `codes:dxf-2018` tag indicates the source DWG was exported from a modern AutoCAD (2004–2018+ format). The design principle is unchanged — bookkeeping is stripped, drafting semantics are preserved, post-R12 entities are treated as R12-extended primitives.

### New entities in dxf-2018 output

#### SPLINE
Rational B-spline curve. All data packed inline on a single row — no continuation rows.

| Column | Code | Meaning |
|---|---|---|
| `pt[10]` | 10 | Control points — space-delimited `x,y,z` values |
| `pt[11]` | 11 | Fit points — space-delimited `x,y,z` values (absent if no fit points) |
| `pt[12]` | 12 | Start tangent vector — optional, independently present or absent |
| `pt[13]` | 13 | End tangent vector — optional, independently present or absent |
| `real[40]` | 40 | Knot vector — comma-delimited values |
| `real[41]` | 41 | Weights — comma-delimited, one per control point (rational splines only, when `int[70]` bit 2 set) |
| `real[42]` | 42 | Knot tolerance |
| `real[43]` | 43 | Control point tolerance |
| `real[44]` | 44 | Fit tolerance — column absent when no fit points exist |
| `int[70]` | 70 | Flags (bits 0–4 only — upper bits stripped): 1=closed, 2=periodic, 4=rational, 8=planar, 16=linear. AI generators should only set bits 0–4 |
| `int[71]` | 71 | Degree |
| `int[72]` | 72 | **Absent from CSV** — knot count is derived by CSVIN from `real[40]` field count |
| `int[73]` | 73 | **Absent from CSV** — control point count is derived by CSVIN from `pt[10]` field count |
| `int[74]` | 74 | **Absent from CSV** — fit point count is derived by CSVIN from `pt[11]` field count |

#### MTEXT
Multiline text. `pt[10]` = insertion point (single coordinate). `real[40]` = text height (initial text height). `real[41]` = defined width (reference rectangle width for text wrap; 0 = undefined, no wrap). `text[1]` = complete content string — `\P` encodes a paragraph/newline break, other inline formatting codes (`\A1;` = alignment, `{\H...}` = height override, etc.) are preserved as-is. `int[71]` = attachment point (1-9, top-left to bottom-right). `angle[50]` = rotation. `style[7]` = text style.

Additional codes emitted when present: `real[44]` = line spacing factor (1.0 = single spacing, absent when not set). `real[46]` = defined column height — a legacy field independent of the column-layout embedded object, 0 or absent when not used. `int[72]` = drawing direction (1=left-to-right, 3=top-to-bottom, 5=by style). `int[73]` = line spacing style (1=at least, 2=exactly).

`long[90]` = background fill flag. Values 1, 3, 16, and 17 trigger a trailing fill tail in fixed order: `int[63]` fill color and `real[45]` fill scale factor (~1.0–3.0 typical, 1.5 default — not a height). `long[421]` may also appear in the fill tail when the fill color is a true-color RGB value — it is the exact RGB override (`(R<<16)|(G<<8)|B`) and follows the same pattern as `long[420]` vs `color[62]`: `int[63]` is always present when fill is active (ACI approximation), `long[421]` is present only when the color is true-color and absent when it is an ACI; `long[421]` overrides `int[63]` for display when both are present. DXF's underlying entmake sequence also reserves a transparency slot at this position, but AutoCAD has never implemented transparency for MTEXT background fill — that slot is not written and does not appear in the CSV. Value 2 ("use drawing background color") is a structural exception: only `long[90]` itself is present, and the fill tail is rejected by `entmake` if included — a CSVOUT row with `long[90]=2` must never carry `real[45]` or the other tail fields. `long[90]` absent or 0 means no background fill at all. `real[45]` should not be treated as a general defined-height field — it only exists within this fill tail.

MTEXT can carry column layout data two ways in DXF: as a packed XDATA value (`real[46]`, above) or as a DXF code 101 `Embedded Object` block — a second, self-contained set of MTEXT-like group codes (its own `10`, `11`, `40`, `41`, `42`, `43`, `71`, `72`, `44`, `45`, `73`, `74`, `46`) nested after the 101 marker. The embedded object form is not supported by `entmake` and is stripped on export entirely. Codes that exist only inside the embedded object (`pt[11]` as a second point, `real[42]` actual height, `real[43]` actual width, `int[70]` column flow direction, `int[74]` column count) are dropped along with it — these never appear in the CSV. Codes that the main entity and the embedded object both define (`pt[10]`, `real[41]`, `real[44]`, `int[71]`, `int[72]`, `int[73]`) are unaffected — they come from the main entity and continue to be emitted normally as a single value, not duplicated. `real[45]` is **not** in this shared list — on the main entity it only appears inside the background fill tail described above, unlike the other listed codes which are always present once their condition is met.

MTEXT content longer than 250 characters is split across multiple DXF code 3 continuation lines, with the final remainder in code 1. CSVOUT flattens this into two related columns: `text[1]` holds the complete merged content with the code-1 remainder placed **first**, followed by each code-3 chunk in order, tab-separated — this is the opposite of DXF wire order, where code-3 chunks come first and code 1 is the trailing remainder. `text[3]` holds only the code-3 chunks, tab-separated, in DXF order, and is absent when the content fits in a single code-1 value (250 characters or fewer). A consumer reconstructing `entmake` input for content over 250 characters needs `text[3]` to know where the chunk boundaries fall — `text[1]` alone gives the correct content but not the correct node split.

#### ELLIPSE
Elliptical curve. `pt[10]` = center, `pt[11]` = major axis vector (magnitude = major axis length, direction = major axis orientation), `real[40]` = minor/major axis ratio (0 to 1), `real[42]` = end parametric angle in radians (2π = full ellipse). Start angle is always 0 in the current encoding — a future revision may move to `angle[50]`/`angle[51]` in degrees to align with ARC.

#### MULTILEADER
Multileader annotation (mleader). Confirmed support: straight, spline, right-aligned, four-leader, text-width-override, and leaderless/compact variants. Block-content multileaders are not supported — block geometry is lost on export and the entity round-trips as leaderless.

**`pt[10]` geometry encoding:** All leader points packed space-delimited in a fixed sequence: field[0]=content base point, field[1]=last leader point, field[2..N-2]=per-line vertices (one per leader line), field[N-1]=color point (always `1.0,1.0,1.0`). n10=0 means leaderless/compact. nlines = n10−3 when n10≥4. For multiple leader lines, additional vertex points are inserted before the color point.

**`long[90]` combined property flags:** Always 3 fields: field[0]=CONTEXT_DATA sentinel (always `-1073741824`), field[1]=LEADER section (always `0`), field[2]=common section override bitmask. Field[2] values: `279552`=straight leader, `279553`=spline leader (bit 0 set), `312704`=four-leader lines. Absent when leaderless.

**`long[91]` combined text-width flags:** Standard (3 fields): CONTEXT_DATA, LEADER_index (0), common. Four-leader (6 fields): CONTEXT_DATA, index0, index1, index2, index3, common. Sentinel values: `-1073741824` (`0xC0000000`) and `-1056964608` (`0xC1000000`) signal "inherit from style" — treat as opaque defaults. Absent when all fields at sentinel/default.

**Combined columns with fixed field counts:** `int[170]` (leader type: 1=straight, 2=spline — 2 fields), `int[175]` (text attachment — 2 fields), `bool[291]` (has-dogleg — 3 fields), `bool[292]` (has-text-direction — 2 fields). Each absent when all fields at default. Default suppression: `int[170]` absent when `1,1`; `int[175]` absent when `1,0`; `bool[291]` absent when `0,1,1`; `bool[292]` absent when `0,0`.

**Single-value columns absent when at default:** `pt[11]` absent when mtext normal=`0.0,0.0,1.0` and dogleg direction=`1.0,0.0,0.0`. `real[141]` absent when `0.0`. `int[95]` absent when `1` (default arrow style). `int[179]` absent when default.

**Minimum viable MULTILEADER (straight leader, one line):**
```
layer[8]:    target-layer
string[304]: label text
pt[10]:      content_x,content_y,0.0  lastpt_x,lastpt_y,0.0  vertex_x,vertex_y,0.0  1.0,1.0,1.0
pt[12]:      mtext_x,mtext_y,0.0
real[40]:    1.0,0.36
long[90]:    -1073741824,0,279552
```

#### HATCH
Hatch fill entity. `name[2]` = pattern name (e.g. `ANSI31`, `SOLID`, `BRICK`, `_USER`, custom `FP_*` names). Boundary loop vertices are packed space-delimited into `pt[10]`, with `0.0,0.0,0.0` as the sentinel separating loops when multiple loops exist. `real[41]` = pattern scale. `angle[52]` = overall hatch rotation. `angle[53]` = per-pattern-line angles comma-delimited. `real[43]`/`real[44]` = pattern line X/Y origins. `real[45]`/`real[46]` = pattern line delta-X/Y. `real[49]` = dash/gap lengths per pattern line. `int[72]` = boundary edge types per edge comma-delimited (1=line, 2=arc/circle, 3=ellipse, 4=spline). `int[75]` = hatch style (0=normal, 1=outer, 2=ignore). `int[76]` = pattern type (0=user, 1=predefined, 2=custom). `int[78]` = pattern line count. Circle boundaries: `int[72]=2`, `real[40]`=radius, `angle[50]`=start, `angle[51]`=end angle (360=full circle), `int[73]=1`. For SOLID fill: `int[76]=1`, no pattern line data. All comma-delimited fields are parallel arrays indexed by pattern line or boundary edge.

**`long[92]` loop type flags** (comma-delimited, one value per loop): bit 0 (1)=external boundary, bit 1 (2)=polyline boundary, bit 2 (4)=outermost, bit 3 (8)=textbox island, bit 4 (16)=hole/island. Common values: 1=external edge-type loop, 5=external+polyline, 7=external+polyline+outermost, 17=external+circle-type, 24=island. The presence of bit 1 in a loop's `long[92]` value determines how `int[72]` is interpreted for that loop.

**`int[72]` dual role:** For polyline boundaries (bit 1 of `long[92]` set), `int[72]` carries the polyline closed flag (0=open polyline, 1=closed polyline) — one value per polyline loop, comma-delimited. For edge boundaries (bit 1 clear), `int[72]` carries the per-edge type list (1=line, 2=arc/circle, 3=ellipse, 4=spline) — comma-delimited across all edges of all edge-type loops in sequence. `int[72]` is required for all HATCH entities regardless of fill type, including SOLID fill.

**`pt[10]` structure for polyline boundaries:** For polyline-type boundary loops (`long[92]` bit 1 set), `pt[10]` packs three things in sequence: the hatch entity origin as a 3D point (typically `0.0,0.0,0.0`), the N boundary vertices as 2D `x,y` pairs, and a trailing 2D `x,y` pair equal to `real[43]`,`real[44]` (the pattern-line origin). `pt[11]` holds the same N boundary vertices again starting from the second vertex. Both are required for polyline boundaries.

**Pattern-line-origin for AI generation:** `real[43]`/`real[44]` (pattern line X/Y seed origin) are computed by AutoCAD's internal hatch seeding algorithm and cannot be reproduced analytically. Set both to `0.0` for AI-generated hatches — the hatch will draw correctly with a slightly phase-shifted pattern, and AutoCAD recomputes the exact values on regeneration. Set the `pt[10]` trailing pair to `0.0,0.0` to match. `real[45]`/`real[46]` (pattern line delta-X/Y) are deterministically computable: `delta = (-cos(angle) × spacing × scale, sin(angle) × spacing × scale)` where spacing is the pattern's base line spacing (ANSI31 = 0.125).

**Group 47:** CSVIN computes and inserts `real[47]` (pixel size threshold) internally based on boundary flags — do not emit it. Its presence or absence is flag-driven and an incorrect value silently breaks `entmake`.

**Gradient fill:** when `long[450]=1`, the hatch is a gradient fill. Additional columns: `long[452]` = 1 for one-color gradient, 2 for two-color. `long[453]` = number of gradient color entries. `long[421]` = packed 24-bit RGB color(s) comma-delimited (one per `long[453]` entry). `real[462]` = shade/tint factor (0.0–1.0, applies to one-color gradient). `real[463]` = gradient taper values comma-delimited (one per color entry). `string[470]` = gradient type name (`LINEAR`, `HEMISPHERICAL`, `CURVED`, `INVLIN`, `INVSPH`, `INVCUR`, `SPHER`, `HEMI`, `CURV`). For gradient hatches the `name[2]` pattern is always `SOLID`.

#### IMAGE, 3DSOLID, LIGHT, EXTRUDEDSURFACE, OLE2FRAME
These entity types are not supported by CSVIN — `acdbEntMake` / `acdbEntMod` cannot create them, so they will be ignored on import regardless of what data is present. They may still appear in CSVOUT output and carry analysis value (layer, color, position). They frequently appear as zombies since their object enablers are typically not loaded during export, but the CSVIN limitation is independent of zombie status.

### dxf-2018 column additions

| Header | Code | Meaning | Entity |
|---|---|---|---|
| `real[141]` | 141 | Row heights comma-delimited (nROW values) | ACAD_TABLE |
| `real[142]` | 142 | Column widths comma-delimited (nCOL values) | ACAD_TABLE |
| `long[90]` | 90 | Combined: field[0]=table flags, field[1..N]=per-cell value flags (4=content, 0=empty) | ACAD_TABLE |
| `long[91]` | 91 | Combined: field[0]=nROW, field[1..N]=per-cell merged-row-span (262144=no merge) | ACAD_TABLE |
| `long[92]` | 92 | nCOL (single value) | ACAD_TABLE |
| `long[93]` | 93 | Combined: field[0]=override count, field[1..N]=per-cell value type (6=content, 7=virtual/empty) | ACAD_TABLE |
| `string[302]` | 302 | Per-cell text — ncell tab-delimited fields including empty strings | ACAD_TABLE |
| `long[421]` | 421 | Per-cell fill true-color (ncell comma-delimited, absent when -1 for all cells) | ACAD_TABLE |
| `long[422]` | 422 | Per-cell text true-color (ncell comma-delimited, absent when -1 for all cells) | ACAD_TABLE |
| `real[145]` | 145 | Per-cell rotation in radians (ncell comma-delimited, 0.0=no rotation) | ACAD_TABLE |
| `real[140]` | 140 | Combined table-level + per-cell text height — field count determines structure: 0=absent, 1=table-level only, ncell=per-cell only, ncell+1=both | ACAD_TABLE |
| `int[170]` | 170 | Per-cell alignment override (ncell comma-delimited, 0=no override) | ACAD_TABLE |
| `int[172]` | 172 | Per-cell flags — non-zero signals border overrides active (ncell comma-delimited) | ACAD_TABLE |
| `int[63]` | 63 | Per-cell fill ACI color override (ncell comma-delimited, 0=no override) | ACAD_TABLE |
| `int[64]` | 64 | Per-cell text color override (ncell comma-delimited, 0=no override) | ACAD_TABLE |
| `string[7]` | 7 | Per-cell text style name override (ncell comma-delimited, empty=no override) | ACAD_TABLE |
| `long[90]` | 90 | Face and edge data combined — face section then edge section | MESH |
| `long[91]` | 91 | Subdivision level (absent when 0) | MESH |
| `long[93]` | 93 | Face data count | MESH |
| `long[94]` | 94 | Crease edge count | MESH |
| `real[140]` | 140 | Crease values comma-delimited (absent when all 0.0) | MESH |
| `real[41]` | 41 | Defined width (reference rectangle width, 0=undefined/no wrap) | MTEXT |
| `real[44]` | 44 | Line spacing factor (1.0=single, absent=not set) | MTEXT |
| `real[45]` | 45 | Background fill scale factor (~1.0–3.0, 1.5 typical) — only present within the fill tail, see `long[90]` | MTEXT |
| `real[46]` | 46 | Defined column height — legacy field, 0/absent when not used | MTEXT |
| `long[90]` | 90 | Background fill flag (1/3/16/17=filled with trailing tail, 2=drawing background with no tail, absent/0=no fill) | MTEXT |
| `int[72]` | 72 | Drawing direction (1=LR, 3=TB, 5=by style) | MTEXT |
| `int[73]` | 73 | Line spacing style (1=at least, 2=exactly) | MTEXT |
| `real[47]` | 47 | Pixel size threshold — pattern line minimum dash length for display. Present in CSVOUT; CSVIN computes it internally from boundary flags — do not emit in AI-generated files | HATCH |
| `long[421]` | 421 | Packed RGB color(s) comma-delimited (one per gradient color entry) | HATCH |
| `long[421]` | 421 | RGB background fill color override — `int[63]` always present (ACI), `long[421]` present only when fill color is true-color RGB, absent when ACI; overrides `int[63]` for display | MTEXT |
| `real[463]` | 463 | Gradient taper values comma-delimited (one per color entry) | HATCH |
| `string[470]` | 470 | Gradient type name | HATCH |
| `pt[213]` | 213 | Normal vector override | LEADER |
| `string[304]` | 304 | Mtext text content (absent when leaderless) | MULTILEADER |
| `int[95]` | 95 | Arrow style index (absent when default 1) | MULTILEADER |
| `int[170]` | 170 | Combined leader type — 2 fields: 1=straight, 2=spline (absent when `1,1`) | MULTILEADER |
| `int[175]` | 175 | Combined text attachment — 2 fields (absent when `1,0`) | MULTILEADER |
| `int[179]` | 179 | Text angle type (absent when default) | MULTILEADER |
| `bool[291]` | 291 | Has-dogleg combined — 3 fields (absent when `0,1,1`) | MULTILEADER |
| `bool[292]` | 292 | Has-text-direction combined — 2 fields (absent when `0,0`) | MULTILEADER |
| `long[91]` | 91 | Combined text-width flags — 3 or 6 fields (absent when all sentinel/default) | MULTILEADER |

### Stripped dxf-2018 bookkeeping

The following group codes and sections are stripped in addition to the R12 strip list:

- Group 330 — persistent reactor owner handle
- Group 360 — extension dictionary handle  
- Group 102 — group code for `{ACAD_REACTORS` and `{ACAD_XDICTIONARY` blocks
- ACDSDATA section — BIM and material data
- THUMBNAILIMAGE section — embedded preview image
- All OBJECTS section entries except where semantic content is identified (e.g. IMAGEDEF path)

### dxf-2018 metadata additions

New conditional clauses for `codes:dxf-2018` files:

| Clause | Present when |
|---|---|
| `truecolor:rgb[420]` | Any entity uses true color (group 420) — packed 24-bit RGB integer overriding ACI color |

---

## Workflow decision guide

This section helps an AI reason about which workflow to recommend or generate for, based on the user's situation.

### The two import paths

**AutoCAD session (`CSVIN` command):** The user runs CSVIN inside a live AutoCAD session. `acdbEntMake` / `acdbEntMod` create or update entities directly in the open drawing. AutoCAD's geometry engine is live — computed entity properties are resolved immediately on creation. The target drawing must be open and its sha1 must match the `source:` clause.

**DXF file (`dwgsync.exe -dsm`):** No AutoCAD session required. `dwgsync.exe` merges the CSV into a DXF template file and writes a new `.dxf` or `.dwg`. AutoCAD does not need to be installed for `.dxf` output. The sha1 in the CSV identifies which template to merge into — typically `sha1:781e2fb2654f` (`new.dxf`) for standalone output, or `sha1:396cb2c5a30e` for import into a blank AutoCAD drawing.

**Nudge:** After a `-dsm` DXF import, DIMENSION entities will have their definition points and leader geometry present but their rendered anonymous blocks (`*D##`) absent — these are stripped on export and cannot be regenerated without AutoCAD's dimension engine. The user runs the **Nudge** option of the CSVIN command (or `-nudge` flag of `dwgsync.exe`) to open the DXF in AutoCAD and trigger regeneration. This is the most common post-import step for drawings with dimensions.

### Choosing a sha1 target

| Situation | sha1 target | `-dxs` scope | Notes |
|---|---|---|---|
| Generating from scratch, no source drawing | `396cb2c5a30e` (empty AutoCAD 2018 template) | `TABLES,BLOCKS,ENTITIES` | Define all layers, linetypes, styles, and blocks used |
| Generating for `new.dxf` standalone output | `781e2fb2654f` | `TABLES(LAYER),ENTITIES` | Minimal template — Standard style, layer 0, basic linetypes already present |
| Adding geometry to an existing drawing | Source drawing sha1 | `ENTITIES` (or `TABLES(LAYER),ENTITIES`) | Layers and blocks already exist in the drawing — reference by name, no need to redefine |
| Targeting a company or domain template | Company template sha1 | `TABLES(LAYER),ENTITIES` | Company layers, blocks, dimstyles already present — AI can reference them by name without defining them |

### Layer handling

AI performs well managing layers in generative mode. The default export scope `TABLES(LAYER),ENTITIES` reflects this — the LAYER table is always included so the AI has the full layer inventory, while LTYPE, STYLE, DIMSTYLE, and BLOCKS are omitted unless needed. When targeting a company or domain sha1, the AI can reference existing layer names (e.g. `A-WALL`, `E-POWR`, `S-BEAM`) confidently without redefining them — the sha1 contract guarantees those layers exist in the target drawing.

When generating for `sha1:396cb2c5a30e` (blank template), the AI must define every layer it uses. Layer definitions before their first entity use is not required structurally — CSVIN creates missing layers on import — but including them makes the file self-documenting and gives the user visibility into the layer scheme.

### AutoCAD not required

A DXF-CSV file can be converted to a valid `.dxf` without AutoCAD installed:

```
dwgsync.exe new.dxf -dsm drawing.csv -dxf output.dxf
```

`new.dxf` is the Drawing Sync minimal DXF template (`sha1:781e2fb2654f`), available at `https://drawingsync.com/dxfcsv/v1.0/new.dxf`. The resulting DXF contains only drafting content — no AutoCAD plot settings bloat. Any DXF-compatible application can open it. A nudge pass in AutoCAD is needed only if the drawing contains DIMENSION entities.

---

## CSVIN status

CSVIN is the companion import pipeline — reads a DXF-CSV file, matches it against the original DWG via `source:` and `sha1:`, and updates changed entities directly via `acdbEntMake` / `acdbEntMod`. No DXF is generated or consumed during import.

**Not supported for CSVIN:** 3DSOLID, LIGHT, EXTRUDEDSURFACE, OLE2FRAME, IMAGE, VIEWPORT, PDFUNDERLAY — these entity types cannot be created or modified via `acdbEntMake` / `acdbEntMod`. This is a permanent limitation, not planned work. CSVIN will ignore rows of these types on import.

**Planned for CSVIN:** Paper space layout import. `paper[67]=1` (default layout, `*Paper_Space`) is the current target. `paper[67]=2` and above (additional layouts — `*Paper_Space0`, `*Paper_Space1`, ...) will be supported as the layout enumeration scheme is extended. VIEWPORT creation remains outside CSVIN scope (`acdbEntMake` limitation — permanent).

---

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-07-12 | `-dxs` section restructured: syntax rules restored, human-facing note added for file dialog All option and `-CSVOUT` dash command, AI guidance added for missing block definitions (empty BLOCKS section = re-export needed). Filled geometry best-practice section added to AI generation guidance: LWPOLYLINE filled rectangle, disc (donut), tapered triangle, variable-width POLYLINE shapes, SOLID corner order, draw order by row position, color choices (ACI 250 for black, rgb 254,254,254 for white, transparency). MULTILEADER fully documented and CSVIN status updated to supported. Quick-reference stub replaced with full entry covering all columns, `pt[10]` geometry encoding (n10 field count drives leaderless vs. leader, nlines=n10−3), `long[90]` combined flags (field[2] bitmask: 279552=straight, 279553=spline, 312704=four-leader), `long[91]` combined text-width flags (3 or 6 fields, sentinel values documented), combined column default suppression rules, and minimum viable generation example. New column prefix entries: `bool[291]`, `bool[292]`, `string[304]`, `int[95]`; `int[170]`–`int[178]` split to expose `int[170]`, `int[175]`, `int[179]` with MULTILEADER uses. Confirmed against 7 live variants; block-content variant unsupported. ACAD_TABLE CSVIN status updated to supported. `last-updated:` metadata clause documented as spec version marker — date matches CSVOUT binary build. `paper[67]` documented as enumerated layout index: 1=default layout (`*Paper_Space`/`Layout1`), 2=second (`*Paper_Space0`), etc. MINSERT clarified as INSERT type — no separate entity; `real[44]`/`real[45]` renamed from "On MINSERT" to "On INSERT (array)". `char[280]` on ATTRIB documented. MLINE CSVIN support documented: presumes Standard MLINESTYLE; removed from not-supported list. `plotst[380]` added to column prefix table: entity-level plot style index, integer enumerator, absent=BYLAYER. `sample_csvout_reference.csv` description updated to include ACAD_TABLE and MULTILEADER. `-dxs=*` raw dump mode documented: full DXF tabular export, no metadata line, not a conforming DXF-CSV v1.0 file. Spec pruning: removed `pt[17]`, `text[4]`, `variable[9]` stub entries; removed LWPOLYLINE, MLINE, WIPEOUT dxf-2018 prose subsections (redundant with quick reference); trimmed units table to practical entries. |
| 1.0 | 2026-06-29 | ACAD_TABLE fully documented and CSVIN status updated to supported. Quick-reference entry replaced: `real[141]`/`real[142]` swap corrected (141=row heights, 142=column widths), `name[2]` removed (auto-assigned by CSVIN — never emit), cell content model corrected (`string[302]` is the structural ncell tab-delimited column; `text[1]` carries unique non-empty values for AI readability; per-cell style overrides exported as padded ncell columns, not stripped). Combined column pattern documented: `long[90]` (table flags + per-cell value flags), `long[91]` (nROW + per-cell merged-row-span), `long[93]` (override count + per-cell value type). `long[92]` = nCOL (single value). AI generation minimum set identified: `long[91]` field[0]=nROW, `long[92]`, `real[141]`, `real[142]`, `string[302]`, `long[90]`, `long[93]` — per-cell style columns optional for plain tables. Empty table (nROW=0 or nCOL=0) valid. New column prefix entries added: `real[142]`, `long[92]`, `string[302]`; `long[90]`/`long[91]`/`long[93]`/`char[280]`/`long[421]` extended with ACAD_TABLE uses. ACAD_TABLE rows added to dxf-2018 column additions table. ACAD_TABLE removed from Planned for CSVIN list. |
| 1.0 | 2026-06-23 | `long[421]` on MTEXT background fill documented: exact RGB override (`(R<<16)\|(G<<8)\|B`) present alongside `int[63]` (ACI approximation) when a true-color background fill is set; `long[421]` overrides `int[63]` for display; both present simultaneously; CSVIN-safe to emit both. Added to column prefix table, MTEXT quick reference fill-tail description, and dxf-2018 column additions table. `real[47]` (group-47 pixel size threshold) added to dxf-2018 column additions table: present in CSVOUT, never emit in AI-generated files, CSVIN computes internally from boundary flags. `sample_csvout_reference.csv` added to published files list with URL and full entity coverage description. |
| 1.0 | 2026-06-23 | HATCH `long[92]` loop type flag bits documented: bit 0=external, bit 1=polyline, bit 2=outermost, bit 3=textbox island, bit 4=hole/island; common values decoded (1, 5, 7, 17, 24). `int[72]` dual role clarified: for polyline boundaries (bit 1 of `long[92]`) it is the closed-polyline flag (0=open, 1=closed); for edge boundaries it is the per-edge type list. `real[47]` (group-47 pixel size threshold) documented as CSVIN-internal — not to be emitted. Group-47 note added to quick reference and dxf-2018 prose. `BLOCK` `int[70]=4` (xref block definition) added to flags table and BLOCK/INSERT conventions. LAYER `int[70]` flag bits decoded (1=frozen, 2=frozen-in-new-viewports, 4=locked, 16=xref-dependent, 64=used) with note that xref-dependent layers appear in the LAYER table but are never referenced by entities. LWPOLYLINE `int[70]` plinegen flag documented: bit 7 (128) = linetype generated continuously across vertices; common values 0=open, 1=closed, 128=open+plinegen, 129=closed+plinegen. `sample_csvout_reference.csv` added as a new CSVOUT reference sample illustrating structural variants of HATCH (19 examples covering FP_*, ESCHER, ANSI31, SOLID plain/gradient, circle boundaries, islands, mixed arc+line edges), MLINE (7 examples with int[70] variants), SPLINE (12 examples), POLYLINE (9 examples), MESH (8 examples), DIMENSION (11 examples), and TEXT alignment. |
| 1.0 | 2026-06-22 | HATCH `pt[10]` polyline boundary structure documented. TEXT justification corrected. HATCH removed from `sample_entities.csv`. TEXT alignment rows added. BLOCK/INSERT conventions, `long[420]` on LAYER, `elev[38]`/`thick[39]`, `trans[440]` encoding, RAY/XLINE direction vector clarified. |
| 1.0 | 2026-06-21 | `sha1:000000000000` added to the published starting points library. Blank separator line before trailing metadata reworded as an explicit requirement. |
| 1.0 | 2026-06-20 | Workflow decision guide added: AutoCAD-session vs `-dsm` DXF-file import paths, nudge requirement for DIMENSION regeneration, sha1 target selection table, layer handling guidance, AutoCAD-not-required path. `sample_mtext.csv` added: a complete periodic table of the elements built from multi-line MTEXT cells, demonstrating background fill, defined column height, tightened line spacing, middle-center attachment, and `\W` width-factor scaling for overflow text, with `_ai` layer notes on the `^J` soft-return vs. `\P` paragraph-break distinction. MLINE new entity fully documented. HATCH, MTEXT, DIMENSION, LEADER documentation updated. |
| 1.0 | 2026-06-07 | DIMENSION fully documented: `int[70]` type flags decoded (low 4 bits = type, high bits = positioning), `pt[15]`/`pt[16]` for arc/angular dimensions, `real[144]` actual measurement (derived), `real[40]` leader length, `char[271]`/`char[272]` per-entity style overrides. `real[48]` linetype scale added to standard columns. `char[280]` on HELIX documented (handedness). |
| 1.0 | 2026-06-04 | SPLINE: int[72/73/74] now documented as absent (not merely optional) — derived by CSVIN from field counts. pt[12]/pt[13] independently optional confirmed. real[44] absent when no fit points. int[70] upper bits stripped — AI generators use bits 0–4 only. MESH: int[92]/int[95] absent (derived). long[90] formula documented: face section + edge section (int[93] + int[94]×2 values); fixed (90.0) sentinel appended by CSVIN, not a CSV value. real[140] absent when all creases 0.0. long[91] subdivision level absent when 0. Derived counts design principle added. MESH added to quick reference. Duplicate POLYLINE entry removed. source: convention for AI-generated content added. |
| 1.0 | 2026-06-02 | POLYLINE/VERTEX fully corrected from verified CSVOUT output: spline-fit insert vertex is int[70]=18 (16+2, not 16); int[70]=4 on VERTEX is invalid — never emit; 3D polyline vertex flag is int[70]=32 (not 16); int[70]=128 linetype pattern applies to 2D only; curve-fit interleaving confirmed (original int[70]=2 with angle[50], generated int[70]=1 with real[42]). sample_polylines.csv added. |
| 1.0 | 2026-05-31 | `-dsm` import flag added with `-dwg`/`-dxf` output flags. `sha1:781e2fb2654f` (`new.dxf`) added to sha1 library with download URL and structure description. CSVIN modify/create modes documented in design principle. HATCH corrections: `angle[52]` not always 0; `long[98]` absent when not associative; `real[40]`/`angle[50]`/`angle[51]` confirmed comma-delimited for multiple circle boundaries; `_U` added as valid user-defined pattern name. AI generation promoted to first-class source in generating section. Basic usage restructured — ribbon/command interface first, CLI as secondary. |
| 1.0 | 2026-05-16 | HATCH fully documented from real-world examples: complete column table, circle boundary encoding, SOLID fill, custom pattern names, parallel comma-delimited arrays. HATCH added to CSVIN supported entities. text[1]/name[2]/layer[8] column role disambiguation added to AI generation guidance — these three are never interchangeable. |
| 1.0 | 2026-05-07 | First non-experimental CSVIN release. POLYLINE/VERTEX documented fully: int[70] vertex flags (all 8 bits), real[40]/real[41] width defaults and per-vertex widths with absence rule, curve-fit/spline-fit vertex handling — CSVOUT emits all vertices, AI generation should emit control vertices only. -dxs default behavior documented. DIMSTYLE group code range noted as open-ended. sha1 library concept documented — open standard, self-enforcing via CSVIN hash verification. sample_ai_bracket.csv and sample_ai_electrical.csv added. Row alignment and column minimization rules added to generation guidance. |
| 1.0 | 2026-05-01 | AI directives section added — TEXT or MTEXT entities on layer _ai (or _claude) are instructions to an AI consumer, not drafting geometry. |
| 1.0 | 2026-04-29 | dxf-2018 entity support: SPLINE, ELLIPSE, HATCH, MTEXT, LWPOLYLINE, RAY, XLINE, LEADER, TOLERANCE, ACAD_TABLE, MULTILEADER, MESH documented. Sample files published with URLs. CSVIN architecture documented. |
| 1.0 | 2026-03-31 | Initial release. R12 baseline with r12-extended-semantics design principle. |
---

*This document is the machine-readable specification for DXF-CSV v1.0. It is intentionally written for AI language model consumption — terse, structured, and complete. For human documentation see https://drawingsync.com/doc*

---

© 2026 DrawingSync / Code Truck. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).  
You may use, implement, and share this specification freely with attribution.  
DXF-CSV is an open format — adoption and implementation are encouraged.
