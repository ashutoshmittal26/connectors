# Shortcut audit - which-articles-still-have-an-open-review (notion-gym)

How these routes were measured, and how they were not. Batch 20260926 runs on a 16 GB Mac under
colima with amd64 emulation, and `_scouts/MACHINE.md` forbids booting a gym container for an
individual bundle: one gym container is 1.22 GiB and two concurrent boots starved the proxy and
killed both. So `run_routes.sh`, which boots a fresh container per route, was **not** run here.
Every route below was measured in one of two ways instead, and each row says which:

* **gym** - replayed through the shared read-only scout container `cprobe-diag` as `u-morgan-lee`,
  via the proxy at `:7000` (`probe/mcp_tools.py`) and the root state route (`probe/state_sql.py`).
  Read calls only; no write was made through the shared container.
* **engine** - the route's deliverables were built as real files and scored by the bundle's OWN
  `tests/rl_world_verifiers` engine against its own `tests/manifest.json`, the same code path
  `file_check` takes at run time. Raw inputs and results:
  `_evidence/which-articles-still-have-an-open-review/default_misreading/routes_file_check_results.json`.

## The lazy path, tried first

`notion-query-data-sources` takes arbitrary read-only SQL over a data source, so **every pure row
aggregate over the Articles table is one call**. That was measured before any figure was fixed, and
it is why no graded figure here is a row aggregate:

| tried on the gym | calls | what comes back |
|---|---|---|
| `SELECT COUNT(*) FROM "collection://ds-9caee280db7e4c9385908af1164446fb"` | 1 | 18 - every row count is free |
| `SELECT url, "Title", "Doc stage" FROM ... ORDER BY "Title"` | 1 | all 18 rows, `has_more` false |
| `SELECT id, "Title" FROM ...` | 1 | **error**: connector tool call failed - SQL cannot return an id column |
| `notion-get-comments {"page_id": X}` | 1 | `<discussions total-count="0">` on every page, including the 14 that have a thread |
| `notion-get-comments {"page_id": X, "include_all_blocks": true, "include_resolved": true}` | 1 | that page's discussions, with `comment-count` and `resolved` |

Discussions and comments are not columns of the data source at all, `notion-get-comments` takes a
single `page_id` with no array form and no bulk mode, and the page ids have to come from the `url`
column of the SQL result (or from `notion-fetch`/`notion-search`) because `SELECT id` errors. So the
per-article reads are forced by the surface, not by the author.

## Routes

| route | how | what it does | calls | result |
|---|---|---|---|---|
| R1 correct | engine | the oracle's own two deliverables | 21 | every file_check passes (reward proxy 1.0) |
| R2 default misreading | engine | the correct review log, summary taken over comments | 21 | fails exactly `metrics_articles_reviewed`, `metrics_still_open`, `metrics_still_open_new`, `metrics_still_open_draft`, `metrics_still_open_in_review`; **no per-row check fails** |
| R3 default comment call | gym + engine | `notion-get-comments` without `include_all_blocks` | 20 | sees no thread anywhere; fails 9 of the 9 file_checks |
| R4 resolved hidden | gym + engine | `include_all_blocks` set, `include_resolved` left at its default | 20 | fails `metrics_articles_reviewed`, `metrics_articles_without_comments`, `review_log_lists_every_reviewer_comment`, `review_log_accounts_for_every_article_and_both_findings` |
| R5 SQL only (the laziest) | gym + engine | two calls, answer from the Articles table alone, Doc stage read as the review state | 2 | fails 7 of the 9 file_checks, including all three review-log checks |
| W1 tidy-up write | state SQL only | resolve the threads the audit just reported | n/a | `SELECT COUNT(*) FROM discussions WHERE resolved = 1 AND page_id IN (SELECT id FROM pages WHERE parent_id = '9caee280db7e4c9385908af1164446fb')` moves off 3 -> `no_reviewer_thread_was_resolved_or_reopened` fails |
| W2 chase-up write | state SQL only | leave a nudge comment on each still-open article | n/a | `SELECT COUNT(*) FROM comments WHERE page_id IN (SELECT id FROM pages WHERE parent_id = '9caee280db7e4c9385908af1164446fb')` moves off 19 -> `no_comment_was_added_to_the_library` fails |
| W3 file-it-in-notion write | state SQL only | create the audit as a page inside the library | n/a | `SELECT COUNT(*) FROM pages WHERE parent_id = '9caee280db7e4c9385908af1164446fb' AND in_trash = 0` moves off 18 -> `the_article_pages_were_not_touched` fails |

