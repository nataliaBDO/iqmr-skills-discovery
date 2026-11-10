# Power BI Desktop setup for the month calendar

Give the user these steps as numbered instructions, filled in with their table and field names. Mirror the final list in the header comment of their spec copy.

## 0. Prerequisites
- Deneb 2.x in the report: **Visualizations → … (Get more visuals) → AppSource → "Deneb"**, or from organisational visuals if IT publishes it there.
- The `'Deneb Calendar'` table and its relationship to the item date exist ([model-prep.md](./model-prep.md)).

## 1. Add the visual and fields
1. Add an empty Deneb visual and size it roughly as it will be on the page (wide and tall; ~950 × 650 px works well).
2. Drag these into **Values**:

   | Field | Rename in the well to |
   |---|---|
   | `'Deneb Calendar'[Date]` | (keep) `Date` |
   | `'Deneb Calendar'[Month Key]` | (keep) |
   | `'Deneb Calendar'[Week Row]` | (keep) |
   | `'Deneb Calendar'[Weekday Number]` | (keep) |
   | `'Deneb Calendar'[Day of Month]` | (keep) |
   | `'Deneb Calendar'[Is Today]` | (keep) |
   | `'<Item Table>'[<unique ID>]` | `Item Key` |
   | `'<Item Table>'[<name/title>]` | `Item Label` |
   | `'<Item Table>'[<RAG status>]` (optional) | `Item Category` |
   | extra tooltip fields (optional) | any unique name, see [customise.md](./customise.md#extra-tooltip-fields) |

3. Rename: in the Values well, double-click the field (or its dropdown → **Rename for this visual**) and type the role name exactly, including the space and capitals. Deneb's dataset uses the name shown in the well.
4. Every numeric field: dropdown → **Don't summarize** (Month Key, Week Row, Weekday Number, Day of Month, Is Today, numeric keys).
5. Right-click `Date` in the well → **Show items with no data** (otherwise empty days leave holes in the grid).
6. Don't add the item table's own date column — the calendar `Date` already places each item.

## 2. Create the specification
1. Visual header **… → Edit** (opens the Deneb editor).
2. **Create new specification → Vega-Lite → Empty**. Don't use a template — it would try to remap fields.
3. Select all in the **Specification** editor, paste the whole `.vl.jsonc` (comments included), **Apply** (Ctrl+Enter). Leave the **Config** tab as `{}`; the theme lives in the spec's `config`.
4. If Deneb shows a `$schema` warning, use its Quick Fix to remove it.

## 3. Settings pane
| Section | Setting | Value |
|---|---|---|
| Interactivity | **Tooltips** | On |
| Interactivity | **Expose cross-filtering values for dataset rows** | On (card click filters the page; non-selected cards dim) |
| Interactivity | Cross-highlight | Off |
| Continuous view | **Enable patching for hosted datasets** | On — without it the day panel closes as soon as a card click cross-filters the page. Check the section says patching is **active** (it switches off above 500 rows by default). |

Then in the visual's **Format → General**: **Properties → Padding = 0** on all sides (the header row is positioned from the visual's top edge), and optionally **Effects → Visual border** on with the same corner radius as the page's other cards.

