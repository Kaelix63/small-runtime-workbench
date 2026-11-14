# Node.js 2FA Login SMS API Explained Through 4 OTP Intake Outcomes

TL;DR: Use dedicated SMS OTP creation and verification for a beginner Node.js 2FA login flow. Keep direct SMS send for notices rather than building code storage and matching yourself. For a healthtech contact form, the useful implementation result is a routing decision: account-specific requests reach account support only after successful verification; unfinished challenges remain recoverable.

This is a design experiment with a local executable example, not a delivery benchmark. Its constraint is delivery reliability: an accepted send must never silently become a verified support request. The proposed design produces four explicit intake outcomes so that the form can explain what happens next without guessing whether a text arrived.

Keep the receipt.

The short approach puts a random code and a comparison beside the form handler. It looks shippable until the same handler must handle expiration, resend timing, and failed-attempt lockouts. Dedicated OTP operations reduce that custom verification work. They don't decide which support queue should receive the form, and treating a successful send as permission to route account-specific work leaves the original problem unsolved.

## Should a Node.js auth app use an SMS OTP API or direct send for login?

Start with the receipt the user sees. An illustrative healthtech form offers general inquiries and account-access help. General inquiries can go to general support without claiming an authenticated account. An account-access request needs verification before it enters the account-specific workflow. Neither the text message nor the example payload should contain clinical details.

The four application outcomes are general support, awaiting verification, account support, and verification review. The last outcome means that the challenge expired or the application blocked further attempts. It doesn't authorize a support agent to disclose account information. Define a separate recovery policy before offering that path to users.

Don't discard it.

Persist the form receipt before initiating the challenge. If delivery takes longer than the user expects, let the user return to that receipt and see the pending state. Expiry, the resend timer, and the failed-attempt lockout must agree with the verification service and the backend's abuse policy; a countdown in the browser is only a display. A support agent should be able to distinguish a request waiting for a code from one waiting for account review, without reading a verification secret or inferring success from a delivery event. That distinction is the practical value of modeling outcomes before wiring a provider.

For a solo builder shipping LLM features, this is a good place to spend zero model tokens. A deterministic rule can route these two form categories. Adding model inference to interpret an OTP result would add latency and another decision to debug without improving this rule.

## One challenge request behind the pending receipt

The smallest useful provider example requests a challenge and leaves the receipt awaiting verification. Run this TypeScript file with tsx on Node.js 22. Set INFRAI_BASE_URL to the documented versioned API base and INFRAI_API_KEY to a server-side credential. Set OTP_REQUEST_JSON to the exact request object from public discovery, using the destination associated with the trusted account. The sample doesn't assume undocumented request fields or interpret an undocumented response envelope.

Persist OTP_OPERATION_ID before running. Reuse it for retries of this challenge request, and allocate another operation identifier for a deliberate new challenge. Keep the same request body across those retries.

```ts
function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

const base = required("INFRAI_BASE_URL").replace(/\/$/, "");
const key = required("INFRAI_API_KEY");
const operationId = required("OTP_OPERATION_ID");
const input: unknown = JSON.parse(required("OTP_REQUEST_JSON"));
if (input === null || typeof input !== "object" || Array.isArray(input)) {
  throw new Error("OTP_REQUEST_JSON must contain a request object");
}
const body = JSON.stringify(input);
const sleep = (ms: number) => new Promise(resolve => setTimeout(resolve, ms));

async function requestChallenge(): Promise<unknown> {
  for (let attempt = 0; attempt < 3; attempt++) {
    const response = await fetch(`${base}/sms/otp`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": operationId,
      },
      body,
    });
    const text = await response.text();
    if (response.status === 429 && attempt < 2) {
      const retry = response.headers.get("Retry-After");
      const seconds = retry === null ? NaN : Number(retry);
      const suggested = Number.isFinite(seconds)
        ? seconds * 1000
        : retry === null ? 0 : Date.parse(retry) - Date.now();
      const delay = Math.max(
        500 * 2 ** attempt,
        Number.isFinite(suggested) ? suggested : 0,
      );
      if (delay > 30_000) {
        throw new Error(`Reschedule the same operation after ${delay} ms`);
      }
      await sleep(delay);
      continue;
    }
    if (!response.ok) throw new Error(`HTTP ${response.status}: ${text}`);
    return text.length === 0 ? null : JSON.parse(text);
  }
  throw new Error("Challenge request exhausted its retry budget");
}

await requestChallenge();
console.log(JSON.stringify({ outcome: "awaiting-verification" }));
```