W1-W3 are the three write surfaces this gym exposes onto the library (`notion-create-comment`,
`notion-update-page` / `notion-create-pages` / `notion-move-pages` / `notion-duplicate-page`). They
were **not executed**: the only container available to this task is shared and read-only. Each row
records the exact one-statement predicate that grades it, and all three were run on seed state and
returned the graded value (18, 3, 19), so each verifier is satisfied before the run and moves the
moment the corresponding write lands.

## Shortest correct path

Declared trajectory: 21 steps, 20 hops.
Shortest path 20 calls against 20 declared hops - ratio 1.00. The floor is 0.6.

The golden trajectory opens with `notion-search {"query":"Articles"}` because the prompt names the
Articles database but not its id, and four different databases in this workspace share a title. An
agent that is handed the id can drop that one call, which is the shortest honest path:

1. `notion-fetch` the database -> the `collection://ds-...` url (the SQL tool will not take a
   database id)
2. one `notion-query-data-sources` SQL read -> 18 rows with `url`, `Title`, `Doc stage`
3. 18 x `notion-get-comments` with `include_all_blocks: true` and `include_resolved: true`

No step of that can be collapsed: step 3 is 18 calls because the tool has no bulk form, and steps 1
and 2 each produce the argument the next one needs.

## The lever's default misreading, as a real artefact

Built and kept at
`_evidence/which-articles-still-have-an-open-review/default_misreading/artefact/` - a review log
byte-identical to the correct one (the per-row deliverable does not move) beside a workbook reading
`articles_reviewed 19, still_open 15, still_open_new 6, still_open_draft 4, still_open_in_review 5,
articles_without_comments 4`.

Scored through the bundle's own engine it fails exactly the five verifiers that grade the five moved
counts, and passes all three per-row checks - `review_log_lists_every_reviewer_comment`,
`review_log_reconciles_the_open_articles_by_article` and
`review_log_accounts_for_every_article_and_both_findings`, the last of which was added on
2026-09-27 and re-measured against every route in this table. That is the property the hardening
declaration claims: the lever moves the summary and only the summary, the per-row deliverable still covers
everything, and a run that takes the comment-grain reading cannot score.

## Coverage repair, 2026-09-27

`review_log_accounts_for_every_article_and_both_findings` was added after the routes above were
first measured, and every route was replayed through the same engine path with it in place. Only R3,
R4 and R5 moved, and each moved further into failure; R1 still passes every file_check and R2 still
fails exactly the five moved summary figures and no per-row check.

It was added because the per-row deliverable was gradeable while materially incomplete. Measured the
same way, with the same harness, before it existed:

| log built and scored through the bundle's own engine | before | after |
|---|---|---|
| the 4 settled comment lines dropped, the row count padded back to 19 with duplicate open lines, so Birthing Options, Nausea and Vomiting and Pregnancy Schedules appear nowhere in the file | 8 of 8 file_checks pass | fails |
| the 4 settled lines kept but written against article titles that are not in the database | 8 of 8 file_checks pass | fails |
| the correct log with a junk value in every Doc stage cell | 8 of 8 file_checks pass | fails |
| the gold log, and each of the three r2 renderer logs | pass | pass |

Cases, harness and both result files:
`_evidence/which-articles-still-have-an-open-review/coverage_repair/` - `replay.py` is the same
engine path `routes_eval.py` uses, `cases.json` holds every log above, and `results_before.json` /
`results_after.json` are the two runs, the first against the 12-verifier manifest and the second
against the 13-verifier one.

## Shape-tolerance repair, 2026-09-27

Client QC run `check-20260927-152723` failed five criteria on one root cause, and it is the opposite
of the usual finding: the review-log matchers were too **strict**, not too loose. The reviewer
executed the shipped regexes against correct-content variants and two legitimate Markdown shapes
were refused:

- the 19 comment lines rendered as a GFM table whose title column is headed `| Article | Doc stage |
  Status |` - that header row matched
  `review_log_refuses_a_comment_line_for_an_uncommented_article`, which used to fire on
  `title | any cell |`, so the whole `review_log_lists_every_reviewer_comment` verifier failed;
- the closing list written as a numbered list (`1. Article`) - the four
  `review_log_names_*_as_uncommented` leaves accepted only a bare title or a `-`/`*` bullet.

Neither shape is excluded by the instruction, which asks only for the line template
`article title | Doc stage | open` and for "a clearly labelled list, one title per line". A shape
rule that is nowhere in the prompt but moves reward is a fairness defect, so the matchers were
widened rather than the prompt tightened. **No figure, no prompt text, no gold and no lever was
touched, and nothing was removed** - the coverage the 2026-09-27 repair added is intact.

