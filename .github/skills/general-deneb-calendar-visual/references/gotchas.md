# Month calendar gotchas

Each entry: **Symptom → Cause → Fix**. Read before adapting or debugging the spec.

## Data / model

### Empty days missing (holes in the grid)
- **Symptom:** days with no items aren't drawn; the grid has gaps.
- **Cause:** Power BI only returns dates that have matching items.
- **Fix:** right-click `Date` in the Values well → **Show items with no data**. The extra rows carry `null` for every item field; the spec already handles them (`hasItem`).

### Items missing from the grid
Check in this order (Deneb debug pane → **Source**, Ctrl+Alt+6):
1. **No relationship / wrong column:** item rows arrive with `null` Date. Create `'<Item Table>'[<Item Date>]` → `'Deneb Calendar'[Date]`.
2. **Inactive relationship** — see below.
3. **Date has times:** relate a date-only column instead ([model-prep.md](./model-prep.md#item-date-with-times)).
4. **Item date outside the calendar range:** the table covers whole years from the earliest to the latest item date (plus this year). Refresh after loading new data.
5. **Blank label:** items with a blank `Item Label` are treated as empty days.

### Inactive relationship
- **Symptom:** relationship shows as a dashed line in Model view; items never appear.
- **Cause:** Desktop made it inactive because it would create an ambiguous filter path — usually another relationship already links the item table and `'Deneb Calendar'`, or a bi-directional chain connects them.
- **Fix:** find the other path in Model view and ask the user how to resolve it. Don't silently deactivate or delete an existing relationship — other visuals depend on it.

### Measures in the Values well
- **Symptom:** every item appears on every day, or the row count explodes.
- **Cause:** a measure in the well is evaluated for every Date × item combination; if it returns a value (e.g. `0` or `"No"`) instead of `BLANK()`, Power BI keeps every combination.
- **Fix:** use calculated **columns** for tooltip extras. If a measure is unavoidable, make it return `BLANK()` when there's no item.

### Identical rows merge
- Power BI groups by all non-aggregated fields: two items with the same label, category and date become one card. Always add `Item Key`.
- Numeric columns default to **Sum** → set **Don't summarize** for every calendar column and numeric keys. A summed `Week Row` puts cards in the wrong row.

### Field with several values per item
- **Symptom:** the same item shows twice on a day.
- **Cause:** a tooltip field (e.g. owner from a related many-side table) has several values for one item.
- **Fix:** remove it, or replace it with a column that has one value per item (e.g. a `CONCATENATEX` calculated column).

### TODAY() at refresh
- `Is Today` and `Month Selector` ("Current Month") are calculated columns: they change only when the dataset refreshes (UTC in the Service).

### Continuous view 500-row limit
- **Symptom:** the day panel closes when a card is clicked, or whenever another visual filters the page, even with **Enable patching for hosted datasets** on.
- **Cause:** Deneb only patches data into the live view while the dataset is at or under the threshold — **500 rows by default** (configurable up to 5,000). Above it, Deneb recompiles and click state is lost. The Continuous view section shows whether patching is active; fallbacks appear in the Logs pane.
- **Fix:** keep the month slicer on one month (≈ 30 empty days + one row per item). With the slicer cleared, every day of every year comes in — expect the panel to stop surviving clicks.

## Spec

### Wrong or missing field names
- **Symptom:** a layer draws nothing, a tooltip row is empty, every card is "No status" — no error.
- **Cause:** a name in the spec doesn't exactly match the dataset; Vega-Lite treats a missing field as `undefined` silently.
- **Fix:** check the role names in the Values well (case and spaces: `Item Label`, not `Item label`). The dataset uses the name shown in the well. Confirm in debug pane → **Source**.

### Category colours wrong or shifting
- **Cause:** a category value in the data isn't in the `CATEGORY COLOURS` domain (spelling, case, trailing space), so it gets a recycled colour.
- **Fix:** rerun `EVALUATE DISTINCT(...)` and copy the values exactly ([customise.md](./customise.md#status-colours-rag)).

### Key shows the wrong categories or colours
- **Cause:** the key is drawn from the `pKeyItems` param, not from the data, and coloured through the shared `categoryLabel` scale. If `pKeyItems` and the `CATEGORY COLOURS` domain differ, the key lists categories that never appear, misses some, or a key value not in the domain gets a recycled colour.
- **Fix:** keep `pKeyItems` = the domain without `"No status"`, same spelling and order. Don't re-enable the built-in legend (`config.legend.disable`) — it would draw a second key.

### Key entries overlap or are spaced unevenly
- **Cause:** Vega-Lite can't measure text; entry widths are estimated as characters × `pFontKey` × `pCharW`. Fonts narrower or wider than Trebuchet MS / Segoe UI throw the estimate off.
- **Fix:** raise `pCharW` (e.g. 0.6) if labels touch the next circle; lower it (e.g. 0.5) if gaps are too wide.

### Header row sits too low or is clipped
- **Cause:** the Deneb visual's own padding (Format → General → Properties) adds space around the spec, so the header row no longer lines up with controls laid over it; or the spec's top padding was reduced.
- **Fix:** set the visual's padding to 0 on all sides; keep the spec's `"padding": {"top": 12, ...}`.

### Axis missing in a layered chart
- **Symptom:** an axis (e.g. a weekday header) set on one layer doesn't show.
- **Cause:** layers sharing the scale merge their axis settings; `"axis": null` on the other layers can win and hide it.
- **Fix:** draw headers/labels as a text layer (as the WEEKDAY HEADER does) instead of relying on an axis.

### datum + field crash
- **Symptom:** visual goes blank after adding a rule/rect; Vega-Lite says `Cannot read properties of undefined (reading 'type')`.
- **Cause:** a constant `"datum"` channel paired with a `"field"` channel on the same axis, e.g. `"y": {"datum": 0}` + `"y2": {"field": "weekRows"}`.
- **Fix:** turn the constant into a field: `{"calculate": "0", "as": "lineTop"}` and use `"field"` on both channels.

### Shared scales
- Layers that encode the same field on the same channel share one scale. Define `domain`/`range` once (the card circle in LAYER 3); leave the panel circle and key circles bare.
- *Infinite extent for field "..."* warnings come from layers that are empty for the current state (closed day panel). Harmless.

### JSONC
- Deneb 2.x accepts `//` comments; keep them. No trailing commas. No `"$schema"` (a certified visual can't fetch URLs; Deneb offers a Quick Fix).
- Use `\u` escapes for symbols (`"\u00B7"` = ·).
- Don't create params named `width`/`height`; use `denebContainer.*`, not the deprecated `pbiContainer*`.

### Dates
- Dates arrive as date values (midnight local). Use `timeFormat(datum['Date'], '%d %b %Y')` in expressions and `"type": "temporal"` in tooltips. `timeFormat` on a string gives `Invalid Date`.

## Files / tooling

### Desktop and files edited on disk
- Power BI Desktop doesn't reload `.pbip` files edited outside it and will overwrite them on save. Edit TMDL only with Desktop closed. If the project sits in OneDrive/SharePoint, let syncing finish before reopening. If Desktop warns about external changes you didn't make, close without saving and reopen.

### Spec copy inside the report
- Deneb stores the spec as one escaped string in `visual.json`. Don't hand-edit it; edit the `.vl.jsonc` and re-paste into the Deneb editor.

### No local Vega-Lite compiler
- Validate with [assets/preview.html](../assets/preview.html) in a browser (needs cdn.jsdelivr.net, no install), then in Desktop.

## Header controls
Applies when the slicer and Month / List toggle are laid over the calendar header ([powerbi-setup.md](./powerbi-setup.md#header-controls-optional)).

### Shapes show a red (or other) fill
- **Cause:** a shape's **Fill** colour defaults to the theme's first data colour, even when Fill is switched off — and it reappears if someone toggles Fill on.
- **Fix:** set the Fill colour explicitly (to the surface colour) and switch it off; colour shapes with **Effects → Background** only where colour is meant to show.

### Button text clipped or off-centre
- **Cause:** the button's default padding squeezes the text in a 36 px tall button.
- **Fix:** set button **padding to 0** (all states), text 12 pt, vertical alignment middle.

### Slicer shows two borders
- **Cause:** the dropdown's own Values border is drawn inside the visual border.
- **Fix:** **Values → Border = None**; keep only the visual border (radius 8).

### Toggle fill has gaps or a rounded middle edge
- **Cause:** the selected half is a single rounded shape, so its inner edge is rounded too, or it's a few px off the frame.
- **Fix:** use a rounded shape plus a square shape covering the middle side (see the table in powerbi-setup.md), all on whole-pixel positions, same height as the frame.

### Controls move by 1 px after each save
- **Cause:** grouped objects in a group with fractional width/height are rescaled on save.
- **Fix:** keep group positions and sizes whole numbers, or don't group the controls.

### Bookmark resets the month or filters
- **Cause:** the bookmark captured **Data**.
- **Fix:** untick Data, keep Display, and use **Selected visuals** with only the toggled objects selected (calendar, table, buttons, fills) — never the slicer.

## Pre-flight checklist
Run before handing the spec to the user:
- [ ] Theme: BDO default kept, or every theme hex replaced with the user's CVI.
- [ ] `CATEGORY COLOURS` domain matches the real status values exactly (e.g. Red / Amber / Green), ends with `"No status"`, one colour per value.
- [ ] `pKeyItems` = the domain values without `"No status"`; `pKeyTitle` set.
- [ ] Type scale params match the CVI; no fixed `fontSize` numbers added to marks.
- [ ] With header controls: `pTitleMode` = `"note"`, `pControlsPx` matches the slicer's right edge; Deneb visual padding 0.
- [ ] Tooltip arrays in LAYER 3 and LAYER 5 identical; extra fields exist in the well.
- [ ] Every `"field"` / `datum['...']` name is a role name, a calendar column, or created by a transform.
- [ ] Renders in preview.html with the sample data: zero errors; only the expected warnings.
- [ ] Desktop steps list the exact source field → role name mapping.
- [ ] Expected row count for one month noted (≈ 30 + items); under 500.
