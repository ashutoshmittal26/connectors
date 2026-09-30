# Shortcut audit - the-documents-i-never-signed (email-calendar-gym)

Declared trajectory: 4 steps, 3 hops.
Shortest path 3 calls against 3 declared hops - ratio 1.00.

## What the declared trajectory is

| step | call | why it cannot be skipped |
| --- | --- | --- |
| s1 | `search_email {"queries":["signature"],"pageSize":50}` | measured: `total_matches` 111, `count` 50, `truncated: true`, `nextPageToken` present. It returns the SHAPE of the stream - the request, reminder and person-keyed subject lines. This gym has five tools and **no per-item read**, so a message can only be reached through `search_email`, and nothing outside these results tells an agent how the service words a reminder or a completion. |
| s2 | `search_email` with the four family phrases, `pageSize:50` | measured: buckets 36 / 46 / 12 / 1, `total_unique_emails` 95, `truncated: false`. Every phrase is lifted verbatim out of s1's subjects. The completion family is invisible to s1's own query - `signature` returns 111 and not one of the 12 `You just signed` messages, because those subjects never contain the word. |
| s3 | `search_email {"queries":["\"Signature requested by\" label:Opened"],"pageSize":50}` | measured: 18 of 36, `truncated: false`. A search result payload carries exactly ten keys - `email_id`, `id`, `thread_id`, `snippet`, `internal_date_ms`, `from`, `to`, `cc`, `subject`, `date` - and **no `labelIds`**. Register membership cannot be read off s1 or s2 at any page size; it has to be re-asked with the label operator and intersected by id. |
| s4 | `draft_email` | the graded write-back. |

`chain_depth` = longest dependency path = s1 -> s2 -> s4 = **3**, which is this gym's floor
(`CHAIN_FLOOR_BY_GYM["email-calendar-gym"] = 3`). Nothing was added to reach it.

## The shortest honest path, measured

**3 calls.** `pageSize` is PER QUERY on `search_email`, and every one of the five buckets this task
needs is under the cap of 50 (36 requests, 46 reminders, 12 completions, 1 everyone-has-signed, 18
register). So an agent that has read s1's page one can merge s2 and s3 into a single five-query
call:

```
search_email {"queries":["\"Signature requested by\"","\"is awaiting your signature\"",
                         "\"You just signed\"","\"Everyone has signed\"",
                         "\"Signature requested by\" label:Opened"], "pageSize":50}
```

That is s1 -> merged call -> `draft_email` = **3 calls, chain depth 3**. Ratio 3/3 = **1.00**.

## Bulk routes ruled out, and why two calls is not enough

* There is **no bulk route** on this gym: five tools, no `get_message`, no `get_thread`, no label
  tool, no draft list. A listing returns fully hydrated matches, but only for the queries asked.
* **One call cannot do it.** `{"queries":["signature","signed","label:Opened"],"pageSize":50}`
  returns 50 of 111, 20 of 20 and 50 of 326: the signature bucket is truncated, so the request and
  reminder counts per document are incomplete, and the register set is a 50-row prefix of a
  326-message label. Measured.
* **Guessing the family phrases is not a path.** `signature` (111) misses every completion;
  `signed` (20) misses every request and reminder; `subject:sign` (178) sweeps in 54 standing
  consent-form notices that name no document. No single query is the log, and the phrasings that
  partition it are only visible once a sweep has been read.
* **The calendar cannot shorten anything.** This is a mail-only question; `search_calendar` is
  never called, and it could not add depth anyway (`pageSize` maxes at 2500 against 2330 served
  events, so it never truncates).

## What a shortcut would look like, and why it is not one

Reading `total_matches` for a quoted document name instead of classifying subjects is one call
shorter per document but **wrong**: `"results (4).pdf"` returns 33 across three different
documents, `"Carmen Squires - results (4).pdf"` returns 16 against a true 15, `"hi.jpeg"` 9 against
8. The extra hits are snippet matches. Every graded count here comes from a classified subject
set, and the verifier `documents_the_ones_signed_with_no_chasing` fails precisely that shortcut -
it requires reminder counts of 0 where the quoted searches answer 3, 2 and 2.