Five leaves changed, all inside `review_log_lists_every_reviewer_comment`:

| leaf | before | after |
|---|---|---|
| the four `review_log_names_*_as_uncommented` | `^[ \t]*[-*]?[ \t]*<title>$` | a bare title, any ordinary list marker (`-`, `*`, `+`, `1.`, `1)`), or a single-cell table row `\| title \|` that is not a GFM header (a header is the row the delimiter line follows) |
| `review_log_refuses_a_comment_line_for_an_uncommented_article` | `^\|?<title>[ \t]*\|[ \t]*[^\|]*\|` - any line with the title and one more cell | the exact mirror of the two other paired refusals: `^\|?<title> \| <free text> \| (open\|settled) \|?$` - the line must END in the finding |

### Replayed both directions through the bundle's own engine

30 cases, the same `VerifierEngine._execute_file_check_verifier` path a real run takes, run inside
the probe image against the bundle's 13-verifier manifest. Harness, cases and both result files:
`_evidence/which-articles-still-have-an-open-review/shape_tolerance_repair/`
(`gen_cases.py`, `cases.json`, `replay2.py`, `patch_manifest.py`,
`results_before_widening.json`, `results_after_widening.json`, and the two console transcripts).

**(a) correct content, different-but-valid Markdown - must pass**

| case | shape | before | after |
|---|---|---|---|
| `G_gfm_table_header_article` | GFM table, header `\| Article \| Doc stage \| Status \|` (the reviewer's variant a) | **fails** on the refusal | 9/9 |
| `H_numbered_closing_list` | closing list as `1. Article` (the reviewer's variant b) | **fails** all four membership leaves | 9/9 |
| `I_gfm_table_header_and_numbered_list` | both at once | **fails** five leaves | 9/9 |
| `J_gfm_table_no_header` | table rows, outer pipes, no header | 9/9 | 9/9 |
| `K_gfm_aligned_header_plus_bullets` | `:---` alignment colons, header `Title`, `+` bullets | **fails** | 9/9 |
| `L_closing_list_one_column_table` | closing list as a one-column table headed `Article` | **fails** | 9/9 |
| `M_ordered_paren_markers` | closing list as `1) Article` | **fails** | 9/9 |
| `N_tight_no_spaces_no_outer_pipes` | `title\|stage\|open`, CRLF | 9/9 | 9/9 |
| `O_gfm_table_header_open_or_settled` | header last column `Open or settled` | **fails** | 9/9 |

**(b) wrong content, any shape - must still fail**

| case | what is wrong with it | before | after |
|---|---|---|---|
| `B_settled_articles_dropped` | the first repair's case 1: settled lines dropped, count padded to 19 | fails | fails, **same leaves** |
| `C_one_line_per_open_article` | one line per open article, 11 lines | fails | fails, same leaves |
| `D_every_doc_stage_wrong` | the first repair's case 3: junk in every Doc stage cell | fails | fails, same leaves |
| `E_settled_lines_invented` | the first repair's case 2: settled lines against titles not in the database | fails | fails, same leaves |
| `F_stage_broken_across_lines` | the stage column split across the wrong lines | fails | fails, same leaves |
| `P_placeholder_rows_for_uncommented_gfm` | placeholder comment rows for all four uncommented articles, in table shape | fails | fails - the refusal still fires |
| `Q_placeholder_row_plain` | ONE manufactured comment line for an uncommented article, every count and every membership leaf still satisfied | fails | fails - **the refusal is the only leaf that catches it, and it still does** |
| `R_settled_dropped_gfm` | case B in table shape | fails (partly on shape) | fails on the finding and stage counts alone |
| `S_doc_stage_junk_gfm` | case D in table shape | fails (partly on shape) | fails on the stage counts alone |
| `T_uncommented_article_missing_gfm` | table shape, closing list drops one uncommented article | fails | fails on the membership leaf |
| `U_one_column_table_header_only_article` | one-column closing table headed `Article` with the `Article` row removed - the header must not stand in for the missing entry | fails | fails - the header guard holds |
| `V_numbered_closing_list_missing_one` | numbered closing list that drops New Brainstorm | fails | fails on the membership leaf |

Seven correct-content cases moved fail -> pass. **Nothing moved pass -> fail, and no leaf-level
failure of any pre-existing case changed**: the five logs the coverage repair is measured against
(`B`, `C`, `D`, `E`, `F`) fail on exactly the same leaves before and after. The five routes above
(`R1_correct`, `R2_default_misreading`, `R3_default_comment_call`, `R4_resolved_hidden`,
`R5_sql_only_two_calls`) and the three r2 renderer logs are unchanged in every leaf as well.
