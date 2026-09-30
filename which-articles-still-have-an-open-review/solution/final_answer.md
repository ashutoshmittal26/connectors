Review sweep of the patient-education library. The two files are at /workspace/review_log.md
and /workspace/metrics.xlsx, and I changed nothing in the library itself.

The Articles database holds 18 pages. 14 of them carry reviewer notes - 19 comments in all, and the
review log has one line for each of those 19, with the article, its Doc stage and whether the thread
that comment sits on has been settled. The other 4 (Article, Exercise with bump, New Brainstorm and
New Product Spec (PRD)) carry no reviewer comment at all and are listed at the end of the log, so
every article in the database appears in the file somewhere. Those same 4 are the only ones with no
Doc stage, so nothing that was reviewed is missing a stage.

Two things are easy to get wrong here and both cost the whole answer. notion-get-comments returns
nothing at all unless include_all_blocks is set - the threads hang off child blocks, so the default
call reports zero discussions on pages that plainly have them. And include_resolved is off by
default, which hides the settled threads and makes a settled article look like an article nobody
has touched.

Of the 14 that were reviewed, 3 have had their thread resolved: Birthing Options, Nausea and
Vomiting and Pregnancy Schedules. That leaves 11 articles still waiting on a person. Five of those
11 carry two comments rather than one, so a tally taken over comments instead of over articles comes
out at 15 and every stage figure comes out one or two high with it.

By stage, the 11 still open are 5 at New (Antenatal Testing, Aspirin in Pregnancy, Fertility after
Pregnancy, Group B Streptococcus, Thinking about a massage?), 2 at Draft (Foods to avoid, Ultrasound
in Pregnancy) and 4 In Review (Induction of Labor, Intercourse During Pregnancy, Title example,
Vaginitis & BV).

The figures:
  articles_reviewed          14
  still_open                 11
  still_open_new             5
  still_open_draft           2
  still_open_in_review       4
  articles_without_comments  4
