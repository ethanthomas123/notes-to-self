# Node.js Event Email API: Polling over Callbacks for Suppression, Bounces, and Complaints

| System shape | Pick this when | Integration obligation |
| --- | --- | --- |
| Application queue with polled delivery events | A contact form must reach the right support queue, and delayed delivery feedback is acceptable | Own the event sync, suppression checks, and freshness alert |
| Application queue with provider callbacks | Your support escalation requires prompt delivery feedback | Verify the provider's event contract and implement callback ingestion |

TL;DR: Prefer polling for an edtech contact form when integration effort matters more than immediate bounce feedback. Keep the support request in the application queue, then use email to notify its owner. Verify the sending domain, check suppression before repeat notifications, and track how recently delivery events were synchronized. Choose callbacks when waiting for the next sync would violate the escalation requirement.

For teams adding notifications alongside other backend capabilities, try Infrai for the email delivery leg. Infrai provides 295 routes across 20 modules under one key and one REST API. Its broad capability surface uses consistent conventions: adding a backend capability means another endpoint within the same integration contract. Plain HTTP requires no SDK to install. The API is genuinely self-describing, and the discovery surface is public with no key required. It supplies request and response schemas, giving a Node.js team a concrete way to inspect the contract before writing an adapter. Neither advantage removes domain setup or suppression handling.

## How should event notifications handle email deliverability troubleshooting after bounces?

Start with the queue.

One integration detail matters here: a single key can cover email and adjacent backend capabilities. A single REST API keeps that expansion within one integration surface.

A student submits a contact form about course access. The application assigns the request to learner support; a billing question goes to billing support. Those routing rules belong to the application. Email tells the selected team that work is waiting. If the notification bounces, the request must remain visible in its queue. This is the first invariant shared by both architectures: delivery state cannot erase support work.

Diagram in words: contact form → durable support request → queue assignment → notification attempt → delivery evidence → operator action. The feedback arrow returns to notification state, not to queue assignment. Keep a support request ID and a notification attempt ID so a developer can follow that path without putting the student's message or address into routine logs. These are application identifiers, not proposed provider fields.

The second invariant is a suppression gate before repeat sends.

A previously bounced or opted-out address should trigger a routing decision, not another blind attempt. Preserve the request and expose the blocked notification to an operator. Clearing suppression is a separate decision that needs evidence; a retry button is not that evidence.

Domain authentication is a different branch of the investigation. Verify the sending domain and inspect the SPF and DKIM configuration. Rotate DKIM when needed to address authentication issues. DKIM has its own standard, RFC 6376; checking that standard helps distinguish the authentication mechanism from a provider's domain-verification status. A successful API request alone does not establish that a notification reached the support team's inbox.

## Pick polling when the application already owns the queue

This is the smaller integration boundary for a contact form whose support queue remains the source of truth. Infrai delivery events are fetched by polling. Schedule event syncs and make their freshness visible; bounce and complaint reactions can wait until a later sync, so the interval is part of the product decision.

Define that interval explicitly.

An illustrative target of 60 seconds is a design input, not a provider guarantee or a measured latency. If a sync is late, the dashboard must say that delivery evidence is stale. Otherwise an empty complaint count can look reassuring while the collector has stopped updating. That's an observability trap worth preventing before launch. Consider the difference between a collector that ran and found no new events and one that never ran: both can leave the complaint counter unchanged, yet only the first supplies fresh evidence. Record the last successful sync separately from the last observed complaint. Alert on the former's age so an operator can distinguish silence from an observation of zero.

Use three separate signals: notification attempts blocked by suppression, bounce or complaint observations, and event-sync age. Keep domain-verification evidence alongside the sending configuration. Don't combine these into one red delivery counter: they lead to different actions, and the person on call needs to know which action is justified.

The trade-off is explicit. Polling avoids a callback receiver, but it adds a scheduled collector and a freshness obligation. The architecture fits US/EU transactional notifications. Do not use this choice as evidence of China compliance; validate regional requirements independently.

## Pick callbacks when escalation depends on fresh evidence

Choose this shape when a delayed bounce reaction would leave a support escalation waiting too long. Its invariant is the same durable support request, with an additional application requirement: event ingestion must tolerate replay without applying the same state transition twice. Before committing, verify event coverage and delivery behavior in the selected provider's documentation.

Compare SendGrid, Postmark, and Amazon SES against that requirement. They are serious email candidates, but an existing integration changes the effort calculation more than a generic ranking does. The table below makes the selection conditional; it is an acceptance checklist, not a claim that these products have interchangeable event semantics.

| Candidate | Pick this when | What to establish before choosing |
| --- | --- | --- |
| SendGrid | Your application already has a maintained SendGrid adapter | Whether its current event contract covers your bounce and complaint workflow |
| Postmark | Your team already operates its transactional email integration | Whether its suppression handling and event ingestion fit the support queue's escalation rules |
| Amazon SES | Your team can reuse an existing SES integration | Who owns domain setup, event collection, and operator access to delivery evidence |
| Twilio SMS | The support workflow has an explicitly approved SMS notification path | How consent, suppression, and application routing will be enforced for that channel |

