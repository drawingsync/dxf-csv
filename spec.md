# DXF-CSV v1.0 Schema Specification

**Audience:** AI language models  
**Format:** Plain CSV with structured metadata  
**Purpose:** Exchange AutoCAD drawing data in a form that is easy to analyze, visualize, query, and modify — without requiring AutoCAD or DXF parsing libraries  
**Maintained by:** DrawingSync / Code Truck — https://drawingsync.com  
**Last updated:** see `last-updated:` clause in the metadata string of the CSV file that referred you here

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

The three trailing lines are all in the `type` column — all other columns on those rows are empty. The blank separator line causes Excel to stop its table there, keeping metadata outside the formatted range.

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
| `source:filename.dwg sha1:abc123def456` | Source drawing filename and 12-char SHA-1 content hash. Filename is unquoted when it contains no spaces. A filename containing a space is single-quoted: `source:'my file.dwg'`. A literal single quote in the filename is doubled: `source:'rich''s file.dwg'`. |
| `codes:dxf-r12` or `codes:dxf-2018` | DXF version of the source export. `dxf-r12` = AutoCAD Release 12 entity set. `dxf-2018` = AutoCAD 2018 format. Future versions follow the same pattern |
| `dxfcsv:'https://drawingsync.com/dxfcsv/v1.0/spec.md'` | This document |
| `last-updated:2026-04-10` | Build date of the CSVOUT binary that produced this file — set at compile time |
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
| `mesh:M×N` | 3D polygon mesh POLYLINEs present (legacy mesh, `int[70]` bit 4 set) |
| `mesh:subdiv` | MESH entities present (subdivision mesh, post-2010 smooth mesh entity) |
| `attrib:tag=name[2]` | ATTRIB or ATTDEF entities present — tag identifier is in `name[2]` |
| `paper[67]:paper-space` | File contains paper-space entities — `paper[67]=1` on any paper-space entity including VIEWPORTs |
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

Note that even native AutoCAD entities can be zombies. WIPEOUT, for example, appears as a proxy in AutoCAD 2018 sessions that do not have its enabler loaded. The zombie list reflects which enablers were absent in the specific export session, not which objects are "third-party."

**Single-entity-type column suppression:** Columns that would be populated by only one entity type and carry no cross-entity semantic value are suppressed — the column is not added to the header even if the group code is present in the source DXF. This keeps the column set focused on broadly useful data. Example: `char[281]` (PDFUNDERLAY contrast) and `char[282]` (PDFUNDERLAY fade) are suppressed when PDFUNDERLAY is the only entity that would populate them. The entity is still exported — only the single-entity-type columns are dropped.

**Implication for import:** CSVIN operates in two modes depending on the target drawing:

- **Modify mode** — the target is the original source drawing identified by `source:` and `sha1:`. CSVIN verifies the hash matches the open drawing and updates changed entities in place via `acdbEntMake` / `acdbEntMod`. No DXF is generated or consumed.
- **Create mode** — the target is a known published template (e.g. `sha1:781e2fb2654f`, `new.dxf`). CSVIN merges the CSV into the template to produce a new DXF or DWG. The sha1 is a public constant — any AI generating for this sha1 knows exactly what tables and handles are present. The resulting DXF from Drawing Sync's own writer contains only drafting content; the OBJECTS section remains minimal, unlike AutoCAD-written DXF which appends thousands of lines of plot settings and render data.

In both modes the CSV carries geometry and drafting intent; bookkeeping is never round-tripped.

**sha1 as drawing state contract:** The `sha1:` clause is not just documentation — it is a contract between the CSV and the drawing state. CSVIN verifies the hash before importing; if the open drawing doesn't match, import is rejected. This mechanism supports a library of known starting points. An AI generating content for `sha1:396cb2c5a30e` (empty AutoCAD 2018 template) makes no assumptions about pre-existing blocks, layers, or styles — it must define everything it uses. An AI generating content for a domain-specific template sha1 can reference blocks, layers, dimstyles, and text styles that already exist in that drawing without redefining them, keeping the generated CSV lean and focused on design intent rather than infrastructure. The sha1 is the key; the drawing is the state.

