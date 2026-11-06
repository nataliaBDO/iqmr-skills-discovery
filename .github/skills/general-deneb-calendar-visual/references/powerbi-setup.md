# Power BI Desktop setup for the month calendar

Give the user these steps as numbered instructions, filled in with their table and field names. Mirror the final list in the header comment of their spec copy.

## 0. Prerequisites
- Deneb 2.x in the report: **Visualizations → … (Get more visuals) → AppSource → "Deneb"**, or from organisational visuals if IT publishes it there.
- The `'Deneb Calendar'` table, its relationship to the item date, and any `Item Flag` column exist ([model-prep.md](./model-prep.md)).

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
   | `'<Item Table>'[<category>]` (optional) | `Item Category` |
   | `'<Item Table>'[Item Flag]` (optional) | `Item Flag` |
   | extra tooltip fields (optional) | any unique name, see [customise.md](./customise.md#extra-tooltip-fields) |

3. Rename: in the Values well, double-click the field (or its dropdown → **Rename for this visual**) and type the role name exactly, including the space and capitals. Deneb's dataset uses the name shown in the well.
4. Every numeric field: dropdown → **Don't summarize** (Month Key, Week Row, Weekday Number, Day of Month, Is Today, Item Flag, numeric keys).
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

## 4. Page
1. Add a **slicer** on `'Deneb Calendar'[Month Selector]`, single select, and select **Current Month**. Because the current month is always labelled "Current Month", the saved selection rolls forward each month.
2. Optional: a native table of the month's items next to the calendar. Clicking a day or card in the calendar filters it.
3. Optional: a table of items with no date (they can't appear on the grid).

## 5. Test
- Today's cell is tinted; the title shows the month and "· Current Month".
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
