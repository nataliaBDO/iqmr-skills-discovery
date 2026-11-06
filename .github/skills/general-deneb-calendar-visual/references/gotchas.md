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
- **Fix:** use calculated **columns** for `Item Flag` and tooltip extras. If a measure is unavoidable, make it return `BLANK()` when there's no item.

### Identical rows merge
- Power BI groups by all non-aggregated fields: two items with the same label, category and date become one card. Always add `Item Key`.
- Numeric columns default to **Sum** → set **Don't summarize** for every calendar column, `Item Flag` and numeric keys. A summed `Week Row` puts cards in the wrong row.

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
- **Symptom:** a layer draws nothing, a tooltip row is empty, every card is "No category" — no error.
- **Cause:** a name in the spec doesn't exactly match the dataset; Vega-Lite treats a missing field as `undefined` silently.
- **Fix:** check the role names in the Values well (case and spaces: `Item Label`, not `Item label`). The dataset uses the name shown in the well. Confirm in debug pane → **Source**.

### Category colours wrong or shifting
- **Cause:** a category value in the data isn't in the `CATEGORY COLOURS` domain (spelling, case, trailing space), so it gets a recycled colour.
- **Fix:** rerun `EVALUATE DISTINCT(...)` and copy the values exactly ([customise.md](./customise.md#category-colours)).

### Legend disappeared
- **Cause:** `"legend": null` on the card colour or the panel dot colour. They share the field `categoryLabel` and title, so Vega-Lite merges them; `null` on either hides both.
- **Fix:** never put `legend: null` on those two. To hide the category legend completely, set `"legend": null` on **both**, or use `config.legend.disable: true` (also hides the flag legend).

### Axis missing in a layered chart
- **Symptom:** an axis (e.g. a weekday header) set on one layer doesn't show.
- **Cause:** layers sharing the scale merge their axis settings; `"axis": null` on the other layers can win and hide it.
- **Fix:** draw headers/labels as a text layer (as the WEEKDAY HEADER does) instead of relying on an axis.

### datum + field crash
- **Symptom:** visual goes blank after adding a rule/rect; Vega-Lite says `Cannot read properties of undefined (reading 'type')`.
- **Cause:** a constant `"datum"` channel paired with a `"field"` channel on the same axis, e.g. `"y": {"datum": 0}` + `"y2": {"field": "weekRows"}`.
- **Fix:** turn the constant into a field: `{"calculate": "0", "as": "lineTop"}` and use `"field"` on both channels.

### Shared scales
- Layers that encode the same field on the same channel share one scale. Define `domain`/`range` once (the card rect); leave the panel dot bare.
- *Infinite extent for field "..."* warnings come from layers that are empty for the current state (closed day panel, the flag legend layer). Harmless.

### JSONC
- Deneb 2.x accepts `//` comments; keep them. No trailing commas. No `"$schema"` (a certified visual can't fetch URLs; Deneb offers a Quick Fix).
- Use `\u` escapes for symbols (`"\u26A0"` = ⚠, `"\u00B7"` = ·).
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

## Pre-flight checklist
Run before handing the spec to the user:
- [ ] Theme tokens replaced with the user's CVI (or the user agreed to keep the placeholder).
- [ ] `CATEGORY COLOURS` domain matches the real category values exactly, ends with `"No category"`, one colour per value.
- [ ] LAYER 6 deleted if there's no `Item Flag`; "Flagged" renamed otherwise.
- [ ] Tooltip arrays in LAYER 3 and LAYER 5 identical; extra fields exist in the well.
- [ ] Every `"field"` / `datum['...']` name is a role name, a calendar column, or created by a transform.
- [ ] Renders in preview.html with the sample data: zero errors; only the expected warnings.
- [ ] Desktop steps list the exact source field → role name mapping.
- [ ] Expected row count for one month noted (≈ 30 + items); under 500.
