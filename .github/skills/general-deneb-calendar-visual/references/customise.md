# Customising the month calendar

Everything here is an edit to the user's copy of [month-calendar.vl.jsonc](../assets/month-calendar.vl.jsonc). Never edit the skill's asset copy.

## Theme (CVI)
The spec ships with neutral placeholder colours. **The user chooses the CVI** — ask for one of:
- a Power BI theme file (`.json`), or
- hex codes for the roles below plus a font, or
- "keep the placeholder".

Never invent brand colours. If the user gives only a few colours, ask which role each one is for.

Every placeholder hex is used for one role only, so replace each one **everywhere it appears** (find/replace on the hex):

| Placeholder | Role | Typical source in a Power BI theme JSON |
|---|---|---|
| `#FFFFFF` | Surface: day cells, day panel, legend background | `background` |
| `#F5F5F5` | Out-of-month cells | `backgroundLight` / a light neutral |
| `#E1DFDD` | Lines: grid, outer border, legend border | `tableAccent` or a light neutral |
| `#EFF6FC` | Today's cell fill | light tint of the accent |
| `#0F6CBD` | Accent: today's number, open-day outline, "+N more" | first `dataColors` entry or brand primary |
| `#252423` | Strong text: title, card text, panel title, legend title | `foreground` |
| `#323130` | Body text: day numbers, legend labels | `foreground` / `foregroundNeutralSecondary` |
| `#605E5C` | Muted text: weekday header | `foregroundNeutralSecondary` |
| `#8A8886` | Panel border, "Click outside to close" | `foregroundNeutralTertiary` |
| `#A19F9D` | Outline on cards with no category | neutral grey |
| `#D13438` | Flag icon (⚠) and its legend symbol | `bad` / `negative` sentiment colour |
| `Segoe UI` | Font (`config.font`) | `textClasses` font family |

Category colours are separate — see below.

Fonts: Deneb is a certified visual and can't load web fonts. Use a font installed on viewers' machines (Segoe UI, Arial, etc.); otherwise the browser falls back silently.

## Category colours
Defined once, in LAYER 3 under `CATEGORY COLOURS`:
```jsonc
"scale": {
  "domain": ["Meeting", "Deadline", "Release", "Training", "No category"],
  "range":  ["#4C6A92", "#D9822B", "#5B8C5A", "#8E6C8A", "#C8C6C4"]
}
```
1. Get the real values. For a `.pbix` (or any report) ask the user to run in **DAX query view**:
   ```dax
   EVALUATE DISTINCT ( '<Item Table>'[<Category Column>] )
   ```
2. Put every value in `domain` **exactly** as returned (case, spaces). Keep `"No category"` last (it's the label for blanks; it must match the text in the shared transforms).
3. Put one CVI colour per value in `range`, same order. Values left out of the domain get recycled colours, and colours would no longer match between months.
4. The legend title is `"Category"` on both the card rect and the panel dot (LAYER 5). Rename both together — they must stay identical to merge into one legend.
5. If native visuals on the page should use the same colours, suggest a DAX colour column (`SWITCH(...)` returning hex) and tell the user to keep both in sync.

### No category in the report
Leave `Item Category` out of the Values well. Then either keep it simple (every card is grey with an outline, legend shows "No category"), or for one solid colour:
- LAYER 3 card rect: replace the whole `"color": {...}` with `"color": {"value": "<hex>"}` and the `"stroke"` with `"stroke": {"value": "transparent"}`.
- LAYER 5 panel dot: replace `"color": {...}` with `"color": {"value": "<hex>"}`.
- Remove the `{"field": "categoryLabel", ...}` tooltip rows in LAYERS 3 and 5.

## Flag (⚠)
- The rule lives in the model, not the spec. Give the user a calculated column on the item table:
  ```dax
  Item Flag = IF ( <condition, e.g. 'Items'[Status] = "Red">, 1, 0 )
  ```
  Use a **column**, not a measure (see [gotchas](./gotchas.md#measures-in-the-values-well)).
- Rename the legend entry: change `"Flagged"` in LAYER 6 (it appears twice: the `calculate` and the `domain`).
- No flag in the report: leave `Item Flag` out of the well and delete LAYER 6. The icon layers simply draw nothing.

## Order of items within a day
Set by `sortKey` in the shared transforms; every layer (cards, "+N more", panel) uses it.
- Alphabetical (default): `"datum['Item Label']"`
- Flagged first: `"(datum.isFlagged ? '0' : '1') + datum['Item Label']"`
- By another field: add it to the well and use it, e.g. `"datum['Start Time']"` (pad numbers or use text that sorts correctly).

## Wording
| Text | Where |
|---|---|
| "item" / "items" in the panel title | `pItemNoun`, `pItemNounPlural` params |
| Tooltip titles ("Item", "Category", "Date") | `tooltip` arrays in LAYER 3 **and** LAYER 5 |
| "Also on this day" | LAYER 4 tooltip |
| "Click outside to close" | LAYER 5 |
| "Current Month" in the title | `title.text` expression (the slicer's "Current Month" label is in the DAX) |
| "Flagged" | LAYER 6 |
| "No category" | shared transforms + the end of the category `domain` |
| Weekday header | `weekdayName` in the WEEKDAY HEADER layer |
| Date formats | `"format": "%d %b %Y"` (tooltips), `'%a %d %b'` (panel title) — d3 time format |

## Extra tooltip fields
1. Add the field to the Values well. It must have **one value per item** (e.g. owner, status); a field with several values per item duplicates the card.
2. Give it a name that doesn't clash with the role names.
3. Add `{"field": "<name>", "title": "<Title>"}` to the `tooltip` arrays in LAYER 3 **and** LAYER 5. Dates need `"type": "temporal", "format": "%d %b %Y"`.

## Size and density
| Param | Default | Effect |
|---|---|---|
| `pMaxCards` | 3 | Cards per day before "+N more". Lower for short visuals. |
| `pHeaderH` | 0.22 | Share of the cell for the day-number strip. |
| `pMoreH` | 0.03 | Margin under the last card. |
| `pCardGap` | 0.02 | Gap between cards. |
| `pPadX` | 0.04 | Card inset from the cell edges. |
| `pCardTint` | 0.2 | Card background opacity. |
| `pPanelRowPx` | 22 | Day panel row height (px). |
| `pPanelCols` | 2.6 | Day panel width in day columns. |

Font sizes are on the marks (`fontSize`). Make the visual taller in the report for bigger cells.

## Not renaming fields in the well
Renaming in the Values well is the default because the spec then needs no field edits. If the user can't or won't rename, replace the role names in their copy of the spec instead: `'Item Key'`, `'Item Label'`, `'Item Category'`, `'Item Flag'` (both `"field": "..."` and `datum['...']` forms). Then re-check every name against the model (pre-flight checklist in [gotchas.md](./gotchas.md#pre-flight-checklist)).

## Monday-first weeks
Weeks run Sunday–Saturday by default. For Monday–Sunday change three things together:
- DAX `Weekday Number`: `WEEKDAY ( [Date], 2 ) - 1`
- DAX `Week Row`: use `WEEKDAY ( DATE ( YEAR ( [Date] ), MONTH ( [Date] ), 1 ), 2 )` inside the formula
- Spec WEEKDAY HEADER `weekdayName`: `['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun']`
