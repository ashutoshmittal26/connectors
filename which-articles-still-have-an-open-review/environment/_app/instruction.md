# System Prompt
You are a careful task-execution agent. Use the tools provided by the configured gym to complete the user's task. Base conclusions on retrieved evidence and return the requested result in the requested format.

# Task
I look after the patient-education library and I have lost track of where review actually stands.
Every article we publish lives as a page in the Articles database, and reviewers leave their notes
as comments on the page itself. A thread somebody has settled is resolved; a thread nobody has
settled is still waiting on a person.

The unit of audit is the article: one article (`page id`) may carry several comments. Findings are assessed comment by comment; the summary counts of flagged and of each finding are counts of articles, by the finding reported for each comment in the review log.

Go through that database and work out where each article stands. Please do not change anything in
the library itself while you are in there; I want a read of where we stand, not an edit.

/workspace/review_log.md - the review log. One line for every reviewer comment in that database,
written as `article title | Doc stage | open` while the thread that comment belongs to is still
waiting on somebody, and `article title | Doc stage | settled` once that thread has been resolved.
Close the file with a clearly labelled list, one title per line, of the articles carrying no
reviewer comment at all, so that every article in the database appears somewhere in this file.

/workspace/metrics.xlsx - one row per metric, the metric name in the first column and the number in
the second column of that same row. The metric names are articles_reviewed, still_open,
still_open_new, still_open_draft, still_open_in_review and articles_without_comments, written
exactly like that. Each figure is the number itself, not text and not a sentence wrapped around it.
Rows may sit in any order, a header row is fine either way, and the sheet may be named anything
you like, so long as it is an ordinary worksheet.