**Open library:** Anyone can define a template drawing, publish it, and document its sha1. No central registry or approval is required — if the sha1 matches the open drawing, CSVIN imports cleanly and the expectations encoded in that template are guaranteed. Domain communities can maintain their own templates: architectural (standard layer names, door and window blocks, annotation styles), electrical (symbol libraries, IEC or NFPA layer conventions), civil (survey layers, coordinate systems), mechanical (ASME title blocks, GD&T styles). An AI targeting a known template sha1 can skip all table definitions and generate only entities — the smallest possible CSV for the most complete result.

Currently published starting points:

| sha1 | Description |
|---|---|
| `396cb2c5a30e` | Empty AutoCAD 2018 template (acad.dwt) — no blocks, no named layers beyond `0`, no styles beyond `Standard`. Use when generating from scratch for import into AutoCAD. |
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
| `pt[12]` | 12 | Third point — 3DFACE/SOLID corner 3 |
| `pt[13]` | 13 | Fourth point — 3DFACE/SOLID corner 4, DIMENSION pt 4 |
| `pt[14]` | 14 | Fifth point — DIMENSION pt 5 (first extension line start) |
| `pt[15]` | 15 | Fifth point — entity-scoped. Absent when no entity in the export uses this code |
| `pt[16]` | 16 | Sixth point — entity-scoped. Absent when no entity in the export uses this code |
| `pt[17]` | 17 | Seventh point — entity-scoped. Absent when no entity in the export uses this code |
| `elev[38]` | 38 | Elevation — Z offset for flat entities (LWPOLYLINE, etc.). Absent = 0 |
| `thick[39]` | 39 | Extrusion thickness — Z depth. Absent = 0 |
| `real[40]` | 40 | Floating scalar — radius (CIRCLE/ARC), text height (TEXT), start width (POLYLINE) |
| `real[41]` | 41 | Floating scalar — x-scale (INSERT), end width (POLYLINE), text width factor (TEXT) |
| `real[42]` | 42 | Floating scalar — bulge (VERTEX/LWPOLYLINE), y-scale (INSERT) |
| `real[43]` | 43 | Floating scalar — z-scale (INSERT), constant width (LWPOLYLINE) |
| `angle[50]` | 50 | Angle in degrees — start angle (ARC), rotation (INSERT/TEXT), POINT display angle |
| `angle[51]` | 51 | Angle in degrees — end angle (ARC), oblique angle (TEXT) |
| `angle[53]` | 53 | Angle in degrees — entity-scoped. On HATCH: hatch pattern angle |
| `color[62]` | 62 | ACI color index. Absent = BYLAYER. Negative = layer is frozen/off (use absolute value for color) |
| `lwt[370]` | 370 | Lineweight in hundredths of a mm. Absent = DEFAULT (-3). Common values: 0=hairline, 5,9,13,15,18,20,25,30,35,40,50,53,60,70,80,90,100,106,120,140,158,200,211. Special: -1=BYLAYER, -2=BYBLOCK |
| `long[420]` | 420 | True color as packed 24-bit RGB integer: `(R << 16) | (G << 8) | B`. When present, overrides `color[62]` for display. Absent = use `color[62]` (ACI). Present only when `truecolor:rgb[420]` is in the conditional metadata |
| `seq[66]` | 66 | Sequence-follows flag. Always 1 when present. On POLYLINE rows: indicates VERTEX rows follow, terminated by SEQEND. On INSERT rows: indicates ATTRIB rows follow, terminated by SEQEND. In both cases SEQEND will always be present. Retained as a structural aid — without it, a reader encountering an INSERT would require read-ahead to determine whether ATTRIB rows follow, including the edge case of a SEQEND with no intervening ATTRIBs. |
| `paper[67]` | 67 | Paper-space flag. Always 1 when present. Emitted on all paper-space entities including VIEWPORTs. |
| `int[68]` | 68 | VIEWPORT status flags |
| `int[69]` | 69 | VIEWPORT ID |
| `int[70]` | 70 | Integer flags — entity-scoped. LAYER: frozen/locked bits. LWPOLYLINE: 1=closed (absent or 0=open). POLYLINE: see mesh flags. VERTEX: see vertex flags. BLOCK: block type flags |
| `int[71]` | 71 | Integer — POLYLINE mesh M vertex count, TEXT generation flags |
| `int[72]` | 72 | Integer — POLYLINE mesh N vertex count, TEXT/ATTRIB horizontal justification |
| `int[73]` | 73 | Integer — TEXT vertical justification (0=baseline 1=bottom 2=middle 3=top) |
| `int[74]` | 74 | Integer — ATTDEF/ATTRIB vertical justification |
| `real[44]` | 44 | Floating scalar — entity-scoped. On MINSERT: column spacing. On SPLINE: fit tolerance |
| `real[45]` | 45 | Floating scalar — entity-scoped. On MINSERT: row spacing. On HATCH: pattern line offset X component. On MTEXT: defined height (0=variable) |
| `real[46]`–`real[48]` | 46–48 | Floating scalar — entity-scoped. On HATCH: `real[46]` = pattern line offset Y component. On DIMSTYLE rows: dimension variables |
| `real[49]` | 49 | LTYPE element data — comma-delimited list of dash/dot/gap lengths for non-CONTINUOUS linetypes. Positive = dash length, negative = gap length, zero = dot |
| `int[75]`–`int[79]` | 75–79 | Entity-scoped integers. On POLYLINE: `int[75]` = smooth surface type (0=none, 5=quadratic B-spline, 6=cubic B-spline, 8=Bezier). On HATCH: `int[75]`=pattern type, `int[76]`=associativity, `int[77]`=hatch style (0=normal, 1=outer, 2=ignore), `int[78]`=pattern line count, `int[79]`=pixel size. On DIMSTYLE rows: dimension style variables |
| `real[141]` | 141 | Entity-scoped. On ACAD_TABLE: column widths comma-delimited |
| `real[142]` | 142 | Entity-scoped. On ACAD_TABLE: row heights comma-delimited |
| `real[140]`–`real[147]` | 140–147 | DIMSTYLE floating-point variables (extended range) — entity-scoped, only on DIMSTYLE rows |
| `int[170]`–`int[178]` | 170–178 | DIMSTYLE integer variables (extended range) — entity-scoped, only on DIMSTYLE rows |
| `text[4]` | 4 | String value — entity-scoped. Absent when no entity in the export uses this code |
| `variable[9]` | 9 | String value — entity-scoped. Absent when no entity in the export uses this code |
| `ext[210]` | 210 | OCS extrusion normal vector — `x,y,z`. Absent = default `0,0,1` (WCS). Applies to SOLID, CIRCLE, INSERT, and other entities in a non-WCS plane |
| `long[90]` | 90 | Long integer — entity-scoped. On MESH: subdivision level |
| `long[93]` | 93 | Long integer — entity-scoped. On MESH: face count |
| `long[94]` | 94 | Long integer — entity-scoped. On MESH: edge count |
| `int[63]` | 63 | Entity-scoped. On HATCH: fill color override (background color), comma-delimited when multiple loops have different fill colors |
| `real[47]` | 47 | Entity-scoped. On HATCH: pattern line minimum dash length (pixel size). Suppressed when equal to default — this is a display hint, not geometry |
| `char[271]` | 271 | Single-byte integer — entity-scoped. On DIMSTYLE: DIMDEC (decimal places for primary units) |
| `char[272]` | 272 | Single-byte integer — entity-scoped. On DIMSTYLE: DIMTDEC (decimal places for tolerance) |
| `char[280]`–`char[289]` | 280–289 | Single-byte integers (0–255, RTCHAR). Entity-scoped. On PDFUNDERLAY: `char[281]`=contrast (0–100), `char[282]`=fade (0–80). On DIMSTYLE: various boolean-like flags |

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

