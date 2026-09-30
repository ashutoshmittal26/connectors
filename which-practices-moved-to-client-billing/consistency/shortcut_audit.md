# Shortcut audit - which-practices-moved-to-client-billing

Declared trajectory: 26 steps, 25 hops (g1 Drive listing by the fourteen practice names, b01-b23 one values batchGet
per statement whose title begins with a listed name, g2 Drive listing by the 13 practice IDs those statements carry as
tab names, r1 one values batchGet of the raw export g2 returns; g1 is the only step whose arguments come from the
request alone). Chain depth 4: g1 -> b02 -> g2 -> r1.

Shortest path 25 calls against 25 declared hops - ratio 1.00.

## Why no bulk route collapses it (measured on the served gym as the Drive's is_me user, 2026-09-28)

Every graded fact - which rows carry an order id, what each Bill Type cell reads, which statements hold nothing - lives
only in cells, and the gym serves the cells of one spreadsheet per call.

- `drive files list` is the one bulk listing: a single call (pageSize up to 1000; the Drive holds 233 live
  spreadsheets, one page) returns every matching file, but no listing carries a cell value, and every spreadsheet
  reports the same size ("50"), so a listing cannot tell an empty statement from a full one. It answers "which
  spreadsheets" and nothing else.
- `sheets spreadsheets values batchGet` takes exactly one `spreadsheetId` (several `ranges` are allowed, all inside that
  spreadsheet). `sheets spreadsheets get` with `includeGridData`, `sheets +read` and `drive files export` are one
  spreadsheet per call too. Every spreadsheet here has one tab, so one read per spreadsheet is enough, and an
  unqualified range echoes the tab title even when the tab is empty.
- The raw export is titled with nothing but a practice ID. That ID is not in the request: it is the tab title of the
  four Empowered Goddess statements, all of them empty, so the export can only be recognised after they are read.
- The 8 empty statements must be read as well: the two silent practices (Des Moines, Elke Barnes) are silent only
  because every one of their statements is empty, and the listing cannot show that.

So the shortest correct path is one listing plus one read per spreadsheet: 1 + 24 = 25 calls. Listing every
spreadsheet at once (233 titles, the ID-titled export among them) saves the second listing g2, because the export's id
is already in hand once the statements have shown which ID belongs to which practice. Replayed as route M1 and scored
1.0. The declared path spends that call on g2 instead, listing the exports by the IDs it has just read; the hop count
(25) is the same either way. Skipping the raw export is route R2 (24 calls, 0.3333); the first listing alone is route
R5 (1 call, 0.0).

## Routes (tools/run_routes.sh, each on a fresh database, scored by the task's own tests/test.sh; 11 of 11 as designed)

| route | calls | reward | fails | what it is |
|---|---:|---:|---|---|
| R0-gold | 26 | 1.0 | none | the golden trajectory and the gold workbook |
| M1-cosmetic-shortest-path | 25 | 1.0 | none | measured shortest correct path (one listing of every spreadsheet, then the 24 reads); Summary first, names in title case and bold, figures as floats in a 0.00 format, file ids upper case, order ids without dashes, booleans as mixed-case text, a column width and frozen panes |
| R1-default-misreading | 26 | 0.6 | summary_billed_practices, summary_cash_pay_practices, summary_client_bill_practices | the lever's DEFAULT MISREADING: the Summary counts billing rows (37/20/17) instead of practices (12/7/7); the Billing sheet is correct |
| R2-raw-export-skipped | 24 | 0.3333 | summary_billed_practices, summary_client_bill_practices, summary_silent_practices, billing_sheet_reconciles_with_the_summary, billing_and_summary_hold_nothing_else | the raw export titled with Empowered Goddess's ID never listed or read: 36 rows, 11/7/6/2/3; the tool floor counts 25 calls (24 gws + the harness finish call) against 24, so the workbook checks are what catch it |
| R3-trash-included | 28 | 0.7333 | billing_sheet_reconciles_with_the_summary, billing_and_summary_hold_nothing_else | Johnson & Associates' trashed raw export listed with trashed = true, read and kept: 38 rows; every Summary figure unchanged |
| R4-empty-statements-counted | 26 | 0.8667 | summary_silent_practices | silent_practices read as the empty statements (8) instead of the practices with no billing row (2) |
| R5-laziest-listing-only | 1 | 0.0 | every check | laziest: the first listing only, one row per titled statement with no order ids read, a summary guessed from the fourteen names |
| R6-greediest-header-row | 26 | 0.7333 | billing_sheet_reconciles_with_the_summary, billing_and_summary_hold_nothing_else | correct figures and log, but a header row on the Billing sheet pushes every row down one |
| R7-exclusive-bill-type-reading | 26 | 0.7333 | summary_cash_pay_practices, summary_client_bill_practices | cash_pay_practices and client_bill_practices read as the practices billed only that way (5 and 5) instead of those with one or more such rows (7 and 7) |
| R8-title-wording-as-practice | 26 | 0.6 | summary_billed_practices, summary_client_bill_practices, summary_silent_practices | each title's full wording taken as the practice instead of the listed name: Johnson & Associates, Empowered Goddess and Elke Barnes split in two (13/7/8/2/4) |
| R9-johnson-may-missed | 25 | 0.7333 | billing_sheet_reconciles_with_the_summary, billing_and_summary_hold_nothing_else | Johnson & Associates searched by the fuller Johnson & Associates Midwifery Group, which misses its May 2025 statement: 29 rows; every Summary figure unchanged because the August statement is client bill too |

Write surfaces: the workbook only. gws is read-only for grading on this image (its state route answers `rows`, not
`result`), so the bundle carries no database_state or gym_state_check verifier; the task asks for no Drive write, and
the routes write nothing to the gym. The tool_execution floor (24 counted calls, below the 25-call shortest path)
fails R5.

## The lever's default misreading

Counting billing rows instead of practices gives billed_practices 37, cash_pay_practices 20, client_bill_practices 17
(gold 12, 7, 7; switched_practices 2 and silent_practices 2 are practice counts under either reading). Built as route
R1: it fails exactly the three Summary checks the lever moves and no per-row check.