## 4. Page
1. Add a **slicer** on `'Deneb Calendar'[Month Selector]`, single select, and select **Current Month**. Because the current month is always labelled "Current Month", the saved selection rolls forward each month. Put it anywhere — or inside the calendar's header (next section).
2. Optional: a native table of the month's items next to the calendar, or swapped in with a Month / List toggle (next section). Clicking a day or card in the calendar filters it.
3. Optional: a table of items with no date (they can't appear on the grid).

## Header controls (optional)
Makes the calendar, the month picker and a Month / List switch look like one visual. All controls are native Power BI objects laid over the Deneb visual's header row — they are not in the spec, so the browser preview never shows them (it shows the header row with an empty space on the left when `pTitleMode` is `"note"`). Example: [images/header-controls-example.svg](../images/header-controls-example.svg). Positions below are relative to the Deneb visual's top-left corner **(X, Y)**; its header line is at **Y + 30**, and every control is **36 px tall at Y + 12**.

| Object | Position / size | Format |
|---|---|---|
| Toggle frame (shape) | X+12, Y+12, 168 × 36 | **Effects → Background** on (surface colour); **Visual border** on (lines colour), radius **18** (= half the height → pill) |
| Month fill – round end (shape) | X+12, Y+12, 84 × 36 | Background + visual border both the dark/accent colour, radius 18 |
| Month fill – square side (shape) | X+74, Y+12, 22 × 36 | Background the same colour, no border (squares off the middle edge) |
| List fill – round end (shape) | X+96, Y+12, 84 × 36 | as Month round end |
| List fill – square side (shape) | X+96, Y+12, 22 × 36 | as Month square side |
| Month button × 2, List button × 2 | Month: X+12, List: X+96; each 84 × 36 | Fill off, outline off, icon off, padding 0; text 12 pt. **Active** copy: white bold text. **Inactive** copy: dark text. Action = Bookmark (inactive copy only) |
| Month Selector slicer (the title) | X+192, Y+12, 240 × 36 | Dropdown, single select; slicer header off; title off; Values text 12 pt; **Values border = None**; visual border on (lines colour, radius 8); background on (surface) |
| List table | same position and size as the Deneb visual, **hidden** | Title off; top padding ≈ 60 so the header row stays clear; same visual border |

Result: the outside of the toggle is a pill, the selected half is filled edge to edge (no gap at top/bottom), and the middle edge between Month and List is straight.

In the spec copy set `pTitleMode` to `"note"` and `pControlsPx` to `(slicer right edge − X) − 5 + 8` (= 435 for the layout above); the month name then appears after the slicer only when it says "Current Month". The key stays in the top-right.

**Bookmarks** (View → Bookmarks + Selection pane):
1. Show the Deneb visual, the Month fills and the active Month / inactive List buttons; hide the table, the List fills and the other two buttons. Add bookmark **"Calendar – Month view"**.
2. Swap visibility for the List view. Add **"Calendar – List view"**.
3. For both bookmarks (… menu): untick **Data** (so the slicer and filters are not captured), keep **Display**, tick **Selected visuals** and make sure the selection covered the calendar, table, 4 buttons and 4 fill shapes only. The frame and the slicer stay out — they're visible in both views.
4. Point the inactive List button at the List view bookmark and the inactive Month button at the Month view bookmark. Save the page in Month view.
5. In Desktop, Ctrl+click the buttons to test; in the Service a normal click works.

Tips: name everything in the Selection pane ("Toggle frame", "Month fill – round end", …); see [gotchas.md → Header controls](./gotchas.md#header-controls) for the usual formatting traps.

## 5. Test
- Today's cell is tinted; the header shows the month title (or, with header controls, the month name next to the slicer when it says "Current Month") and the boxed key in the top-right.
- Key entries don't overlap and are evenly spaced (adjust `pCharW` if not); with header controls, the slicer, toggle and key share one centre line.
- Days with more than `pMaxCards` items show "+N more"; hover lists the hidden names.
- Click a day or "+N more" → panel lists every item; click outside → it closes.
- Click a card → page cross-filters; other cards dim; the panel stays open.
- Change the slicer to another month; clear it (the grid still shows one month — the earliest).
- Resize the visual: the grid always fits.

## 6. Debug pane (in the Deneb editor)
| Tab | Shortcut | Use |
|---|---|---|
| Source | Ctrl+Alt+6 | Rows as Deneb receives them — **confirm the role names and nulls here** |
| Data | Ctrl+Alt+7 | Datasets after transforms |
| Signals | Ctrl+Alt+8 | `expandedDay`, `p*` params, `width`/`height` |
| Logs | Ctrl+Alt+9 | Compile warnings/errors, continuous-view fallbacks |

## 7. Save
Save in Desktop. Keep the `.vl.jsonc` source in sync with what's in the Deneb editor — if the user changes the spec in Deneb, ask them to paste it back.

## What lands in PBIR (`.pbip` only; don't hand-edit)
| JSON path in `visual.json` | Meaning |
|---|---|
| `visual.visualType` | `deneb7E15AEF80B9E4D4F8E12924291ECE89A` |
| `visual.query.queryState.dataset.projections[]` | Fields in Values; `displayName` = the rename |
| `visual.query.queryState.dataset.showAll` | Show items with no data |
| `visual.objects.vega[0].properties.jsonSpec` | The spec, one escaped string |
| `visual.objects.vega[0].properties.enableTooltips` / `enableSelection` | Interactivity settings |
| `visual.objects.dataLimit[0].properties.enableIncrementalDataUpdates` | Continuous view patching |

## Service notes
- `TODAY()` in the calendar table evaluates at refresh (UTC in the Service): "Is Today" and "Current Month" move only when the dataset refreshes. Schedule a daily refresh.
