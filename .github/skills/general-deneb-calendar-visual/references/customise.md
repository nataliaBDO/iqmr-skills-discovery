# Customising the month calendar

Everything here is an edit to the user's copy of [month-calendar.vl.jsonc](../assets/month-calendar.vl.jsonc). Never edit the skill's asset copy.

## Theme (CVI)
The spec ships with the **BDO theme** (BDOADVANTAGE theme colours, Trebuchet MS). To use another CVI, ask for one of:
- a Power BI theme file (`.json`), or
- hex codes for the roles below plus a font, or
- "keep the BDO theme".

Never invent brand colours. If the user gives only a few colours, ask which role each one is for.

Every theme hex is used for one role only, so replace each one **everywhere it appears** (find/replace on the hex):

| BDO default | Role | Typical source in a Power BI theme JSON |
|---|---|---|
| `#FFFFFF` | Surface: day cells, item cards, day panel, key box | `background` |
| `#F7F6F7` | Out-of-month cells | `backgroundLight` |
| `#E7E7E7` | Lines: grid, outer border, card border, key border | a light neutral (BDO card border) |
| `#E6F6FC` | Today's cell fill | light tint of the accent |
| `#1183B1` | Accent: today's number, open-day outline, "+N more" | a brand primary that is **not** a status colour (BDO dark ocean) |
| `#404040` | Titles: month title, panel title, key title | `foreground` |
| `#333333` | Body text: day numbers, card and panel labels, key labels | body text colour |
| `#5B6E7F` | Secondary text: weekday header, month note | secondary / axis label colour (BDO slate) |
| `#828282` | Panel border, "Click outside to close" | mid grey |
| `#B2B2B2` | "No status": hollow circle outline and its domain colour | neutral grey |
| `Trebuchet MS` | Font (`config.font`) | `textClasses` font family |

Status colours are separate — see below. Font sizes are in the TYPE SCALE params — see [Type scale](#type-scale).

Fonts: Deneb is a certified visual and can't load web fonts. Use a font installed on viewers' machines (Trebuchet MS, Segoe UI, Arial, etc.); otherwise the browser falls back silently.

## Status colours (RAG)
Cards are plain (surface + border); the item's status shows as a small circle on each card, in the day panel and in the key ("Status ● Red ● Amber ● Green"). The status comes from the `Item Category` role. Defined once, in LAYER 3 under `CATEGORY COLOURS` (the circle's `color` encoding):
```jsonc
"scale": {
  "domain": ["Red", "Amber", "Green", "No status"],
  "range":  ["#E81A3B", "#FD7E14", "#54A575", "#B2B2B2"]
}
```
1. Get the real values. For a `.pbix` (or any report) ask the user to run in **DAX query view**:
   ```dax
   EVALUATE DISTINCT ( '<Item Table>'[<Status Column>] )
   ```