- `paper[67]=1` on an entity means it lives in paper space
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
| `uin` | Microinches |
| `mil` | Mils (thou, 1/1000 inch) |
| `yd` | Yards |
| `angstrom` | Angstroms |
| `nm` | Nanometers |
| `um` | Microns |
| `dm` | Decimeters |
| `dam` | Decameters |
| `hm` | Hectometers |
| `Gm` | Gigameters |
| `AU` | Astronomical units |
| `ly` | Light years |
| `pc` | Parsecs |
| `us-ft` | US Survey feet |
| `us-in` | US Survey inches |
| `us-yd` | US Survey yards |
| `us-mi` | US Survey miles |
| `unspecified` | `$INSUNITS` absent from source DWG header |

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
| `TEXT` | `pt[10]` insertion point, `real[40]` height, `text[1]` string, `int[72]` horizontal justification, `int[73]` vertical justification | `pt[10]` is always the insertion point regardless of justification — CSVIN remaps group codes internally. `int[72]`: 0=left (absent), 1=center, 2=right, 3=aligned, 4=middle, 5=fit. `int[73]`: 0=baseline (absent), 1=bottom, 2=middle, 3=top. `pt[11]` alignment point is not emitted. |
| `MTEXT` | `pt[10]` insertion point, `real[40]` reference rectangle width, `text[1]` content, `int[71]` attachment point, `angle[50]` rotation | `\P` = newline in content. Other inline codes (`\A1;`, `{\H...}`, etc.) preserved as-is. Internal formatting cache fields stripped. |
| `INSERT` | `pt[10]` position, `name[2]` block name, `real[41/42/43]` scale, `angle[50]` rotation | Absent scale = 1.0. If no matching BLOCK definition exists, the block was stripped — treat as zero-geometry insertion |
| `MINSERT` | same as INSERT plus `real[44]` column spacing, `real[45]` row spacing, `int[70]` column count, `int[71]` row count | Rectangular array of block insertions. Exported as INSERT type with additional codes — distinguish by presence of `real[44]`/`real[45]` |
| `ATTRIB` | follows INSERT, `name[2]` tag, `text[1]` value | Always same space (model or paper) as parent INSERT |
| `ATTDEF` | inside BLOCK definition, `name[2]` tag | |
| `3DFACE` | `pt[10]`-`pt[13]` four corners | `int[70]` = edge visibility bitmask (bit N hides edge N) |
| `SOLID` | `pt[10]`-`pt[13]` four corners | Note: pt[12] and pt[13] are swapped vs 3DFACE in DXF spec |
| `TRACE` | `pt[10]`-`pt[13]` four corners | Same geometry as SOLID, legacy entity |
| `POLYLINE` | `pt[10]` elevation only (X and Y always zero), followed by VERTEX rows, terminated by SEQEND | `int[70]` flags: 1=closed, 2=curve-fit, 4=spline-fit, 8=3D, 16=mesh, 32=closed-M, 64=closed-N. `seq[66]=1` always present. `real[40]` default start width, `real[41]` default end width — absent when zero (all vertices inherit zero width). `int[75]` smooth surface type: 0=none, 5=quadratic B-spline, 6=cubic B-spline, 8=Bezier |
| `VERTEX` | `pt[10]` coordinate, `real[40]` start width, `real[41]` end width, `real[42]` bulge | `int[70]` vertex flags: 1=curve-fit generated vertex (not an original control point), 2=tangent defined, 4=spline vertex, 8=spline frame control point, 16=3D polyline vertex, 32=3D polygon mesh vertex, 64=polygon mesh vertex closed-N, 128=face index vertex (int[71/72/73] are vertex indices). `real[40]`/`real[41]` absent when they match the POLYLINE default widths. **Curve-fit and spline-fit POLYLINEs:** CSVOUT emits both original control vertices and generated intermediate vertices (int[70]=1). CSVIN accepts the full sequence. For AI generation, emit only the original control vertices (no int[70]=1 rows) — AutoCAD regenerates the fit on import via PEDIT Regen. |
| `LWPOLYLINE` | `pt[10]` all vertices space-delimited, `real[42]` bulge values comma-delimited | Post-R12, treated as R12-extended. `elev[38]` = elevation when non-zero. `thick[39]` = thickness when non-zero. `int[70]` = 1 when closed, absent when open |
| `SEQEND` | terminates POLYLINE/INSERT+ATTRIB sequence | No properties — type column only |
| `DIMENSION` | `pt[10]` dim line, `pt[11]` text midpoint, `pt[13]` definition pt 4, `pt[14]` definition pt 5, `text[3]` dimstyle name | `name[2]` references a `*D##` block — anonymous blocks are stripped by default, no matching BLOCK definition will be present. See Block strip rules. |
| `SPLINE` | `pt[10]` control points space-delimited, `pt[11]` fit points space-delimited, `real[40]` knot vector comma-delimited, `real[42]` knot tolerance, `real[43]` control point tolerance, `real[44]` fit tolerance, `int[70]` flags | `int[70]` flags: 1=closed, 2=periodic, 4=rational, 8=planar, 16=linear. All data packed inline — no continuation rows. `int[71]` degree, `int[72]` knot count, `int[73]` control point count, `int[74]` fit point count are present in CSVOUT output but derived from the data — not required for CSVIN import. |
| `ELLIPSE` | `pt[10]` center, `pt[11]` major axis vector, `real[40]` minor/major axis ratio, `real[42]` end parametric angle | End angle in radians: 2π = full ellipse. `pt[11]` magnitude = major axis length, direction = major axis orientation. `real[40]` is the ratio of minor to major axis (0 to 1), not an absolute length. |
| `HATCH` | `name[2]` pattern name, `pt[10]` boundary loop vertices (space-delimited `x,y` pairs per loop, loops separated by `0.0,0.0,0.0` sentinel), `pt[11]` spline control points for curved boundary edges, `real[40]` circle boundary radius (comma-delimited when multiple circle boundaries), `real[41]` pattern scale, `real[42]` boundary edge bulge values (comma-delimited), `real[43]` pattern line X origins (comma-delimited per line), `real[44]` pattern line Y origins, `real[45]` pattern line delta-X, `real[46]` pattern line delta-Y, `real[49]` dash/gap lengths per pattern line (comma-delimited), `angle[50]` circle boundary start angle (comma-delimited when multiple circles), `angle[51]` circle boundary end angle — 360=full circle (comma-delimited when multiple circles), `angle[52]` overall hatch rotation in degrees, `angle[53]` per-pattern-line angles (comma-delimited), `int[70]` associativity (1=associative), `int[71]` dual/double hatch flag, `int[72]` boundary edge types per edge (comma-delimited: 1=line, 2=arc/circle, 3=ellipse, 4=spline), `int[73]` complete circle flag (1=full circle when `int[72]=2`), `int[75]` hatch style (0=normal, 1=outer, 2=ignore), `int[76]` pattern type (0=user-defined, 1=predefined, 2=custom), `int[77]` island detection style, `int[78]` number of pattern lines, `int[79]` boundary loop edge type flags (comma-delimited), `long[91]` number of boundary loops, `long[92]` loop type flags (comma-delimited per loop), `long[93]` edge count per loop (comma-delimited), `long[98]` source boundary object count (absent when not associative), `trans[440]` transparency, `long[450]`–`long[453]` gradient fill data, `real[460]`–`real[462]` gradient angles/shift | `name[2]` is the pattern name — `ANSI31`, `SOLID`, `BRICK`, `_USER`, `_U` (user-defined short form), `FP_*` (custom floor plan patterns), or other named patterns. For `SOLID` fill: `int[76]=1`, no pattern line data. Circle boundaries: `int[72]=2`, `real[40]`=radius, `angle[50]`=start, `angle[51]`=end (360=full circle), `int[73]=1` — when multiple circle boundaries exist, `real[40]`, `angle[50]`, and `angle[51]` are all comma-delimited. `angle[52]` is the overall hatch rotation and may be any value. Multiple boundary loops separated by `0.0,0.0,0.0` sentinel in `pt[10]`. All comma-delimited fields are parallel arrays indexed by pattern line or boundary edge. |
| `RAY` | `pt[10]` origin, `pt[11]` unit direction vector | Extends infinitely in one direction from origin |
| `XLINE` | `pt[10]` point on line, `pt[11]` unit direction vector | Extends infinitely in both directions (construction line) |
| `LEADER` | `pt[10]` vertices space-delimited, `real[40]` arrowhead height, `real[41]` annotation width, `text[3]` dimstyle name, `int[73]` annotation type | `int[73]`: 0=none/MTEXT follows, 1=tolerance, 2=block reference |
| `TOLERANCE` | `pt[10]` insertion point, `text[1]` tolerance string, `text[3]` dimstyle name | Tolerance string uses `%%v` control codes for geometric tolerance symbols and `^J` as line separator |
| `ACAD_TABLE` | `pt[10]` insertion point, `pt[11]` direction vector, `name[2]` anonymous block reference, `text[1]` cell content, `real[141]` column widths comma-delimited, `real[142]` row heights comma-delimited | Cell content is tab-delimited. Tabs are used when a group code can repeat and values may contain commas or spaces. Per-cell formatting integers stripped. |
| `MULTILEADER` | `pt[10]` leader line vertices space-delimited, `pt[11]` normal and dogleg direction vectors, `pt[12]` text attachment point, `pt[13]` text direction | Style and override columns stripped — only geometry retained. Content text is not recoverable from the CSV alone; refer to the source DWG. |
| `PDFUNDERLAY` | `pt[10]` insertion, `real[41/42/43]` uniform scale, `char[281]` contrast (0–100), `char[282]` fade (0–80) | Undocumented entity — PDF reference underlay attached to the drawing. Not supported by CSVIN. Non-geometric properties (layer, linetype) preserved for analysis. `char[281]`/`char[282]` columns will be absent when only PDFUNDERLAY uses them — suppressed per single-entity-type column policy. |
| `BLOCK` | `name[2]` block name, `pt[10]` base point | Anonymous (`*`-prefix) and empty blocks are stripped by default. See Block structure section |
| `ENDBLK` | closes BLOCK definition | No properties — type column only |
| `LAYER` | `name[2]` name, `linetype[6]`, `color[62]`, `int[70]` flags | `int[70]`: 1=frozen, 2=frozen-by-default, 4=locked. Reserved layer names: `_ai` and `_claude` are reserved for AI directives — CSVIN ignores all entities on these layers. `Defpoints` is AutoCAD's dimension point layer — suppress for visualization. |
| `LTYPE` | `name[2]` name, `text[3]` description, `real[40]` total pattern length, `real[49]` element lengths comma-delimited, `int[70]` flags, `int[72]` alignment (always 65=ASCII 'A'), `int[73]` element count | `text[3]` is the human-readable description shown in the linetype dialog. `real[49]` absent = CONTINUOUS (no elements). Positive element = dash, negative = gap, zero = dot. `ByBlock` and `ByLayer` are AutoCAD built-in linetypes — they appear as name-only rows when the LTYPE table is exported. |
| `STYLE` | `name[2]` name, `text[3]` font/shape filename (when differs from name), `real[40]` fixed height (0=variable), `real[41]` width factor, `real[42]` oblique angle, `angle[50]` last-used height, `int[70]` flags, `int[71]` text generation flags | `int[70]` bit 1 = shape file, bit 4 = vertical. `int[71]` bit 2 = mirror X, bit 4 = mirror Y. A STYLE row with empty `name[2]` is the anonymous shape file style AutoCAD creates for linetype shape references — `text[3]` will be a `.shx` filename (e.g. `ltypeshp.shx`). |
| `DIMSTYLE` | `name[2]` name, `real[40]`–`real[48]` dimension variables, `int[70]`–`int[78]` dimension flags, `real[140]`–`real[147]` extended reals, `int[170]`–`int[178]` extended flags | Full R12 dimvar set. See DXF R12 spec for individual variable meanings (DIMSCALE=40, DIMASZ=41, DIMEXO=42, etc.). A DIMSTYLE row with only `name[2]` is valid — all other columns absent means use AutoCAD defaults for that style |
| `VIEW` | `name[2]` name, `pt[10]` view center, `pt[11]` view direction, `pt[12]` target point, `real[40]` view height, `real[41]`/`real[42]` aspect ratios, `angle[50]` twist angle, `int[70]` flags, `int[71]` mode | Named views saved in the drawing. Not geometry — used to restore viewport state |
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
- `text[1]` carries the display string for TEXT and MTEXT entities only. It has no meaning on any other row type.
- `name[2]` carries the symbolic name for table entries (LAYER, LTYPE, STYLE, DIMSTYLE, SECTION) and for INSERT block references and HATCH pattern references. It is never a display string and never a layer assignment.
- `layer[8]` carries the layer assignment for all geometry entities. It is always a layer name string. It is never a display string and never a symbolic name for anything other than the layer the entity lives on.