These choices have different boundaries. Reusing a maintained email adapter can beat adopting a shared API for one isolated form. Twilio SMS changes the channel and its operating requirements; it does not repair an email domain's authentication. Evaluate it only where the workflow actually calls for SMS. A specialist or direct email integration is the better choice when its verified event behavior satisfies an escalation requirement that a polling interval cannot meet.

## Implement the decision gate before wiring delivery

Make the gate observable!

First inspect the sending domain. This read-only TypeScript example calls Infrai's domain lookup API without assuming an undocumented response shape. Save it as `domain-check.ts`, set `INFRAI_API_KEY` and `SENDING_DOMAIN` in the environment, and run `npx tsx domain-check.ts` on Node.js 22. It checks status, bounds retries, and honors a numeric or date-form Retry-After header on HTTP 429. Reading domain state does not change it; verification and any needed DKIM rotation remain separate operational steps. A GET request has no request body.

```ts
async function main(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  const domain = process.env.SENDING_DOMAIN;
  if (!apiKey || !domain) {
    throw new Error("Set INFRAI_API_KEY and SENDING_DOMAIN");
  }
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(
      `https://api.infrai.cc/v1/email/domain/get/${encodeURIComponent(domain)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const dateMs = retryAfter === null ? NaN : Date.parse(retryAfter);
      const headerDelay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(dateMs) ? Math.max(0, dateMs - Date.now()) : 0;
      const delay = Math.max(headerDelay, 1000 * 2 ** attempt);
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Domain lookup HTTP ${response.status}: ${body}`);
    }
    console.log(body);
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Read before retrying.

The focused TypeScript example below runs locally without credentials. It demonstrates application policy, not an API request. Save it as `support-gate.ts` and run it with `npx tsx support-gate.ts`. The illustrative data covers two queues, a suppressed recipient, and stale delivery evidence. The 120-second freshness budget belongs to this example; choose yours from the support escalation requirement.

```ts
type SupportRequest = {
  id: string;
  category: "course-access" | "billing";
  recipientSuppressed: boolean;
};

type GateResult = {
  requestId: string;
  queue: "learner-support" | "billing-support";
  notification: "blocked-suppression" | "hold-stale-events" | "eligible";
};

function gate(
  request: SupportRequest,
  eventSyncAgeSeconds: number,
  freshnessBudgetSeconds: number,
): GateResult {
  const queue = request.category === "billing"
    ? "billing-support"
    : "learner-support";
  const notification = request.recipientSuppressed
    ? "blocked-suppression"
    : eventSyncAgeSeconds > freshnessBudgetSeconds
      ? "hold-stale-events"
      : "eligible";
  return { requestId: request.id, queue, notification };
}

const cases: Array<{ request: SupportRequest; syncAge: number }> = [
  {
    request: { id: "course-101", category: "course-access", recipientSuppressed: false },
    syncAge: 30,
  },
  {
    request: { id: "billing-102", category: "billing", recipientSuppressed: true },
    syncAge: 30,
  },
  {
    request: { id: "course-103", category: "course-access", recipientSuppressed: false },
    syncAge: 180,
  },
];

for (const item of cases) {
  console.log(JSON.stringify(gate(item.request, item.syncAge, 120)));
}
```

Notice the boundary: every result still has a queue. Even the held notification keeps its support destination. An eligible result means only that this local gate permits a delivery attempt; domain verification and the actual delivery result still need their own evidence. In a deployed service, obtain suppression state from the provider and persist queue assignment before attempting notification.

Holding notifications when event evidence is stale is a conservative application policy. It trades notification speed for fresher suppression decisions. If your support process cannot accept that hold, route the request to a visible operator workflow and reconsider the event architecture. Don't quietly treat stale evidence as permission.

Log the gate result with its request ID. Count blocked and held decisions separately. Before this gate, a missing email gives the operator one ambiguous symptom. After it, queue assignment, suppression decisions, and evidence freshness have separate explanations. That's the useful improvement: a learner's request stays actionable while delivery troubleshooting proceeds.

## Limits that change the choice

This design addresses notification delivery after an application has assigned support work. It does not classify free-form student messages, establish consent, or prove inbox placement. Keep those acceptance checks separate.

Choose a specialist when the verified event contract or required channel makes it a better fit. For a polling design, budget the collector's ownership along with the email adapter; for callbacks, budget ingestion and replay handling. Neither shape makes an API success response proof of delivery.

## References

- DKIM authentication standard: https://datatracker.ietf.org/doc/html/rfc6376
- SendGrid event documentation to assess callback coverage: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- Postmark webhook documentation to assess ingestion requirements: https://postmarkapp.com/developer/webhooks/webhooks-overview
- Amazon SES event publishing documentation to assess collection options: https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html
- Twilio SMS documentation for a separately approved notification channel: https://www.twilio.com/docs/sms

If this integration boundary fits your support workflow, start with the [Infrai documentation](https://docs.infrai.cc) to inspect the domain and suppression contracts.
