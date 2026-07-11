# Atrada Assembly Builder

Monday.com-style board for tracking Atrada assemblies (site boxes, lifting cages, modular
units), with automatic BOM tree extraction from Master BOM PDFs and revision history PDFs.

## Project Identity

- **App:** Single-file static app (`index.html`), all HTML/CSS/JS in one file.
- **GitHub repo:** https://github.com/ChrisAviso/atrada-assembly-builder
- **Live URL:** https://chrisaviso.github.io/atrada-assembly-builder/
- **Deployed via:** GitHub Pages, builds from `main` branch on every commit to `index.html`.
- **Supabase project:** "Atrada Assembly Builder", ref `fiptponjqtoysddrphmo`, region `ap-southeast-1` (Singapore)
- **Supabase anon key** (safe client-side, protected by RLS):
  `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImZpcHRwb25qcXRveXNkZHJwaG1vIiwicm9sZSI6ImFub24iLCJpYXQiOjE3ODM2OTQ3OTksImV4cCI6MjA5OTI3MDc5OX0.gZCyCX-NeVXMf8gqlve7NJ81QmtcspNYNTjjPfa01m4`
- Auth: Supabase Auth, invite-only (public signup disabled, email confirmation disabled).
  Invite team members via Supabase Dashboard → Authentication → Users → Add user (set a
  password directly there).

This is a separate Supabase project from the earlier "Assembly Builder" project
(ref `fjqrjjejwzkicyobelry`) behind the sub-assembly classifier tool
(`atrada-subassembly-classifier`) — Chris asked for a fresh project and a fresh repo for
this monday.com-style board rather than extending the classifier.

## Data Model

- **`assemblies`** — one row per assembly (e.g. TMSB201). Board row = one assembly.
- **`bom_nodes`** — recursive tree (`parent_id` self-reference), arbitrary depth.
  `node_type` is `sub_assembly` (structural group) or `item` (leaf line item).
  `category` is `sheetmetal` / `weldment` / `fastener` / `label` / `other`, matching the
  Master BOM's SHEETMETAL / WELDMENTS / FASTENERS & FITTINGS / LABELS & STICKERS grouping.
  Item columns (`item_no`, `part_number`, `description`, `material`, `length_mm`, `qty`,
  `netsuite_code`, `remarks`) map 1:1 to the Master BOM's 8-column table
  (# / SUB-ASSEMBLY-PART NAME / DESCRIPTION / MAT'L / LENGTH(mm) / QTY / NETSUITE CODE / REMARKS).
- **`standard_parts`** — reusable library keyed by NetSuite code (not yet populated
  automatically — see "Known gaps" below).
- **`revisions`** — full audit trail, one row per revision entry (`part_number` null =
  whole-assembly revision).
- **`attachments`** — uploaded PDF/DXF files, stored in the `assembly-drawings` Storage
  bucket (private), one row per file with `doc_kind` and `parse_status`.

## PDF Parser

Parsing happens client-side via pdf.js (text-position extraction, no OCR). Two document
types are auto-parsed on upload:

1. **Master BOM PDF** (`parseMasterBom`) — the formatted "BILL OF MATERIALS" printout with
   SHEETMETAL / WELDMENTS / FASTENERS & FITTINGS / LABELS & STICKERS sections. The parser
   distinguishes the **detail** section (bare category header, e.g. plain "SHEETMETAL") from
   the earlier **procurement rollup** sections (e.g. "SHEETMETAL ORDER SUMMARY (PROCUREMENT)",
   "WELDMENTS - 8M BAR ORDER SUMMARY", "CUTTING PLAN", "ORDER SUMMARY - REMAINING GROUPS") by
   matching category headers *exactly* (bare name only) rather than by substring, and treats
   any header containing ORDER SUMMARY / PROCUREMENT / BAR / CUTTING PLAN / REMAINING GROUPS as
   a marker to skip. A `pageHasSkipMarker` flag (reset per page) prevents bare category names
   that appear *inside* a procurement rollup (e.g. "REMAINING GROUPS" listing "FASTENERS &
   FITTINGS" as a plain sub-label) from incorrectly re-entering detail mode — this was a real
   bug caught by testing against the actual TMSB201 Master BOM PDF, fixed by verifying the
   parser logic in Python against pdfplumber word positions before porting to JS.
   Column boundaries (x-position → field) were read directly off the real PDF's word
   coordinates, not guessed.
2. **Revision History PDF** (`parseRevisionHistory`) — the standalone multi-column revision
   log. This table has a real rendering quirk: a revision's multi-line "Revision Details"
   cell can start rendering a few points *above* that revision's own Rev# marker row (because
   short single-line cells in the same row appear bottom-aligned while the tall details cell
   starts top-aligned). The parser handles this with a small lookahead tolerance (12pt) when
   attributing continuation rows to a revision, verified against the real TMSB201 revision
   history PDF.

Both parsers were validated by porting the exact logic to Python and running it against the
real uploaded PDFs via `pdfplumber` before being ported into the browser's pdf.js-based code,
to avoid shipping unverified heuristics.

**Full Assembly Reference** and **Standard Parts** PDFs are detected (by filename convention
first, since both contain a mix of "ASSEMBLY DRAWING REFERENCE" and "PART DRAWING REFERENCE"
pages and can't be reliably told apart by content alone) and stored as attachments, but are
not yet auto-parsed into the BOM tree — see "Known gaps."

DXF files are stored as attachments only, never parsed.

## Known gaps / next steps

- **Full Assembly Reference PDFs are not parsed.** These are the raw multi-page exploded-view
  assembly + part drawings (e.g. `_TMSB201__FULL_ASSY_-_REV_03.pdf`). They contain the true
  parent→child explosion (which items are themselves sub-assemblies that get exploded on
  later pages) but every sheet uses a different BOM table shape, so this needs its own parser
  in a future session — likely reusing groundwork from the sub-assembly classifier project's
  "Stage 1 raw assembly drawing parser."
- **Standard Parts PDFs are not parsed into `standard_parts`.** These are single-part
  reference sheets (stickers, plaques) that could seed the reusable parts library
  automatically.
- **Revisions tab only supports adding via a prompt() dialog**, not inline editing of
  existing rows — fine for now since Master BOM / Revision History re-uploads replace the
  whole tree/revision list, but a proper edit UI would help for one-off manual corrections.
- Revision History parsing occasionally attributes a line to the wrong column when text wraps
  right at a column boundary (a few words can land in "details" vs "reason" ambiguously) —
  content is preserved, just not perfectly split. Not worth over-engineering further unless it
  causes real problems.

## Deployment process

Same pattern as the SO Schedule board and BOM classifier — no local persistent dev
environment, so each session must:

1. Fetch the live source fresh: `https://raw.githubusercontent.com/ChrisAviso/atrada-assembly-builder/main/index.html`
2. Edit locally, verify JS syntax with:
   `node -e "const fs=require('fs'); const html=fs.readFileSync('index.html','utf8'); const m=html.match(/<script>([\s\S]*)<\/script>\s*<\/body>/); new Function(m[1]); console.log('OK');"`
3. Deploy via GitHub's web upload/edit page using browser automation (Chris is non-technical
   and doesn't manage git locally).
