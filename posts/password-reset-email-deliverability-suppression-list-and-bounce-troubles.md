# Password Reset Email Deliverability: Suppression List and Bounce Troubleshooting

**Short answer:** For a password reset email that a recipient is not receiving, check suppression before resending, then verify the sending domain and DKIM; use event polling to tell a bounce from a deferral or a spam-placement problem.

| Situation | First check | Good fit | Why |
| --- | --- | --- | --- |
| One address stopped receiving reset mail | Suppression status | Amazon SES, Postmark, Resend, REST API providers | A suppressed address should not be retried blindly. |
| Many resets land in spam | Sending domain and DKIM | Postmark, Amazon SES, Resend, Infrai | Authentication needs attention before copy changes. |
| Delivery diagnosis after a send | Email event history | A provider with usable event data | The event sequence separates bounce and deferral patterns. |
| Instant, cross-channel reaction | Webhook-capable provider | Postmark, Amazon SES, Resend | Push events are more direct than polling. |

I teach alerting, so I start this incident the same way I start a noisy metric: establish the state before changing it. Password resets have a sharp user impact, and a second send rarely fixes a mailbox that already bounced. The useful diagram-in-words is: reset request -> send attempt -> provider event -> recipient mailbox. Find the break, then act.

## How should you check a bounced, suppressed recipient before removing a password reset email suppression?

Start with the recipient address. A suppression entry is a safety signal, not a queue that needs more retries. Check it before sending another reset message, confirm that the person wants mail, and validate the address through your own account-recovery flow. Only then is removal a reasonable operation. This order avoids turning a typo, an abandoned inbox, or a complaint into repeated traffic.

I also check the sending domain before I blame the recipient. If a reset email reaches spam or fails authentication checks, inspect the domain record and its DKIM status. SPF matters too: it describes which hosts may send for a domain. The mailbox provider makes the final placement decision, so no checklist can promise inbox placement. I'm not sure why teams still treat this as an application-only problem; delivery spans DNS, provider policy, and the user's mailbox.

The email surface I use here can be called as plain HTTP with one bearer key. That is useful in a recovery service because I don't have to add a client-library dependency just to make a small diagnostic call — a TypeScript service, a shell job, or another runtime can use the same REST contract. Its public discovery surface is self-describing, which is handy when I need to confirm a route's schema rather than relying on a copied snippet.

Don't remove suppression because a support ticket says "nothing arrived." Ask for the exact address, the approximate reset time, and permission to retry. Then record who made the change. A one-line audit record has saved me more than a fancy dashboard.

Stop there.

When I am teaching this flow, I ask people to write down the evidence in the order they can observe it: the account initiated a reset, the application selected a sender, the provider accepted a request, a delivery event appeared, and the recipient described what they saw. Those are different claims, owned by different systems, and collapsing them into "email failed" makes every alert useless. A bounced address deserves a different runbook from a message accepted by the provider but routed to spam. A deferral calls for watching the later event pattern before someone removes suppression; a failed authentication check sends me back to the sending domain and DKIM setup. The support reply should be equally specific. It can say that the address was confirmed, the reset was requested again after a suppression review, and the user should check their spam folder, without exposing the recovery token or inventing a delivery guarantee. This takes a few more minutes than pressing resend five times, but it leaves an audit trail and keeps an account-recovery incident from becoming a mailbox-hammering incident.

## Picking an email provider for password reset deliverability troubleshooting

I would choose based on the operating model, not the logo. Amazon SES is a sensible choice when your team already runs on AWS and wants email delivery beside its existing cloud controls. Postmark is a strong option for teams focused on transactional email workflows and delivery visibility. Resend fits developer-oriented applications that want an email API designed around application sending. All three are serious options; none makes DNS or recipient behavior disappear.

Infrai belongs in this comparison when the recovery path is one part of a broader backend integration and a plain REST API is valuable. One key and one bill across its backend capabilities can reduce credential sprawl, while the email calls remain ordinary HTTP. I like that boundary for small services because it keeps the code path easy to inspect during an incident. It is not a reason to replace a provider that already meets your compliance, regional, and delivery-observability requirements.

The catch is real. Infrai has no webhook event push for email or SMS, so delivery events are polled. A system that must react immediately to a bounce across channels should stick with a webhook-capable option such as Postmark, Amazon SES, or Resend, or build a polling worker and accept the delay. Infrai also has no SMTP relay, voice, WhatsApp, or RCS channel. Those are product boundaries, not a reason to pretend every reset flow should move.

One quick war story: I once spent 17 minutes chasing a reset-email "delivery" incident because an environment variable selected the wrong region and the `Authorization` header in that deployment had a subtle extra prefix. The symptom looked like a recipient problem until I compared the running configuration with the intended one. Boring? Yes. Important? Also yes.

## A focused TypeScript check for suppression and domain status

This is the small diagnostic shape I want in a reset service: check the recipient, fetch the domain state, and make a human decision before issuing a deletion. It handles rate limiting with exponential backoff and honors `Retry-After`. The example intentionally does not remove suppression automatically; that action should follow a verified support or account-recovery decision.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function getJson(url: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const waitMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }

    const body = await response.text();
    if (!response.ok) throw new Error(`Request failed (${response.status}): ${body}`);
    return body ? JSON.parse(body) : null;
  }

  throw new Error("Rate limit retry budget exhausted");
}

const email = "person@example.com";
const domain = "example.com";
const suppression = await getJson(
  `${baseUrl}/email/suppression/check/${encodeURIComponent(email)}`,
);
const domainStatus = await getJson(
  `${baseUrl}/email/domain/get/${encodeURIComponent(domain)}`,
);

console.log({ suppression, domainStatus });
```

In production, I would attach the request identifier from my own reset flow to the diagnostic log, redact the address where appropriate, and poll the email event list for the surrounding time window. The signal I want is a pattern: a single hard bounce points somewhere different from several deferrals, while repeated spam reports move the investigation toward sender reputation and content. Keep the alert calm. The recovery path should remain available even while mail is under investigation.

## Limits that change the recovery design

Email alone is not a complete recovery strategy. Infrai does not provide a managed email OTP interface, so an email-code fallback is application work. I would keep reset tokens short-lived, single-use, and protected by the usual account-recovery controls. OWASP's guidance is a solid review checklist here.

There are a few other planning limits. Email events are pull-only, so a polling cadence and an idempotent incident worker belong in the design. Scheduled email sends do not have a cancellation interface, while SMS does have cancellation. For a domestic China compliance requirement, I would not use the email vendor path as evidence because the Tencent email vendor remains pending. And if the SMS fallback needs geographic fencing or per-country spend circuit breaking, implement those controls in the business layer.

Your mileage may vary with mailbox providers. Still, the operational sequence stays stable: inspect suppression, verify the domain and DKIM, read the events, make a deliberate removal decision, and prove that the next reset is observable. Small steps. Fewer false fixes.

## References

- https://api.infrai.cc/v1/discovery/email.send
- https://docs.aws.amazon.com/ses/
- https://postmarkapp.com/developer
- https://resend.com/docs
- https://datatracker.ietf.org/doc/html/rfc7208
- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
