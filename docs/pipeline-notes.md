# Pipeline notes

## Why the reply-history cut matters

Email replies quote everything above them. The original RFQ states the item and
the quantity requested, so a model reading the full reply body sees our own
numbers and returns them as if the supplier had quoted them. The extracted table
then looks plausible and is wrong in exactly the field a buyer trusts most.

The fix is a search for the first reply marker — `От:`, `From:`, `Отправлено:`,
`Sent:` — and truncation there, with a minimum offset so a reply that opens with
the marker is not reduced to nothing.

This is worth stating plainly because it generalises: most failures in LLM
pipelines over real business data are input hygiene failures, not model failures.
The model was never wrong; it was answering about text that should not have been
in front of it.

## Why the output is a fixed JSON schema

Free-form extraction produces something a human still has to read. A fixed field
set — supplier, unit price, currency, quantity, lead time, payment terms, comment
— produces something a spreadsheet can consume.

Two safeguards around it:

- The model is told to return `null` for missing fields rather than guess. A
  guessed lead time is worse than an empty cell, because an empty cell prompts a
  follow-up call and a guess does not.
- Code fences are stripped before parsing, and a parse failure produces a marked
  row rather than an exception. The pipeline degrades, it does not stop.

## Cost shape

Extraction runs once per supplier reply on a short text. With a small fast model,
per-RFQ cost is negligible against the clerical time it replaces. Cost scales
with the number of replies, not with the value of the purchase — which means the
economics are best on high-frequency low-value sourcing, exactly the segment
where buyer time is hardest to justify.

## What production would require

1. Conversation-ID matching instead of subject-line matching.
2. A processed-message store, so re-runs are idempotent.
3. A scheduled trigger, an RFQ closing time, and reminders to non-responders.
4. Currency and unit-basis normalisation before any sorting by price.
5. Structured output validation against the schema, rejecting malformed objects
   rather than accepting whatever parses.
6. An audit trail linking each extracted row to the source message.