2. Put every value in `domain` **exactly** as returned (case, spaces) — e.g. if the data says `"R"`/`"A"`/`"G"` or `"On track"`, either use those values or add a DAX column that maps them to `Red`/`Amber`/`Green`. Keep `"No status"` last (it's the label for blanks; it must match the text in the shared transforms). Items with no status get a **hollow** circle (outline `#B2B2B2`).
3. One colour per value in `range`, same order. Values left out of the domain get recycled colours, and colours would no longer match between months.
4. Copy the same values (same order, **without** `"No status"`) into the `pKeyItems` param — that's the list the key shows. Both lists must match.
5. If native visuals on the page should use the same colours, suggest a DAX colour column (`SWITCH(...)` returning hex) and tell the user to keep both in sync.

Other categories (event type, team…) work the same way: replace the domain, range, `pKeyItems` and `pKeyTitle`.

### No status in the report
Leave `Item Category` out of the Values well. Every card then gets a hollow circle. Also:
- Set `pKeyItems` to `[]` and hide the key — see [Header key](#header-key).
- Or for one solid circle colour: LAYER 3 circle and LAYER 5 panel circle — replace `"color": {...}` with `"color": {"value": "<hex>"}` and remove the `fillOpacity` / `strokeWidth` conditions.
- Remove the `{"field": "categoryLabel", ...}` tooltip rows in LAYERS 3 and 5.

## Header key
The key is a boxed row in the top-right of the header (white box, `#E7E7E7` border, 8 px corners), drawn by the HEADER ROW layer — the built-in Vega-Lite legend is off (`config.legend.disable`).
| Param | Default | Meaning |
|---|---|---|
| `pKeyTitle` | `"Status"` | Bold caption at the left of the key |
| `pKeyItems` | `["Red", "Amber", "Green"]` | One circle + label each, coloured by `CATEGORY COLOURS` |
| `pCharW` | `0.55` | Estimated character width (share of the font size). Vega-Lite can't measure text, so entries are spaced from this. Raise it if a label touches the next circle; lower it if gaps look too wide. |

The key sits flush with the grid's right edge, centred on the header line (58 px above the grid). To hide it completely, delete the second sub-layer ("KEY") of the HEADER ROW layer.

## Month title
| `pTitleMode` | What's drawn at the left of the header row |
|---|---|
| `"full"` (default) | Bold month title at `pFontTitle`, plus "· Current Month" when it's the current month |
| `"note"` | Nothing, except the month name (muted, `pFontKey`) when it's the current month. Use when a Month Selector slicer is laid over the header as the title — the slicer only says "Current Month", so the note names it. |

`pControlsPx` = px from the grid's left edge where the title / note starts (0 = flush left). Set it to the right edge of any Power BI controls laid over the header (slicer, Month / List toggle) plus a small gap — see [powerbi-setup.md → Header controls](./powerbi-setup.md#header-controls-optional).

## Type scale
Every font size comes from four params (px; 1 pt = 1.333 px), defaulting to a 16 / 12 / 10 / 9 pt scale:
| Param | Default | Used for |
|---|---|---|
| `pFontTitle` | 21.33 (16 pt) | Month title (`pTitleMode` "full") |
| `pFontKey` | 16 (12 pt) | Key title and labels, month note |
| `pFontBody` | 13.33 (10 pt) | Weekday header, day numbers, card labels, panel title and rows |
| `pFontSmall` | 12 (9 pt) | "+N more", "Click outside to close" |

If the user's CVI has a type scale, convert its pt sizes to px and set these four — don't add `fontSize` numbers on individual marks. Match the Power BI side too: slicer and button text in the header at 12 pt, table text at 10 pt.

## Order of items within a day
Set by `sortKey` in the shared transforms; every layer (cards, "+N more", panel) uses it.
- Alphabetical (default): `"datum['Item Label']"`
- Red first: `"indexof(['Red', 'Amber', 'Green'], datum.categoryLabel) + datum['Item Label']"` (items with no status sort first)
- By another field: add it to the well and use it, e.g. `"datum['Start Time']"` (pad numbers or use text that sorts correctly).

## Wording
| Text | Where |
|---|---|
| "item" / "items" in the panel title | `pItemNoun`, `pItemNounPlural` params |
| Tooltip titles ("Item", "Status", "Date") | `tooltip` arrays in LAYER 3 **and** LAYER 5 |
| "Also on this day" | LAYER 4 tooltip |
| "Click outside to close" | LAYER 5 |
| "Current Month" in the title | HEADER ROW layer, `titleText` (the slicer's "Current Month" label is in the DAX) |
| Key caption | `pKeyTitle` |
| "No status" | shared transforms + the end of the status `domain` |
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
| `pPanelRowPx` | 22 | Day panel row height (px). |
| `pPanelCols` | 2.6 | Day panel width in day columns. |

Font sizes: see [Type scale](#type-scale). Make the visual taller in the report for bigger cells.

## Not renaming fields in the well
Renaming in the Values well is the default because the spec then needs no field edits. If the user can't or won't rename, replace the role names in their copy of the spec instead: `'Item Key'`, `'Item Label'`, `'Item Category'` (both `"field": "..."` and `datum['...']` forms). Then re-check every name against the model (pre-flight checklist in [gotchas.md](./gotchas.md#pre-flight-checklist)).

## Monday-first weeks
Weeks run Sunday–Saturday by default. For Monday–Sunday change three things together:
- DAX `Weekday Number`: `WEEKDAY ( [Date], 2 ) - 1`
- DAX `Week Row`: use `WEEKDAY ( DATE ( YEAR ( [Date] ), MONTH ( [Date] ), 1 ), 2 )` inside the formula
- Spec WEEKDAY HEADER `weekdayName`: `['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun']`
