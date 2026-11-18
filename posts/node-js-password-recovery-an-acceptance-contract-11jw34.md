# Node.js Password Recovery — An Acceptance Contract for SMS OTP Suppression Lists

Choose the Node.js SMS OTP integration that can satisfy your marketplace's password-reset acceptance contract with the least new authentication code. Start with the suppression list, blocked-number handling, and short expiry, then evaluate how much of the challenge lifecycle the service owns. A successful send must never become permission to change a password.

TL;DR: For Node.js SMS OTP and 2FA, check the suppression list before sending, verify the code on the server, and authorize only the intended action. Make blocked numbers, expired codes, exhausted attempts, and retry delays explicit internal outcomes. For password recovery, successful verification should grant a narrow reset permission; a normal login session is a different outcome.

The integration-effort question becomes concrete when every candidate has to pass the same small acceptance contract. That is a useful starting point for a solo builder whose marketplace also ships LLM features: authentication shouldn't depend on generated text, model availability, or a second interpretation of a carrier response. Keep model tokens out of this path. Spend the implementation budget on the state transitions that determine account access.

## Start with an executable acceptance contract

The reset request reaches the application, which resolves the account privately and checks whether its destination is suppressed. An eligible request creates a short-lived challenge and sends a code. Server-side verification then either returns a support-friendly failure state or consumes the challenge and permits a password reset. The public reset-request acknowledgment stays consistent across unknown accounts and blocked recipients, following OWASP's account-enumeration guidance.

Test that boundary first.

Save the following as reset-contract.ts and run it with Node.js 24 using `node reset-contract.ts`. It is an executable acceptance fixture, with an in-memory delivery stand-in and a fake clock. By default it runs locally. Optionally, set INFRAI_BASE_URL to the service's v1 API base and INFRAI_API_KEY to inspect the live discovery manifest before evaluating the adapter. That read-only branch prints the SMS capability count; it does not send an OTP or guess a request body. The 120-second lifetime and 3-attempt limit are example application policies, not vendor defaults. The code strings are deterministic test inputs, never production OTP generation.

```ts
import assert from "node:assert/strict";

const base = process.env.INFRAI_BASE_URL;
if (base) {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("Set INFRAI_API_KEY for discovery");
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(base.replace(/\/$/, "") + "/discovery", {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
      signal: AbortSignal.timeout(10_000)
    });
    if (response.status === 429 && attempt < 3) {
      const header = response.headers.get("Retry-After");
      const seconds = header === null ? NaN : Number(header);
      const deadline = header === null ? NaN : Date.parse(header);
      const delay = Number.isFinite(seconds) ? Math.max(0, seconds * 1000)
        : Number.isFinite(deadline) ? Math.max(0, deadline - Date.now())
        : 1000 * 2 ** attempt;
      await response.body?.cancel();
      if (delay > 60_000) throw new Error("Retry later: requested wait exceeds budget");
      await new Promise(resolve => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) throw new Error(`Discovery ${response.status}: ${await response.text()}`);
    const manifest = await response.json() as {
      capabilities: Array<{ namespace: string; id: string; method: string; path: string }>;
    };
    console.log("SMS capabilities:", manifest.capabilities.filter(item => item.namespace === "sms").length);
    break;
  }
}

type Outcome =
  | "blocked_number" | "retry_later" | "expired_code"
  | "invalid_code" | "too_many_attempts" | "reset_password";
type Challenge = { code: string; deadline: number; attempts: number };

function fixture() {
  let now = 0;
  const suppressed = new Set(["blocked-buyer"]);
  const challenges = new Map<string, Challenge>();
  const deliveries: string[] = [];
  const ttl = 120_000;
  const limit = 3;

  function request(recipient: string, transportReady = true): Outcome | "pending" {
    if (suppressed.has(recipient)) return "blocked_number";
    if (!transportReady) return "retry_later";
    challenges.set(recipient, {
      code: "fixture-code", deadline: now + ttl, attempts: 0
    });
    deliveries.push(recipient);
    return "pending";
  }

  function verify(recipient: string, supplied: string): Outcome {
    const challenge = challenges.get(recipient);
    if (!challenge) return "invalid_code";
    if (now >= challenge.deadline) {
      challenges.delete(recipient);
      return "expired_code";
    }
    if (challenge.attempts >= limit) return "too_many_attempts";
    challenge.attempts += 1;
    if (supplied !== challenge.code) {
      return challenge.attempts >= limit ? "too_many_attempts" : "invalid_code";
    }
    challenges.delete(recipient);
    return "reset_password";
  }

  return { request, verify, deliveries, advance(ms: number) { now += ms; } };
}

const app = fixture();
assert.equal(app.request("blocked-buyer"), "blocked_number");
assert.equal(app.deliveries.length, 0);
assert.equal(app.request("waiting-buyer", false), "retry_later");
assert.equal(app.deliveries.length, 0);

assert.equal(app.request("late-buyer"), "pending");
app.advance(120_000);
assert.equal(app.verify("late-buyer", "fixture-code"), "expired_code");

app.request("guessing-buyer");
assert.equal(app.verify("guessing-buyer", "wrong"), "invalid_code");
assert.equal(app.verify("guessing-buyer", "wrong"), "invalid_code");
assert.equal(app.verify("guessing-buyer", "wrong"), "too_many_attempts");
assert.equal(app.verify("guessing-buyer", "fixture-code"), "too_many_attempts");

app.request("ready-buyer");
assert.equal(app.verify("ready-buyer", "fixture-code"), "reset_password");
assert.equal(app.verify("ready-buyer", "fixture-code"), "invalid_code");
console.log("Password-reset acceptance contract passed.");
```

