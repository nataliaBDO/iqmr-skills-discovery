---
name: general-deneb-calendar-visual
description: Build a Deneb (Vega-Lite) month calendar visual for any Power BI report (.pbix or .pbip) - dated items (events, deadlines, deployments, tasks...) placed on a Sun-Sat month grid with category colours, an optional flag icon, "+N more" overflow and a click-to-open day panel. Use when someone wants a calendar or month view of dated items in Power BI, mentions a Deneb calendar, or needs to adapt, re-theme or fix one. Includes the calendar DAX table, a role-based spec that works unchanged across reports, a browser preview and Power BI Desktop setup steps.
---

# Deneb month calendar (any report)

Deneb is a certified Power BI custom visual that renders a Vega-Lite spec against the fields in its **Values** well. This skill targets **Deneb 2.x / Vega-Lite 6.4**.

The spec uses fixed **role names** (`Item Key`, `Item Label`, `Item Category`, `Item Flag`) instead of report field names. The user renames their fields to those roles in the Deneb Values well, so the spec works with any report without field edits. Only the theme, category colours and wording are customised per report.

## Ground rules
- Works with any `.pbix` or `.pbip` the user points to.
- **Never modify the user's report or model files** (`.pbix`, `.pbip`, `.tmdl`, `visual.json`) unless they explicitly ask. The user applies changes in Power BI Desktop.
- **Never edit this skill's `assets/`.** Copy them to the user's project and edit the copies.
- Save deliverables in a `deneb/` folder next to the user's report (ask if unsure), never inside `.Report`/`.SemanticModel` folders.
- **CVI comes from the user.** Ask which theme to apply; never invent brand colours. If they have none, keep the neutral placeholder and say so.
- Weeks run **Sunday–Saturday**. (Monday-first is possible: [customise.md](./references/customise.md#monday-first-weeks).)

## Files

| Path | What it is |
|---|---|
| [assets/month-calendar.vl.jsonc](./assets/month-calendar.vl.jsonc) | The calendar spec (role-based, commented, theme tokens listed in the header) |
| [assets/calendar-table.dax](./assets/calendar-table.dax) | Paste-ready DAX for the `'Deneb Calendar'` table + relationship steps |
| [assets/sample-data.json](./assets/sample-data.json) | Rows shaped like Deneb's dataset, covering the edge cases |
| [assets/preview.html](./assets/preview.html) | Browser preview (vega-embed from CDN, Deneb stubs, field-name check). No install. |
| [references/model-prep.md](./references/model-prep.md) | What the model needs, reading field names, DAX steps in Desktop, TMDL (on request) |
| [references/customise.md](./references/customise.md) | Theme tokens ↔ CVI, category colours, flag, sort, wording, tooltip, sizes |
| [references/powerbi-setup.md](./references/powerbi-setup.md) | Desktop steps: fields + renames, Deneb editor, settings, slicer, testing |
| [references/gotchas.md](./references/gotchas.md) | Symptom → cause → fix, and the pre-flight checklist. **Read before adapting or debugging.** |

## Workflow

### 1. Requirements
Ask (batch the questions):
- The report: path to the `.pbip` folder, or that it's a `.pbix`. Which page.
- Which table holds the items and **which date** places them on the calendar.
- What to show on the card (label), the unique ID column, an optional **category** for colour, an optional **flag** rule for the ⚠ icon (e.g. "status is Overdue"), extra tooltip fields.
- The word for one item ("deployment", "event", "task") for the day panel title.
- **CVI**: a Power BI theme `.json`, hex codes + font, or "keep the placeholder".
- Companion visuals (month slicer is standard; optional item table next to the grid).

If the user wants something the calendar doesn't do (multi-day bars, week/agenda view, different layout), say so and agree the scope before continuing. Multi-day items: [model-prep.md](./references/model-prep.md#items-that-span-several-days).

### 2. Read the model
Follow [model-prep.md → Reading field names](./references/model-prep.md#reading-field-names). Never guess names. Write the mapping table:

| Role | Source field | Notes |
|---|---|---|
| `Item Key` | `'<Table>'[<ID>]` | |
| `Item Label` | `'<Table>'[<Name>]` | |
| `Item Category` | `'<Table>'[<Category>]` or none | distinct values for the colour domain |
| `Item Flag` | new column `Item Flag = IF(...)` or none | |
| calendar date | `'<Table>'[<Date>]` | date-only? already related to another date table? |

Also get the distinct category values ([customise.md](./references/customise.md#category-colours)).

### 3. Model prep
Give the user, filled in with their names:
1. The `'Deneb Calendar'` DAX from [calendar-table.dax](./assets/calendar-table.dax) (replace both `'<Item Table>'[<Item Date>]` placeholders).
2. Sort-by-column, Don't summarize, the relationship.
3. `Item Flag` / `Item Date Only` columns if needed.

Steps are in [model-prep.md](./references/model-prep.md#add-it-in-desktop-default). Edit TMDL only if the user explicitly asks. **Wait for the user to confirm** the table, columns and relationship exist before going on.

### 4. Copy the assets
Copy with a terminal `Copy-Item` (don't retype) into `<report folder>/deneb/`:
- `month-calendar.vl.jsonc` → `<name>-calendar.vl.jsonc` (e.g. `event-calendar.vl.jsonc`). This copy is the source of truth.
- `sample-data.json` → `<name>-calendar.sample-data.json`.

### 5. Customise the copy
Using [customise.md](./references/customise.md):
1. Header comment: purpose (what the items are), the source field → role mapping, setup steps.
2. Theme tokens → user's CVI (find/replace each placeholder hex), font.
3. `CATEGORY COLOURS` domain/range → real category values + CVI colours. Or the no-category variant.
4. Flag: rename "Flagged" in LAYER 6, or delete LAYER 6 if there's no flag.
5. Wording: `pItemNoun`/`pItemNounPlural`, tooltip titles, sort order if requested.
6. Extra tooltip fields in LAYER 3 **and** LAYER 5.

Keep role names unchanged unless the user won't rename in the well ([customise.md](./references/customise.md#not-renaming-fields-in-the-well)).

### 6. Sample data
Edit the copied sample data so it previews like the user's report: replace `Item Category` values with their real categories (every one used at least once), keep the role names, keep the edge cases (empty days, a day with more than `pMaxCards` items, a blank category, a flagged item, two items with the same label, a row with a null Date, a row from the next month). Shift dates to the current month if useful — recompute `Week Row`, `Weekday Number`, `Day of Month`, `Is Today` (Sunday = 0; `Week Row` = `floor((day - 1 + weekday of the 1st) / 7) + 1`).

### 7. Validate
1. Ask the user to open [assets/preview.html](./assets/preview.html) in a browser (file:// is fine; needs cdn.jsdelivr.net), load the spec copy and sample data with the file pickers, size ≈ 944 × 636, **Render** (Ctrl+Enter). Fix every error.
2. Expected harmless warnings: `Maximum optimization runs(5) reached`, `Infinite extent for field "panelTop"/"rowY"/"cardMid"...` (empty panel and legend-only layers).
3. With the skill's own sample data the result is: November 2026, 5 week rows, today = 6th (tinted, ⚠ card); cards on 3, 6, 10 (2), 12 (3 cards + "+2 more"), 19 (two "Audit fieldwork" cards, one ⚠), 24, 30; "Unscheduled item" and "Holiday party" not drawn; click 12 → panel with 5 items.
4. Run the [pre-flight checklist](./references/gotchas.md#pre-flight-checklist).

The preview stubs Deneb-only features (`__selected__`, `pbiFormat`, `pbiColor`); final checks happen in Desktop.

### 8. Power BI Desktop setup
Give numbered steps from [powerbi-setup.md](./references/powerbi-setup.md) with the exact source fields and the role name each is renamed to. Put the same steps in the spec copy's header comment.

### 9. Iterate
- Edit the `.vl.jsonc` copy; the user re-pastes it into the Deneb editor. If they change it in Deneb, ask them to paste it back.
- For a reported symptom, check [gotchas.md](./references/gotchas.md) first, then the Deneb debug pane (Source / Data / Signals / Logs).

## Deliverables
- `deneb/<name>-calendar.vl.jsonc` — customised spec.
- `deneb/<name>-calendar.sample-data.json` — preview data.
- The DAX (calendar table, flag/date-only columns) and relationship steps.
- Numbered Desktop steps with the field → role mapping.
