# RFQ Automation Agent

**Automated request-for-quotation dispatch and quotation collection for industrial
procurement — n8n workflows with LLM extraction of unstructured supplier replies
into a comparison table.**

Status: working proof of concept, tested end to end on a controlled test RFQ.
Not yet in production use.

Credentials, mailbox addresses and the corporate domain have been replaced with
placeholders. No supplier data, pricing or company records are included.

---

## The problem

Requesting quotations is one of the most repetitive tasks in a procurement
function, and one of the least suited to a human.

For a single sourcing event, a buyer writes near-identical emails to a supplier
list, waits, then reads replies that arrive in no common format — some as prose,
some as an attached price list, some as three words and a number. The buyer then
retypes all of it into a comparison sheet.

The cost is not intellectual, it is clerical: hours per event, multiplied by the
number of events. And the retyping step is where transcription errors enter the
comparison that decisions are made from.

The part worth automating is not the decision. It is everything around it:
dispatch, collection, extraction, and assembly of the comparison table.

## What it does

Two workflows.

**`01-rfq-dispatch`** builds the request from a small parameter block (RFQ code,
item, quantity), fans it out across the supplier list, and sends one email per
supplier through Microsoft 365 / Outlook.

**`02-rfq-collection`** pulls the mailbox, keeps only replies belonging to this
RFQ, strips quoted history from each reply, sends the remaining text to Claude
for extraction into a fixed JSON shape, sorts the results by unit price, and
mails back an HTML comparison table.

```
01  trigger → build parameters → fan out to supplier list → send RFQ emails

02  trigger → fetch mailbox (50 most recent)
            → filter: subject matches RFQ code, sender is not us
            → strip quoted reply history
            → LLM extraction → JSON per quotation
            → parse, sort by unit price, build HTML table
            → email comparison table to the buyer
```

## Design decisions worth explaining

**Extraction into a fixed schema, not summarisation.** The model is asked for a
single JSON object with a defined field set — supplier, unit price, currency,
quantity, lead time in days, payment terms, comment — and told to return `null`
for anything absent. A summary would still need reading; a schema goes straight
into a table. The prompt is in `prompts/`.

**Claude Haiku rather than a larger model.** Extraction from a short email is not
a reasoning task. The cheap, fast model does it, and the cost per RFQ stays low
enough that the economics work at volume. Model choice follows task difficulty,
not availability.

**Quoted history is stripped before the model sees it.** Email replies carry the
entire prior thread. Without the cut, the model reads our own original request
and extracts the quantity we asked for as if the supplier had quoted it. A regex
finds the reply marker and truncates there. This is the single change that moved
extraction from unreliable to reliable, and it is not an LLM problem — it is an
input hygiene problem.

**Unparseable replies land in the table, not in a log.** If the model returns
something that is not valid JSON, the row still appears, marked as unrecognised
with the first fragment of the raw text. A supplier who replied must never
silently disappear from the comparison — an incomplete table that looks complete
is worse than an obviously incomplete one.

**The buyer receives a table, not a decision.** Output is sorted by unit price
and sent for a human to act on. Award decisions involve technical fit, supplier
history and payment risk that this pipeline does not see.

## What is in this repository

```
/workflows/01-rfq-dispatch.json      RFQ dispatch workflow (n8n export)
/workflows/02-rfq-collection.json    collection, extraction and comparison workflow
/prompts/extract-quotation.txt       the extraction prompt and its output schema
/docs/                               notes on the pipeline
```

Import the JSON files into n8n via **Import from File**, then attach your own
Outlook and Anthropic credentials — the placeholders will not connect to
anything.

## Known limitations

This is a proof of concept, and the gap between it and production is mostly in
these points.

- **Replies are matched by RFQ code in the subject line.** It works while
  suppliers use Reply. A forwarded reply, a rewritten subject, or a supplier
  answering from a different address breaks the match. Matching on the Outlook
  conversation ID would be robust; the subject string was chosen for speed of
  assembly, not correctness.
- **No state between runs.** The collection workflow reads the 50 most recent
  messages every time and has no memory of what it already processed. Running it
  twice produces the table twice. Production use needs a processed-message store
  and a date bound.
- **Both workflows are manually triggered.** No schedule, no closing time for the
  RFQ, no reminder to non-responders.
- **No currency or unit normalisation.** The schema records a currency field but
  nothing converts it, and nothing checks that a quoted price is per unit rather
  than for the lot. Sorting mixed-currency or mixed-basis quotations by price
  produces a ranking that looks authoritative and is wrong. This is the most
  dangerous limitation of the set, because unlike the others it fails silently.
- **The RFQ code is hard-coded in several places** rather than passed as a
  parameter from a single source.
- **The HTML table is built by string concatenation** from model output, without
  escaping. Fine for an internal test, not for content that originates outside
  the organisation.
- **Tested against a controlled test RFQ**, not against a live supplier panel.

## Author

Mirzhan Dukenov — procurement and supply chain, process design and automation.
[LinkedIn](https://www.linkedin.com/in/mirzhan-dukenov/)

---

*Built during employment. Published with credentials, addresses and company
identifiers removed; no supplier data, pricing or company records are included.*
