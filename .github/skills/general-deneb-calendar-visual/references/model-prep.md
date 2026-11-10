# Model prep for the month calendar

## What the calendar needs from the model
| Need | Typical source | Notes |
|---|---|---|
| One date per item | an existing date column on the item table | Date-only. Times break the relationship (see below). |
| Unique item ID | GUID / ID / number column | Without it, two items with the same label on the same day merge into one card. If no ID exists, an index column added in Power Query works. |
| Item label | name / title column | Text on the card. |
| Category (optional) | status / type / phase column | Card colour. One value per item. |
| Calendar grid columns | `'Deneb Calendar'` table | [assets/calendar-table.dax](../assets/calendar-table.dax) |

## Reading field names
Field names must be exact. Never guess them.
- **`.pbip`** (read only): `<Name>.pbip` → `<Name>.Report/definition.pbir` (`datasetReference.byPath.path`) → `<Name>.SemanticModel/definition/tables/*.tmdl`. Grep for `column ` and `measure ` lines; note `dataType`, `summarizeBy`, `formatString`, `///` descriptions. Ignore `LocalDateTable_*` / `DateTableTemplate_*`.
- **`.pbix`** (binary — don't open or unzip it): ask the user to run these in Desktop's **DAX query view** and paste the results:
  ```dax
  EVALUATE INFO.VIEW.COLUMNS()
  EVALUATE INFO.VIEW.RELATIONSHIPS()
  ```
  Alternatives: a screenshot of the Data pane or a pasted field list.
- Category values (for the colour domain), either format:
  ```dax
  EVALUATE DISTINCT ( '<Item Table>'[<Category Column>] )
  ```

## Add it in Desktop (default)
Works for `.pbix` and `.pbip`. Give the user the DAX and these steps; don't touch their files.

| Change | Desktop steps |
|---|---|
| Calendar table | **Modeling → New table**, paste [calendar-table.dax](../assets/calendar-table.dax) from `Deneb Calendar =` with the two placeholders filled in |
| Sort the month columns | Select `Month Selector` → **Column tools → Sort by column → Month Key**; same for `Month Label` |
| Don't summarize | Select each numeric calendar column → **Column tools → Summarization → Don't summarize** |
| Relationship | **Modeling → Manage relationships → New**: `'<Item Table>'[<Item Date>]` (many) → `'Deneb Calendar'[Date]` (one), single direction, active |
| Calculated column | Select the item table → **Modeling → New column**, paste `Column Name = <DAX>` |

Then ask the user to confirm the new table/columns appear in the Data pane and the relationship line shows in Model view before writing the spec.

### Calendar table name
`'Deneb Calendar'` avoids clashing with an existing `Calendar`/`Date` table. Don't reuse the report's own date table: it lacks the grid columns, and adding them there changes a shared table. Keep `'Deneb Calendar'` unrelated to other date tables. If Desktop creates the relationship as **inactive**, see [gotchas](./gotchas.md#inactive-relationship).

### Item date with times
If the date column holds times (e.g. `05/11/2026 14:30`), it won't match the calendar's midnight dates and items disappear. Add a date-only column and relate that instead:
```dax
Item Date Only = DATE ( YEAR ( '<Item Table>'[<Item Date>] ), MONTH ( '<Item Table>'[<Item Date>] ), DAY ( '<Item Table>'[<Item Date>] ) )
```
Point the calendar table's `MIN`/`MAX` at the same column.

### Items that span several days
The calendar places each item on **one** date. For a start/end range the model needs one row per item per day — a bridge table, e.g.:
```dax
Item Days =
GENERATE (
    SELECTCOLUMNS ( '<Item Table>', "Item ID", '<Item Table>'[<ID>], "Start", '<Item Table>'[<Start>], "End", '<Item Table>'[<End>] ),
    DATESBETWEEN ( 'Deneb Calendar'[Date], [Start], [End] )
)
```
then relate `'Item Days'[Date]` → `'Deneb Calendar'[Date]` and `'Item Days'[Item ID]` → item table. Check the row count stays reasonable ([gotchas](./gotchas.md#continuous-view-500-row-limit)). Confirm the design with the user before going this way.

## Editing TMDL files (only when the user explicitly asks)
`.pbip` only, with Power BI Desktop **closed** (it doesn't reload external edits and overwrites them on save). If the project is in OneDrive/SharePoint, let syncing finish first.

### File chain
```
<Name>.pbip                                         → <Name>.Report
<Name>.Report/definition.pbir                       datasetReference.byPath → ../<Name>.SemanticModel
<Name>.SemanticModel/definition/model.tmdl          `ref table` list
<Name>.SemanticModel/definition/tables/<Table>.tmdl one file per table
<Name>.SemanticModel/definition/relationships.tmdl  all relationships
```

### Syntax rules
- Indent with **tabs**. Properties one tab deeper than their object; multi-line DAX one tab deeper than the properties.
- Quote names with spaces: `'Deneb Calendar'`, `'Week Row'`.
- Every table/column needs a unique `lineageTag` GUID: `1..12 | ForEach-Object { [guid]::NewGuid() }` in a terminal.
- Columns returned by a calculated table's expression use `isNameInferred` + `sourceColumn: [Name]` (no `=`).
- `summarizeBy: none` for every numeric calendar column; `sortByColumn: 'Month Key'` on `Month Selector` and `Month Label`.

### `tables/Deneb Calendar.tmdl`
Repeat the column block for each column. Types: `Date`, `Month Start` → `dateTime` (`formatString: Short Date`); `Month Key`, `Day of Month`, `Weekday Number`, `Week Row`, `Is Today` → `int64` (`formatString: 0`); `Month Label`, `Month Selector` → `string`.
```tmdl
/// Calendar grid for the Deneb month calendar.
table 'Deneb Calendar'
	lineageTag: <guid>

	column Date
		dataType: dateTime
		formatString: Short Date
		lineageTag: <guid>
		summarizeBy: none
		isNameInferred
		sourceColumn: [Date]

		annotation SummarizationSetBy = User

	column 'Week Row'
		dataType: int64
		formatString: 0
		lineageTag: <guid>
		summarizeBy: none
		isNameInferred
		sourceColumn: [Week Row]

		annotation SummarizationSetBy = User

	column 'Month Selector'
		dataType: string
		lineageTag: <guid>
		summarizeBy: none
		isNameInferred
		sourceColumn: [Month Selector]
		sortByColumn: 'Month Key'

		annotation SummarizationSetBy = Automatic

	partition 'Deneb Calendar' = calculated
		mode: import
		source =
				<the DAX from calendar-table.dax, without the "Deneb Calendar =" line, indented 4 tabs>
```
Add `ref table 'Deneb Calendar'` to `model.tmdl` with the other `ref table` lines. Calculated tables don't go in the `PBI_QueryOrder` annotation.

### Relationship (append to `relationships.tmdl`)
```tmdl
relationship <guid>
	fromColumn: '<Item Table>'.'<Item Date>'
	toColumn: 'Deneb Calendar'.Date
```
`fromColumn` = many side. Both columns `dateTime`.

### After editing
1. User reopens the `.pbip`. A TMDL error blocks opening with a file/line message — fix and retry.
2. Model view: relationship line present and active.
3. DAX query view: `EVALUATE TOPN ( 40, 'Deneb Calendar' )` to eyeball the columns.
