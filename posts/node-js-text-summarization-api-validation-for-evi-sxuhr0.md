# Node.js Text Summarization API Validation for Evidence Checked Sales Actions

A lower text summarization API cost can be a poor bargain if a model's output needs manual repair before it can become a CRM action. For a B2B SaaS sales workflow, make structured output correctness the admission rule: accept a proposal only after its fields and evidence pass validation, then compare spending across the proposals that survive. TL;DR: measure the cost of accepted call summaries, include failed attempts, and keep CRM writes behind a separate approval boundary. A quoted token rate alone cannot settle that decision.

The proposed flow is deliberately small. Give each transcript segment a stable ID, request a summary and action proposals, parse the response into an unknown value, and check it against an application contract. Put passing proposals in a review queue. Record rejected attempts in the same evaluation ledger, because excluding them would make a weak candidate look artificially efficient. Approval and CRM mutation happen afterward.

## A runnable acceptance check for call summaries

Start with the boundary you own. The following TypeScript example runs locally without a model connection; it demonstrates acceptance and rejection using an explicitly fictional sales call. Run it through your existing TypeScript runner or compile it with your project's TypeScript setup. Every identifier, quote, and action below belongs to this example, rather than to a measured deployment.

The contract permits one action kind: a request to send material. It has no deadline or owner field. That restriction is intentional. Adding fields that the transcript does not establish creates more opportunities for confident guesses, while a narrow contract is enough to exercise the review path.

```ts
type Segment = { id: string; text: string };
type Action = {
  kind: "send_material";
  description: string;
  segmentId: string;
  quote: string;
};
type Proposal = { summary: string; actions: Action[] };

const transcript: Segment[] = [
  { id: "s1", text: "Please send the security questionnaire." },
  { id: "s2", text: "We have not agreed on a purchase date." },
];

function object(value: unknown): Record<string, unknown> {
  if (typeof value !== "object" || value === null || Array.isArray(value)) {
    throw new Error("Expected an object");
  }
  return value as Record<string, unknown>;
}

function exactKeys(value: Record<string, unknown>, keys: string[]): void {
  const actual = Object.keys(value);
  if (actual.length !== keys.length || keys.some(key => !(key in value))) {
    throw new Error("Unexpected or missing fields");
  }
}

function nonempty(value: unknown): string {
  if (typeof value !== "string" || value.trim().length === 0) {
    throw new Error("Expected nonempty text");
  }
  return value;
}

function validate(raw: string, segments: Segment[]): Proposal {
  const parsed: unknown = JSON.parse(raw);
  const root = object(parsed);
  exactKeys(root, ["summary", "actions"]);
  const summary = nonempty(root.summary);
  if (!Array.isArray(root.actions)) throw new Error("Expected actions array");
  if (root.actions.length > 3) throw new Error("Too many proposed actions");

  const seen = new Set<string>();
  const actions = root.actions.map((item: unknown): Action => {
    const action = object(item);
    exactKeys(action, ["kind", "description", "segmentId", "quote"]);
    if (action.kind !== "send_material") throw new Error("Unknown action kind");
    const description = nonempty(action.description);
    const segmentId = nonempty(action.segmentId);
    const quote = nonempty(action.quote);
    const source = segments.find(segment => segment.id === segmentId);
    if (!source || !source.text.includes(quote)) {
      throw new Error("Evidence does not match the referenced segment");
    }
    const key = JSON.stringify([action.kind, description, segmentId, quote]);
    if (seen.has(key)) throw new Error("Duplicate proposal");
    seen.add(key);
    return { kind: "send_material", description, segmentId, quote };
  });
  return { summary, actions };
}

const candidate = {
  summary: "The buyer requested security material; no purchase date was agreed.",
  actions: [{
    kind: "send_material",
    description: "Send the security questionnaire.",
    segmentId: "s1",
    quote: "Please send the security questionnaire.",
  }],
};

console.log(validate(JSON.stringify(candidate), transcript));
try {
  validate(JSON.stringify({ ...candidate, approved: true }), transcript);
  throw new Error("The validator accepted an unexpected field");
} catch (error) {
  if (!(error instanceof Error) || error.message !== "Unexpected or missing fields") {
    throw error;
  }
  console.log("Rejected the unexpected approval field");
}
```

This checker does less than a CRM integration, and that makes its boundary inspectable. It requires the expected keys, checks field types, limits a response to 3 actions, rejects exact duplicate proposals, and verifies that each evidence quote occurs in its referenced segment. The number 3 is an example application limit, not a model capability or an industry benchmark. An empty action array is valid; a conversation can end without an actionable commitment.

Keep it narrow.