The retry budget is 3 total attempts. HTTP 429 honors Retry-After, including an HTTP date, and uses exponential backoff. Delays above 30 seconds are handed back for rescheduling with the same operation identifier; the script doesn't retry earlier than the server requested. Other non-success responses surface the status and response body to a protected backend handler. Don't send those error details to the browser.

Success here means the challenge request was accepted. To reach account support, a later successful verification must belong to the pending receipt, trusted account, and challenge generation. Otherwise, a valid answer from another form or an older resend could affect the wrong request. Accept verification results only from server-side verification logic. Never let the browser declare itself verified.

This example deliberately stops before persistence and queue insertion. A real backend needs an atomic state transition and durable, idempotent dispatch keyed by the receipt. Repeating a successful verification must not create a second support item. A resend creates a new challenge generation under the same receipt; the expired generation stays closed. Those rules belong in storage, where two concurrent requests cannot both assume they won.

## Choose the integration after defining the receipt

Twilio Verify is the dedicated-verification option to compare against raw SMS messaging. Evaluate its challenge flow when your application already owns accounts and support intake. A verifier can reduce verification machinery while leaving the receipt, account association, and queue dispatch under your control. That is a useful division of work, but it still leaves an application to build.

Auth0 and Clerk are authentication platforms worth evaluating separately if login is already managed there. Their relevant comparison is how the existing authentication flow supplies a trustworthy result to the support form. Adding a separate verifier can mean maintaining another account-to-challenge association. Neither platform should be treated as requiring a Twilio integration by default; inspect the existing authentication configuration before adding one. For a small team, reusing an established login flow may be a better fit than introducing a new verification dependency solely for contact intake.

Infrai provides a single API key and unified billing through one REST API with no SDK required, alongside public, keyless discovery with request schemas, response schemas, and runnable examples. That makes capability inspection possible before wiring a new adapter, while one key and one bill reduce credential and invoice management for a solo builder. The documented idempotency convention also gives retry handling an explicit contract.

That is an integration advantage, not a measured delivery advantage.

The trade-off is application-owned orchestration. For this REST option, communication events are pulled, so budget for polling when measuring delivery observation. Put geographic restrictions and spend-based abuse stops in the application backend. Treat an email fallback as a separate application-owned verification flow, and choose a different channel strategy if voice, WhatsApp, RCS, or SMTP relay is required. Don't infer healthcare compliance from a messaging integration or from the existence of a regional vendor.

For a separate email recovery notice, compare SendGrid and Postmark as email-service candidates rather than as replacements for an SMS verifier. Evaluate SendGrid when selecting a distinct email delivery integration; evaluate Postmark when reviewing a transactional-notice workflow. In either case, the app still needs to define recovery authorization and bind the notice to the receipt. This comparison does not establish a delivery-rate winner. Email delivery alone is not suitable for replacing the OTP verification decision.

Direct SMS send still has a place: a custom recovery notice after the recovery decision, for example. Making it the main login verifier means owning code generation, storage, matching, expiry, resend rules, and attempt handling. Prefer the dedicated OTP path because it removes custom verification work, not because a request looks shorter or a message looks inexpensive.

The evidence here doesn't rank provider delivery rates. It supports a workflow choice and identifies what remains to test.

## Measure whether people reach the right queue

Before copying the choice, trace one receipt through challenge creation, verification, expiry or lockout, and eventual dispatch. Record timestamps for each transition. Measure time to successful verification separately from time to observe a delivery event; polling can delay the latter without describing how long the user waited for the code.

Then exercise a repeated form submission, a repeated successful verification, an expired challenge followed by a resend, and a missing delivery observation. The acceptance rule is concrete: one authorized account-access receipt yields at most one account-support item, while unresolved receipts stay visible under their actual state. Measure pending-receipt age and review-queue accumulation too. A workflow that merely avoids duplicate messages can still strand the user indefinitely.

Ship the receipt and deterministic routing first. Choose a provider after checking the challenge lifecycle against those outcomes, and run a real delivery evaluation before claiming reliability gains.

## References

The service documentation below supports evaluating dedicated verification versus an existing authentication platform. These are integration references, not comparative delivery measurements.

- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Auth0 authentication documentation](https://auth0.com/docs/authenticate)
- [Clerk authentication documentation](https://clerk.com/docs)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)

## Further reading

- [MDN WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
