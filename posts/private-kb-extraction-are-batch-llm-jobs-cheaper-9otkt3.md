# Private-KB Extraction: Are Batch LLM Jobs Cheaper Than Realtime Node.js API Calls?

Short answer: use async LLM work for private knowledge-base updates that can wait, and keep realtime API calls for a person who is waiting for an answer. The cost decision is secondary to structured-output correctness: an inexpensive run that drops a document ID, invents a field, or produces JSON your application cannot parse is not a saving.

This is an experiment note. The evaluation constraint is the same source corpus, the same prompt, the same schema, and the same acceptance test in both paths. I would measure complete usable records, not just submitted requests or a top-level job marked complete. The simple approach is a loop of realtime calls. The chosen approach is a deadline-aware worker that can submit deferred work, reconcile every result, and reserve realtime capacity for interactive questions.

The important measurement comes before copying the choice: collect a representative slice of the knowledge base, estimate input and output tokens, and record the time by which each result is useful. Without that baseline, “cheaper” is a guess wearing a dashboard.

## How can a Node.js team compare batch LLM jobs with realtime calls for cheaper extraction?

Consider a developer tool that answers questions over private design notes, API references, and incident records. A user-facing answer needs a short feedback loop. Rebuilding an index, extracting metadata from a newly imported folder, or producing summaries for search can happen later. Those are different service-level objectives, even when they use the same model family.

That distinction is the first filter. Async processing is suitable when the application can tolerate a queue and a delayed result. It is not suitable when a user is blocked on the next response, when a document must be available immediately after upload, or when a job's completion deadline is shorter than the provider's documented processing window. Stick with realtime calls for those paths. The extra waiting time is a product behavior, not an implementation detail.

The cost comparison also needs a wider boundary than the API invoice. A realtime worker carries concurrency limits, backoff, request state, partial retries, and result persistence. A deferred job may reduce the amount of request orchestration, but it still needs an input manifest, status tracking, output retrieval, validation, and recovery. Queue storage, database writes, observability, and human review belong in the estimate. For a small team, this accounting is easy to skip: a script can finish its HTTP calls while leaving a second, unpriced system behind it to inspect malformed JSON, find missing IDs, replay timeouts, and explain why the search index has fewer records than the import manifest. Put those actions on the same worksheet as model usage before calling one lane cheaper.

No magic rate.

For structured answers, the acceptance test should reject a result before it enters the knowledge base. Validate the JSON shape, required keys, enum values, source-document ID, and citation or span references used by the application. Then run a content rubric: did the summary retain the required constraints, did tagging use the allowed vocabulary, and did extraction preserve the fields that downstream code expects? A successful HTTP response proves very little.

I treat HTTP 429 as a control-flow event, not as a reason to hammer the endpoint. The worker should apply bounded retries, preserve the original record ID, and distinguish retryable transport failures from permanent validation failures. If the same record is submitted twice, an idempotency key or a deterministic result key should prevent the second result from silently replacing the first.

## What should a Node.js batch experiment measure before changing the API path?

Run the same corpus through two small lanes. The realtime lane sends bounded requests and records each response as it arrives. The async lane creates an input manifest, submits work as one tracked unit, waits for completion, and reconciles exported results to that manifest. Do not compare a finished async job with an incomplete realtime queue; compare usable outputs for the same accepted inputs.

The manifest can be deliberately boring. Boring is good here.

Count twice.

```ts
type KnowledgeRecord = {
  id: string;
  text: string;
};

type Measurement = {
  id: string;
  lane: "async" | "realtime";
  submittedAt: string;
  completedAt?: string;
  inputTokens?: number;
  outputTokens?: number;
  validStructure: boolean;
  accepted: boolean;
  failureClass?: "retryable" | "permanent" | "validation";
};

function startMeasurement(
  record: KnowledgeRecord,
  lane: Measurement["lane"],
): Measurement {
  return {
    id: record.id,
    lane,
    submittedAt: new Date().toISOString(),
    validStructure: false,
    accepted: false,
  };
}

function acceptStructuredResult(
  measurement: Measurement,
  result: unknown,
): Measurement {
  if (!isKnowledgeAnswer(result)) {
    return { ...measurement, failureClass: "validation" };
  }

  return {
    ...measurement,
    completedAt: new Date().toISOString(),
    validStructure: true,
    accepted: true,
  };
}

function isKnowledgeAnswer(value: unknown): value is {
  summary: string;
  tags: string[];
  fields: Record<string, string>;
} {
  if (!value || typeof value !== "object") return false;
  const answer = value as Record<string, unknown>;

  return (
    typeof answer.summary === "string" &&
    Array.isArray(answer.tags) &&
    answer.tags.every((tag) => typeof tag === "string") &&
    typeof answer.fields === "object" &&
    answer.fields !== null
  );
}
```