The main limitation is that evidence matching cannot establish whether an action follows from the conversation. Substring matching is also sensitive to transcript edits, so keep the evaluated transcript version with the proposal. The duplicate check catches identical objects; it does not recognize two descriptions that mean the same thing. This example assumes segment IDs are unique in application-owned input. Validate that assumption when ingesting a transcript. Consider a response whose description says to schedule a purchase while its quote reproduces the buyer's refusal to agree on a date. Both strings can satisfy their field checks, and the quote can exist verbatim in the transcript, while the proposed action reverses the meaning. Rejecting that proposal requires semantic evaluation or a reviewer. This design is not suitable for unattended CRM updates: its trade-off is a small, inspectable validator at the expense of a remaining human review step. Those details matter more here than adding another adapter before the first workflow has an acceptance test.

## Does valid JSON make a CRM action correct?

No. Parsing answers a syntax question. Field validation answers a contract question. Neither establishes that a salesperson actually promised to send a document, that a buyer accepted a deadline, or that the summary preserved a qualification. The example's evidence check establishes a narrower property: the quoted words occur in the supplied segment. A proposal could quote the sentence about an unagreed purchase date and still describe a purchase commitment. It would pass the mechanical check and deserve rejection during review.

Treat evidence as something a reviewer can inspect, not as a correctness certificate. In the review interface, show the proposed action beside the referenced segment and its neighboring conversation. Keep approval outside model-generated fields. A string saying that approval has occurred should have no authority to authorize a CRM write; in this contract, the extra field is rejected outright.

For evaluation, label more than the happy path. Include an explicit request, a negated commitment, a tentative suggestion, and a call with no action. These are proposed fixture categories, not claims about their frequency in sales calls. Check summary fidelity separately from action fidelity: a readable summary can accompany a fabricated task, and a correct task can accompany a misleading summary.

This is also where omission needs its own measurement. Accepting an empty action array prevents forced invention, but a candidate that always returns an empty array must not win the comparison. Give each fixture an expected set of actions, and review both unsupported additions and missed commitments. Only then does an accepted output mean something useful.

## How should correctness affect a text summarization API cost comparison?

The search for the cheapest text summarization API needs a workload and an acceptance rule before it needs a price table. For this workflow, use the same frozen call set, transcript boundaries, output contract, and review rubric across candidates. Keep input and output usage separate, retain the model identifier reported for each run, and record the date of the comparison. Avoid silently substituting character counts when token usage is missing. Mark the measurement incomplete instead.

Use an evaluation ledger with one row per attempt and a shared ID for all attempts on the same call. Record the configured candidate, usage, billed amount when available, elapsed time, validation result, and review outcome. This is a proposed measurement design; it does not assume a particular provider response shape. Define an accepted call as one whose summary and action set pass the rubric, including calls correctly judged to have no action.

Compute spend per accepted call as total evaluation spend divided by accepted calls. Include rejected attempts and retries in the numerator.

Zero accepted calls means no usable result.

If no calls pass, report no accepted result instead of a zero unit cost. Keep the count of accepted calls visible next to the ratio, because a candidate that abandons difficult calls can otherwise look attractive on its surviving subset.

That denominator changes the decision.

A lower request charge can lose its appeal if many outputs need another attempt or human repair, but the direction and size of that effect require measurements on your workload. Report review minutes separately from API spend unless you have an explicit, documented way to value that time. Don't invent a combined savings percentage.

Latency belongs beside correctness, too. Measure the interval through validation and review readiness, not just the generation interval. Limit repair attempts in the experiment; otherwise, an apparently successful candidate may consume an open-ended amount of time. Use the same repair allowance for each candidate. The resulting comparison answers an operational question: how much work reaches the review queue within the application's waiting budget?

## Shipping the review boundary

Before enabling CRM writes, replay the fixed fixtures through the deployed parser and validator. The local example tests only an unexpected field; extend the evaluation fixtures to cover missing evidence, an unknown segment, duplicate proposals, malformed JSON, and valid outputs that are semantically wrong. Track each failure category separately so a contract change cannot hide a loss of summary fidelity inside an improved parse rate.

Keep generation retries separate from write retries. Give the approved action a durable application identifier, record its destination and write state, and check the destination's actual duplicate-prevention behavior before enabling automatic retry. A unique local identifier alone does not prove that a remote CRM will suppress repeated writes. If the result of a write is uncertain, reconcile its status before attempting another mutation.

For a small team, ship the review queue first and automate only the fields the evaluation can defend. Version the transcript, prompt, contract, and validator in each evaluation record. When one changes, replay the same fixtures before rollout and inspect a sample of passing results as well as failures. Log rejection categories and timing while keeping transcript text out of routine metrics; retain the evidence through an access-controlled review record.

The release check is practical: the parser rejects malformed responses, the validator enforces the contract, the reviewer can inspect the evidence, and the writer can reconcile uncertain mutations. The comparison ledger must include failed work and correctly empty calls. Once those conditions hold, spending and waiting time can guide the choice among candidates that meet the correctness bar.

## Sources

References for the parsing and type-boundary mechanisms used in the example. The acceptance rules, fictional transcript, and measurement procedure above are application design choices.

- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse
- https://www.typescriptlang.org/docs/handbook/2/narrowing.html
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/includes
