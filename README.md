# NESST-tree-watering

The backend google sheet used to produce the map can be found here: https://docs.google.com/spreadsheets/d/17wqRM2MQVLbRcZiMKTLVl_Ypx7yZcrtEYcGHoowHypg/edit?usp=sharing

## How it fits together

- `Map-script` is the Google Apps Script bound to the sheet. NFC tags open its web app URL with `?id=<tree id>` to log a watering. `?view=data` returns the map data as JSON.
- `index.html` is the public map, served by GitHub Pages at https://nesst-trees.github.io/ (from the nesst-trees/nesst-trees.github.io repository). It loads its data from the Apps Script `?view=data` endpoint. If the address changes, update `MAP_PAGE_URL` in `Map-script` too (the tap page links to the map and loads Leaflet from it).

## The sheet

- **Trees**: one row per tree. Columns are found by their header in row 1, so they can be moved and new ones added (like `Road`); the script needs `Tree ID`, `Species`, `Location`, `Lat`, `Long`, `Planted` and `Responsibility` (case and spacing don't matter; `Latitude` and `Longitude` also work). If one is missing or renamed, the map says which, and tag taps are still logged. New rows appear on the map the next time it loads. `Tree ID` is what the NFC tag links to, so never renumber or reuse one. `Planted` can be a planting season (`Winter 25/26`, counted as planted on 1 March 2026), a date or a year. `Responsibility` is `Amey` or `Community - until <month> <year>`.
- **Log**: one row per tag tap, written by the script, so keep its columns in this order (Timestamp, Tree ID, Litres, Undone). `Litres` is filled in only if the volunteer picks an amount on the tap page; blank counts as the default amount. `Undone` is filled in if the volunteer chose "Tapped by mistake? Remove this watering" (allowed for an hour); those rows are kept but don't count.
- **Summary**: columns A–F list every tree with its status worked out exactly as the map does. The script writes them: use the sheet's **Tree map > Refresh summary** menu, or **Tree map > Refresh summary every hour** once to keep it up to date automatically (it asks for permission the first time).
- **Summary** settings (column H, blank means the default):
  - H1: no longer used (it was the base interval before H7–H10)
  - H2: rain in 48 h that counts as a watering, in mm (25)
  - H3 / H4: first and last day of the watering season, like `1 Apr` / `15 Sep` (the default); a month name alone means its first / last day
  - H5: drought mode, `on` or `off`
  - H6: litres assumed when no amount is picked (5)
  - H7 / H8 / H9: days between waterings in the 1st / 2nd / 3rd summer after planting (3 / 7 / 14)
  - H10: days between waterings for established trees in drought mode (28)

## Watering priority

How often a tree needs the community to water it depends on its age, who looks after it and whether drought mode is on (with the default settings in Summary H7–H10):

| Tree age | Community | Community, drought | Amey | Amey, drought |
|---|---|---|---|---|
| 1st summer after planting | 3 days | 2 days | Amey only | 3 days |
| 2nd summer | 7 days | 4 days | Amey only | 7 days |
| 3rd summer | 14 days | 7 days | Amey only | 14 days |
| Established | none | 28 days | none | none |

Drought mode halves the young trees' intervals (rounded). Amey waters its trees weekly, so in a drought they need a community top-up on the normal schedule. Drought mode also switches watering on outside the season (1 April to 15 September by default). During the season, trees with no watering yet count from the day the season started. Heavy rain (Summary H2) counts as a watering.

A tree shows as "due soon" (amber) for the last 2 days before it is due, or the last day for intervals of 3–5 days. Trees on a 2-day interval go straight from watered to due.

The map's "trees need water" list is sorted by urgency: days since watering ÷ the tree's interval. A tree in its first year therefore comes ahead of an older tree that has been dry for the same time. It can also be sorted by distance ("Nearest"), using the phone's location.

## Publishing changes

Every change is checked for JavaScript syntax errors (`.github/check-syntax.js`, run by `.github/workflows/pages.yml`) and the map is only published if the check passes. This needs **Settings > Pages > Build and deployment > Source** set to **GitHub Actions**. A failed check shows as a red cross next to the commit, and the previous version of the map stays live.

After changing `Map-script`, paste it into the Apps Script editor and publish a new version of the existing deployment (Deploy > Manage deployments > Edit > New version), so the web app URL on the tags stays the same.

## The map

### Demo controls (season and drought preview)

For now the map always shows a small demo panel with two switches: Season (In / Out) and Drought (Off / On). Each switch starts on the real state; picking the other option previews it, and picking the real one again returns to it. Nothing in the sheet changes. Links can open a view directly: `?preview=in`, `?preview=out`, `?drought=on`, `?drought=off`.

To hide the panel again, set `DEMO_CONTROLS = false` in `index.html`. After that, only links containing `?preview` or `?drought` show it.

The map always opens showing every tree. The button under the location button ("Show all trees") returns to that view, and the map does so itself when a phone restores the page from memory or after 30+ minutes away (when it also refreshes the data). Links with `?tree=<id>`, like the one on the tap page, open on that tree.

On the map every tree is its own dot, all the same size; the dots are smaller when zoomed out and grow as you zoom in. Where dots overlap, the most urgent is drawn on top. Newly planted (1st year) trees that are 2 or more days overdue get a "!" badge.

To go back to grouping nearby trees into numbered circles until you zoom in, set `CLUSTER_TREES = true` in `index.html`. To compare without changing anything, add `?cluster=on` to the map address (or `?cluster=off`).

## Rolling back

The November 2026 changes (separate dots, same-size dots, columns found by header, 3/7/14-day intervals, the 1 Apr–15 Sep season) can be undone in parts:

- **Map page** (`index.html`): on GitHub, open the merged pull request and press **Revert**, then merge the pull request that creates. The previous map is published again in a few minutes. For clustering alone, set `CLUSTER_TREES = true` instead.
- **Apps Script** (`Map-script`): Apps Script keeps every published version. In the editor, Deploy > Manage deployments > Edit, pick the previous version number under **Version** and Deploy. The tag links keep working.
- **Sheet**: the old script ignores Summary H7–H10 and uses H1 (7) again, so the sheet can stay as it is. Before November 2026 the season was May to September.