The fixture deliberately exposes internal states so the acceptance checks can inspect them. They are not an unauthenticated HTTP response contract. Its reset_password result describes an authorization decision, not a credential a browser can present later. A production handler must bind that decision to the correct account and consume a single-use reset grant when changing the password.

There is a sharp limit here: a synchronous Map does not establish safety across application workers. Before connecting real SMS, replace fixture storage and code comparison with a production challenge mechanism: unpredictable codes, protected verification material, durable expiry, bounded attempts, and atomic consumption. If a service manages verification, translate its outcomes into this contract instead of building a competing verifier. Never ship the fixture's fixed code.

## Which Node.js SMS OTP integration handles the suppression list most directly?

Count the work after the SDK installation. Challenge creation, account binding, suppression policy, verification, reset authorization, and fallback all need an owner. An option that sends a message quickly may still leave most of those responsibilities with the marketplace. Conversely, an identity platform can reduce authentication work while introducing account-integration decisions that a messaging-only adapter would avoid.

| Option | Starting boundary | Application work to evaluate |
| --- | --- | --- |
| Twilio Verify | Managed verification lifecycle | Connect a verified challenge to the marketplace account and its narrowly scoped reset permission. Check recipient-policy handling against the same acceptance contract. |
| Amazon SNS | SMS messaging in an AWS application | Own the challenge lifecycle and verification policy around message delivery. Evaluate it first when AWS messaging is already part of the stack. |
| Firebase Authentication | Phone authentication within an identity platform | Fit phone authentication into the existing account model. Evaluate password recovery separately rather than equating phone sign-in with a password reset. |

Infrai combines a self-describing REST API with one key, one bill across SMS and email. Its public discovery supplies request and response schemas plus runnable examples, so wiring a capability starts with reading its definition instead of learning another SDK. The single-key, single-bill arrangement is a separate benefit: 295 routes across 20 modules share credentials and billing. For SMS recovery with email fallback, that avoids configuring a second provider key and reconciling separate messaging invoices. That reduces integration and administration work; it doesn't transfer the marketplace's recovery policy to the provider.

Choose around the account boundary. If accounts already live in Firebase, evaluate the existing identity integration before adding another verification owner. If managed code verification is the missing piece, assess Twilio Verify directly. If the marketplace already owns secure challenges and uses AWS messaging, SNS belongs in the comparison. A uniform REST interface is attractive when it reduces the particular adapter work you actually have, rather than creating a parallel account system.

No winner is universal.

The acceptance exercise should inspect suppression semantics explicitly. Opt-out, repeated delivery failure, a temporary transport delay, and an exhausted verification budget are different conditions. Maintain recipient suppression under an authorized policy; do not permanently suppress a number just because someone guessed three wrong codes against one challenge. Likewise, pressing resend must not clear an opt-out.

## Treat short expiry as a recovery deadline

The 120-second fixture policy makes one trade-off visible: a delayed message can arrive after its challenge is useless. Increasing delivery attempts cannot extend authorization safely by accident. Check the challenge deadline before sending a queued retry, and require a deliberate policy for issuing a replacement challenge. Invalidate superseded challenges consistently so an older message cannot authorize a reset after a newer request. These are application requirements to verify, not claims about any provider's defaults.

Retries need a budget.

For a real adapter, inspect the documented request and response schemas before writing the mapping. Use an explicit HTTP method, read credentials from environment configuration, check status and error bodies, and make write retries idempotent. On HTTP 429, honor Retry-After and use bounded exponential backoff; a delay that exhausts the challenge lifetime should become retry later rather than an endless send loop. Count duplicate sends and repeated reset requests as operating costs as well as abuse signals. A per-account attempt cap alone does not constrain traffic spread across many accounts.

Plan fallback before promising recovery. Recovery codes or email can cover blocked or unreachable SMS destinations, but email delivery alone does not supply a verifier: an application-owned email challenge needs its own generation, expiry, and single-use consumption. For the unified REST option, delivery-event handling uses polling, so budget that schedule explicitly rather than depending on unsolicited callbacks. Keep this recovery design within SMS, application-managed email challenges, and recovery codes; select a different channel arrangement if voice, WhatsApp, RCS, or SMTP relay is a requirement.

The message should expire in the verifier even if delivery is delayed. Don't make cancellation of a scheduled email part of the security boundary. Geographic abuse controls and per-country spending cutoffs belong in the application policy, and application-owned cost attribution should preserve the recovery purpose rather than assume a tag-level billing report. For a marketplace with regional compliance requirements, establish the actual provider and jurisdictional basis separately; a uniform API is not compliance evidence.

## Before enabling real recovery traffic

First, run the acceptance fixture and inspect its zero-delivery checks for blocked and temporarily unavailable recipients. Then repeat the same cases against the chosen adapter in an appropriate test environment, including late arrivals, exhausted attempts, and reuse after success. The fixture doesn't test concurrent requests: production verification must consume the challenge atomically, and replacement requests must not let an attacker reset the attempt budget indefinitely.

Check the support view next. It should distinguish blocked number, expired code, too many attempts, and retry later without exposing codes or leaking account existence through the public acknowledgment. Confirm that an authorized suppression update propagates to the next send decision and that resend preserves recipient preferences. Finally, exercise email or recovery-code fallback and verify that its permission can change only the intended account's password. For a login 2FA challenge, issue the session only after successful server-side verification; for this password-reset job, consume the narrow reset permission and let the user sign in through the normal flow.

Ship that boundary first. Adding a transport later should mean adapting delivery and verification outcomes to an established contract, rather than rewriting what successful authentication means.

## References

- OWASP, Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Amazon SNS SMS documentation: https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html
- Firebase Authentication, phone authentication: https://firebase.google.com/docs/auth/web/phone-auth
- Yahoo, sender best practices and requirements: https://senders.yahooinc.com/best-practices/