This sample intentionally stops at the contract boundary. A real client must use the selected service's documented submission and retrieval operations; inventing a route from familiar REST naming is an easy way to make a cost experiment invalid. The experiment code should also log the exact model configuration, prompt version, schema version, accepted input count, valid output count, elapsed time, retry count, and observed billing metadata when that metadata is available.

For each lane, calculate at least four rates: accepted outputs per submitted input, structurally valid outputs, records requiring intervention, and total elapsed time until the result was searchable. Calculate cost per accepted record, not cost per request. The denominator matters. A run that needs a second pass for malformed extraction may lose its apparent advantage after retries and review are included.

I'm not sure any public rate card can predict the bill for an unfamiliar corpus. Input length, generated length, model selection, rejected records, and retry behavior all move the result. A measured slice resolves that uncertainty better than a single headline price.

## The failure modes are more expensive than the request

The first failure mode is silent incompleteness. A parent job reaches a terminal state, but one source ID has no corresponding output. The search index then looks healthy while one part of the private corpus is missing. Reconciliation must fail closed: compare the submitted manifest with the result manifest, retain unmatched IDs, and expose them for a bounded retry or review queue.

The second is schema drift. A prompt may still produce readable prose after a field is renamed, an enum changes, or a nested value becomes nullable. Treat the schema as a versioned application contract. Store the raw result for diagnosis, but write to the searchable index only after validation and normalization.

The third is duplicate work. A process restart can resubmit records whose requests were accepted before the process died. Stable keys make retries observable. They also let the cost report answer a question that aggregate token totals cannot: how many records were paid for twice?

The fourth is deadline failure. Deferred work can be operationally neat and still miss the moment when a user expects a fresh answer. Put a deadline on each import, measure age at publication, and fall back to a realtime path only when the product explicitly allows it. A fallback should be visible in metrics; otherwise the application will quietly pay the interactive rate for most of the supposedly deferred workload.

For private material, data handling is part of correctness. Decide which fields may leave the system, how long request and result data are retained, which region and access controls apply, and how deletion propagates. HIPAA's Security and Privacy Rules are a useful reminder that regulated workflows carry organizational and contractual duties; a model response alone cannot establish compliance with 45 CFR Part 164. The same discipline helps non-regulated developer tools avoid putting secrets or incident details into a shared test corpus.

## A decision rule for small teams

Choose async processing when the deadline is flexible, the corpus is large enough to amortize job orchestration, the output can be reconciled by stable ID, and the same quality gate applies to both lanes. The strongest case is a repeatable import or nightly metadata refresh where an incomplete result is visible and repairable.

Choose realtime calls when the result changes a live interaction, when the workload is too small or irregular to justify a second lane, or when a missing record must be repaired before the user can continue. The catch is operational complexity: a realtime path may look simpler in a short script, but production code still needs bounded concurrency, retry policy, persistence, and monitoring.

Do not make a lower quoted API cost the primary decision. It can be one input to cost per accepted record, alongside queue operations, retries, storage, and review. Your mileage may vary with document length and output shape; publish the measurement window and the acceptance criteria with the result so a future prompt or model change does not turn an old comparison into folklore.

The sensible first run is small: enough records to include short notes, long references, malformed source text, duplicate-looking content, and every structured field the product needs. Ship the measurement harness before committing the architecture.

## References

- AWS Bedrock official page: https://aws.amazon.com/bedrock/
- 45 CFR Part 164 (HIPAA Security and Privacy Rules, eCFR): https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164

## Further reading

- https://aws.amazon.com/bedrock/
- https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164
