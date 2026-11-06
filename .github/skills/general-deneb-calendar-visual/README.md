# general-deneb-calendar-visual

A Copilot agent skill that builds a **month calendar** for any Power BI report using the Deneb custom visual: dated items on a Sunday–Saturday grid, coloured by category, with an optional ⚠ flag, "+N more" when a day is full, and a panel listing every item when you click a day.

It works with any report because the visual uses fixed role names (`Item Key`, `Item Label`, `Item Category`, `Item Flag`). You rename your own fields to those names in the Deneb visual, so the visual itself doesn't need field edits.

## Install
1. Copy the whole `general-deneb-calendar-visual` folder (don't rename it) into your workspace at `.github/skills/general-deneb-calendar-visual/`.
   Or keep it anywhere and add its **parent** folder to the VS Code setting `chat.agentSkillsLocations`.
2. Run **Developer: Reload Window**.
3. In Copilot Chat type `/` and check `general-deneb-calendar-visual` is listed.

## Use
Type `/general-deneb-calendar-visual` and describe what you want, e.g.:
- `/general-deneb-calendar-visual build a calendar of tasks by due date for C:\Reports\Team Tracker.pbip`
- "I have a .pbix with an Events table - make me a Deneb month calendar of events coloured by event type"
- "Re-theme my Deneb calendar with this theme file" (attach your Power BI theme `.json`)

Have ready:
- The report (`.pbip` folder, or a `.pbix` — the agent will ask you to run a DAX query and paste the result, because it can't read inside a `.pbix`).
- Which table and date the items come from, what to show on each card, and optionally a category for colour and a rule for the ⚠ flag.
- Your **colour theme**: a Power BI theme `.json` or hex codes. Without one you get neutral greys.

You get back (your report is never edited):
- DAX for a `'Deneb Calendar'` table and steps to add it in Power BI Desktop.
- A commented spec in a `deneb/` folder next to your report, themed for you.
- Sample data and a browser preview to check it.
- Numbered Power BI Desktop steps.

## Requirements
- Power BI Desktop with the **Deneb** visual (free, certified, from AppSource).
- Internet access to cdn.jsdelivr.net for the browser preview only.