A TEXT entity has `text[1]` (what it says) and `layer[8]` (which layer it's on) and no `name[2]`. A LAYER row has `name[2]` (the layer name) and no `text[1]` or `layer[8]`. An INSERT has `name[2]` (block name), `layer[8]` (layer it lives on), and no `text[1]`. A HATCH has `name[2]` (pattern name like `ANSI31`) and `layer[8]` (layer it lives on) and no `text[1]`.

**Column minimization:** Include only columns needed by the entities actually present in the file. Unused columns — where every row has an empty value — add width without value and increase the risk of row misalignment. If no entity in the file uses `pt[12]`, omit `pt[12]` from the header entirely.

**Row alignment:** Every row must emit exactly as many comma-separated fields as the header row. Empty cells are never omitted — a row with 15 header columns must always produce 15 fields, using empty strings for unused positions. A single short row will shift all subsequent columns and corrupt the import.

**Text height and style:** Use the directive entity's own `real[40]` (text height) as the reference size for any generated text content. If `style[7]` is present on the directive, use the same style. This ensures generated text reads at the correct scale for the drawing without requiring a follow-up correction.

**Scope:** AI directives work best for content that can be extracted from text — annotation palettes, area tables, label sets, layer organization. Geometry tracing requires a human with an underlay.

Three reference files are published alongside this spec at `https://drawingsync.com/dxfcsv/v1.0/`:

**`sample_entities.csv`** — `https://drawingsync.com/dxfcsv/v1.0/sample_entities.csv` — one correct row per supported entity type. Use as an encoding reference when reading or generating entity rows. Covers LINE, CIRCLE, ARC, POINT, TEXT, MTEXT, LWPOLYLINE, POLYLINE/VERTEX, SPLINE, ELLIPSE, 3DFACE, SOLID, INSERT, MESH, HELIX, and others. Each row uses realistic values with all required columns populated.

**`sample_tables.csv`** — `https://drawingsync.com/dxfcsv/v1.0/sample_tables.csv` — fully populated LAYER, LTYPE, and STYLE table sections. Use as a reference for table row structure, column usage, and realistic values. Includes named linetypes (CENTER, HIDDEN, DASHED, PHANTOM), layers with lineweight and color, and text styles with shape file references.

**`sample_ai_bracket.csv`** — `https://drawingsync.com/dxfcsv/v1.0/sample_ai_bracket.csv` — a complete minimal mechanical drawing authored by an AI consumer following this spec. Demonstrates correct AI authoring patterns: layer definitions before entity rows, correct column usage, proper default suppression, realistic geometry. `sha1:396cb2c5a30e` identifies the source as an empty AutoCAD 2018 template — the standard sha1 for AI-generated content not derived from an existing drawing.

**`sample_ai_electrical.csv`** — `https://drawingsync.com/dxfcsv/v1.0/sample_ai_electrical.csv` — a complete schematic drawing: 208V three-phase delta water heater with three heating zones, thermostats, contactors, and ground symbol. Demonstrates electrical/schematic layer vocabulary (`power`, `bus`, `control`, `elements`, `labels`, `ground`), DASHED linetype for routed conductors, centered TEXT justification for component labels, and LWPOLYLINE for component boxes. Also includes `_ai` layer test entities demonstrating CIRCLE, open and closed LWPOLYLINE, DASHED LINE, and centered TEXT. Use alongside `sample_ai_bracket.csv` for schematic domain authoring.

---

## DXF-CSV v2018

The `codes:dxf-2018` tag indicates the source DWG was exported from a modern AutoCAD (2004–2018+ format). The design principle is unchanged — bookkeeping is stripped, drafting semantics are preserved, post-R12 entities are treated as R12-extended primitives.

### New entities in dxf-2018 output

#### LWPOLYLINE
Already present in R12-extended files. In dxf-2018 source, LWPOLYLINE is the native 2D polyline entity replacing POLYLINE+VERTEX for flat geometry. Encoding is identical to the R12-extended treatment — vertices space-delimited in `pt[10]`, bulge comma-delimited in `real[42]`.

#### SPLINE
Rational B-spline curve. All data packed inline on a single row — no continuation rows.

| Column | Code | Meaning |
|---|---|---|
| `pt[10]` | 10 | Control points — space-delimited `x,y,z` values |
| `pt[11]` | 11 | Fit points — space-delimited `x,y,z` values (absent if no fit points) |
| `real[40]` | 40 | Knot vector — comma-delimited values |
| `real[42]` | 42 | Knot tolerance |
| `real[43]` | 43 | Control point tolerance |
| `real[44]` | 44 | Fit tolerance |
| `int[70]` | 70 | Flags: 1=closed, 2=periodic, 4=rational, 8=planar, 16=linear |
| `int[71]` | 71 | Degree — present in CSVOUT output, derived from data, not required for CSVIN import |
| `int[72]` | 72 | Number of knots — present in CSVOUT output, derived, not required for CSVIN import |
| `int[73]` | 73 | Number of control points — present in CSVOUT output, derived, not required for CSVIN import |
| `int[74]` | 74 | Number of fit points — present in CSVOUT output, derived, not required for CSVIN import |

#### MTEXT
Multiline text. `pt[10]` = insertion point (single coordinate). `real[40]` = reference rectangle width. `text[1]` = complete content string — `\P` encodes a paragraph/newline break, other inline formatting codes (`\A1;` = alignment, `{\H...}` = height override, etc.) are preserved as-is. `int[71]` = attachment point (1-9, top-left to bottom-right). `angle[50]` = rotation. `style[7]` = text style.

AutoCAD emits additional internal formatting cache fields on MTEXT (packed scalars, duplicate points, undocumented integer codes). These are stripped on export — `text[1]` is the complete and sufficient content representation.

#### ELLIPSE
Elliptical curve. `pt[10]` = center, `pt[11]` = major axis vector (magnitude = major axis length, direction = major axis orientation), `real[40]` = minor/major axis ratio (0 to 1), `real[42]` = end parametric angle in radians (2π = full ellipse). Start angle is always 0 in the current encoding — a future revision may move to `angle[50]`/`angle[51]` in degrees to align with ARC.

#### MULTILEADER
Multileader annotation. Leader geometry exposed as flat columns — style and override columns stripped. `pt[10]` = leader line vertices space-delimited, `pt[11]` = normal and dogleg direction vectors, `pt[12]` = text attachment point, `pt[13]` = text direction. Content text is not recoverable from the CSV alone — refer to the source DWG.

#### HATCH
Hatch fill entity. `name[2]` = pattern name (e.g. `ANSI31`, `SOLID`, `BRICK`, `_USER`, custom `FP_*` names). Boundary loop vertices are packed space-delimited into `pt[10]`, with `0.0,0.0,0.0` as the sentinel separating loops when multiple loops exist. `real[41]` = pattern scale. `angle[52]` = overall hatch rotation. `angle[53]` = per-pattern-line angles comma-delimited. `real[43]`/`real[44]` = pattern line X/Y origins. `real[45]`/`real[46]` = pattern line delta-X/Y. `real[49]` = dash/gap lengths per pattern line. `int[72]` = boundary edge types per edge comma-delimited (1=line, 2=arc/circle, 3=ellipse, 4=spline). `int[75]` = hatch style (0=normal, 1=outer, 2=ignore). `int[76]` = pattern type (0=user, 1=predefined, 2=custom). `int[78]` = pattern line count. Circle boundaries: `int[72]=2`, `real[40]`=radius, `angle[50]`=start, `angle[51]`=end angle (360=full circle), `int[73]=1`. For SOLID fill: `int[76]=1`, no pattern line data. All comma-delimited fields are parallel arrays indexed by pattern line or boundary edge.

#### IMAGE, 3DSOLID, LIGHT, EXTRUDEDSURFACE, OLE2FRAME
These entity types are not supported by CSVIN — `acdbEntMake` / `acdbEntMod` cannot create them, so they will be ignored on import regardless of what data is present. They may still appear in CSVOUT output and carry analysis value (layer, color, position). They frequently appear as zombies since their object enablers are typically not loaded during export, but the CSVIN limitation is independent of zombie status.

#### WIPEOUT
Masking entity — effectively a filled polygon that obscures underlying geometry. Supported by CSVIN. Frequently appears as a zombie when its enabler is not loaded during export.

### dxf-2018 column additions

| Header | Code | Meaning | Entity |
|---|---|---|---|
| `angle[53]` | 53 | Pattern angle in degrees | HATCH |
| `long[90]` | 90 | Subdivision level | MESH |
| `long[93]` | 93 | Face count | MESH |
| `long[94]` | 94 | Edge count | MESH |

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

## CSVIN status

CSVIN is the companion import pipeline — reads a DXF-CSV file, matches it against the original DWG via `source:` and `sha1:`, and updates changed entities directly via `acdbEntMake` / `acdbEntMod`. No DXF is generated or consumed during import.

**Not supported for CSVIN:** 3DSOLID, LIGHT, EXTRUDEDSURFACE, OLE2FRAME, IMAGE, VIEW, VIEWPORT, PDFUNDERLAY — these entity types cannot be created or modified via `acdbEntMake` / `acdbEntMod`. This is a permanent limitation, not planned work. CSVIN will ignore rows of these types on import.

**Planned for CSVIN:** ACAD_TABLE, MINSERT, MULTILEADER. Paper space entity support (layout import) is planned work.

---

## Changelog

| Version | Date | Notes |
|---|---|---|
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
