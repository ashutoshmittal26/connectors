# System Prompt
You are a careful task-execution agent. Use the tools provided by the configured gym to complete the user's task. Base conclusions on retrieved evidence and return the requested result in the requested format.

# Task
Our paperwork goes out for signature through a document service that mails me at every turn, and I
cannot say what I owe.

The log is every message that service sent me about a named document: asking me to sign it,
chasing me, or saying it has been signed. Its other notices are keyed to a person - who a request
went out to, who completed one - and out of scope.

The unit of audit is the document, named exactly as its subject line writes it: names differing by a
bracket are different documents, and a name that is not a file name counts too.
A document is signed when the log says so; a notice that everyone has signed it counts.

The audit's document register (`at least one of the messages requesting that document is filed
under my Opened label`) is the population the summary counts are taken over: the log also carries
documents that were never entered in the register, and every document in the log is assessed and
reported in the audit sheet.

Both go on ordinary worksheets in /workspace/audit.xlsx. Put the audit on a sheet named Documents,
starting with a header row document, requests, reminders, signed, tracked, then one row per
document in the log and no other rows: its name, how many log messages asked me to sign it, how
many chased me, then signed and tracked, yes or no.

Put the summary on a sheet named Summary, header row metric, value, then one row per metric in that
order and no other rows, spelled so: documents_tracked, the register's size; tracked_never_signed, those of it
not signed; tracked_signed_after_a_reminder, those signed and chased at least once;
tracked_signed_with_no_reminder, those signed and never chased.

Then draft one email, subject exactly Documents still awaiting my signature, body each unsigned
register document's name, one to a line and nothing else.
