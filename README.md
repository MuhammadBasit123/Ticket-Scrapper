# Ticket-Scrapper
# Ticket Extraction Pipeline

Turns messy support emails/tickets into validated, typed data.

## How it meets the assessment criteria

**A typed schema the output must satisfy** — [`src/schema.ts`](src/schema.ts)
defines `SupportTicketSchema` with `zod`: enums for category/priority/
sentiment, a validated email format, a length-capped summary, and nullable
fields for anything that's genuinely optional in a real email.

**Validation with an explicit failure path** — [`src/extractor.ts`](src/extractor.ts)
never throws for bad input. `extractTicket()` always returns a tagged union
(`ExtractionResult`): `success`, `invalid_schema` (with the zod issues
attached), or `extraction_failed` (transport/API error). Callers branch on
`status` instead of wrapping calls in try/catch.

**Measured extraction accuracy on 50+ samples** — [`data/samples.json`](data/samples.json)
has 65 labeled examples (regenerate with `node scripts/generate-samples.mjs`).
[`src/evaluate.ts`](src/evaluate.ts) runs every sample through the pipeline
and reports per-field accuracy (category, priority, sentiment, action
required, and whether name/email were correctly detected as present vs
absent), plus counts of schema-invalid and failed extractions.

**Handles documents with none of the target fields** — the schema has a
`hasSupportContent` flag specifically for this. 10 of the 65 samples are
non-tickets (newsletters, spam, out-of-office replies, meeting notes). The
extraction prompt instructs the model to set `hasSupportContent: false` and
leave the rest null/default rather than inventing values, and this is scored
like any other field in the evaluation.

## Setup

```bash
npm install
export ANTHROPIC_API_KEY=sk-ant-...
npm run evaluate        # runs the real pipeline against Claude, prints accuracy report
```

To sanity-check the pipeline without an API key:

```bash
npm run evaluate:mock   # crude regex-based extractor, low accuracy by design — just proves the wiring works
```

## Project layout

```
src/
  schema.ts       # zod schema + ExtractionResult tagged union
  llmClient.ts     # Claude tool-use call that returns structured JSON
  mockClient.ts    # offline regex-based stand-in for evaluate:mock
  extractor.ts     # orchestrates call + validation, no throws for bad input
  evaluate.ts      # runs all samples, prints per-field accuracy
data/
  samples.json     # 65 labeled examples (55 tickets + 10 non-tickets)
scripts/
  generate-samples.mjs  # regenerates data/samples.json
```

## Design decisions

- **Tool-use over free-text JSON parsing.** The extraction call uses Claude's
  tool-use feature with a JSON-schema tool definition, so the response is
  structured input rather than prose we'd have to regex out of a response.
- **Nullable, not omitted.** Optional fields are `null` when absent rather
  than missing from the object, so downstream consumers don't need
  `?.`-chains everywhere and "we don't know" is distinguishable from "we
  forgot to ask."
- **Non-ticket detection is a first-class field, not a separate code path.**
  `hasSupportContent` lives in the same schema and is scored the same way as
  every other field, which keeps the "no target fields" case from being an
  afterthought bolted onto the happy path.
