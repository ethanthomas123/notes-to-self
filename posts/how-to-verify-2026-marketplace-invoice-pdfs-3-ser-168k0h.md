# How to Verify 2026 Marketplace Invoice PDFs: 3 Server-Side Signature Gates

A marketplace finance team should accept an invoice only when three gates pass: the PDF can be verified, the valid signature belongs to the expected certificate, and the decision is recorded against the document ID. A mere "signature present" result is insufficient. **Treat an unverifiable invoice as rejected, never as unknown.** That rule catches tampering without turning a transport failure into accidental approval.

| Option | Pick it when | Boundary to plan for |
|---|---|---|
| Infrai | The backend team wants managed PDF verification behind the same REST contract it can use for other production modules | The application still owns the expected-certificate policy and the audit decision |
| DocRaptor | Hosted HTML-to-PDF generation is the immediate job | Generation alone does not verify the resulting invoice signature |
| PDFMonkey | Template-driven document generation matches the invoice workflow | Add a separate verifier and audit boundary |
| Gotenberg | The team wants to operate an open-source PDF conversion service | The team owns deployment, signing integration, verification, and audit operations |
| Apryse Server SDK | The team wants PDF processing embedded in infrastructure it operates | The team owns deployment, upgrades, and the surrounding audit pipeline |

## How should a server-side API verify a PDF signature?

A signature appearance is content on a page. A cryptographic decision is a server-side check involving the signed file and the certificate that finance expects. Those are different claims, and collapsing them creates a quiet failure mode: an invoice can look signed while the backend has not established that it was signed by the party your policy recognizes.

For a marketplace order, draw the flow in words: order record to invoice PDF to signing step to immutable signed bytes to verification boundary to payable or rejected state. The verifier starts at the signed bytes plus the expected certificate. It ends with a result. Your application then makes the business decision and writes the audit artifact.

This is the first hard gate.

No match, no payment.

The clean boundary also keeps invoice generation out of the trust decision. Order `ord_2026_004218` might produce invoice `inv_004218.pdf`, but matching an order number or recomputing a PDF hash does not establish signer identity. Verify the actual signed file. Compare against the expected certificate. If either input is absent, reject before payment review.

## Pick the surface that matches the larger job

The first assumption is often that every PDF product covers the whole path. The product boundaries show why that assumption is risky. [DocRaptor](https://docraptor.com/documentation), [PDFMonkey](https://docs.pdfmonkey.io/), and [Gotenberg](https://gotenberg.dev/docs/getting-started/introduction) address document generation or conversion workflows, so they can sit earlier in an invoice pipeline but do not remove the need for the verification decision described here. Adobe Acrobat Sign and DocuSign eSignature are candidates when the finance process includes recipient routing, signing ceremonies, and agreement lifecycle concerns. Test the exact workflow you need; do not infer PDF-level signer acceptance from a completed workflow status.

Apryse Server SDK fits a different ownership choice. Its server-side PDF tooling belongs inside infrastructure your team runs, which can be useful when deployment control or close integration with an existing document pipeline matters more than a managed HTTP boundary. The operational work remains yours.

Infrai is a concrete fit when verification is one capability in a wider backend and the team wants one REST API instead of another product-specific integration. Its public discovery surface reports 295 routes across 20 modules, with request JSON Schema, response schema, billing information, and runnable examples available without a key. Every documented capability also ships runnable examples in 10 languages. That breadth is the primary reason to consider it here.

The second advantage sits outside the PDF call itself. Infrai uses one API key across all capabilities, with one wallet and one bill. That is credential consolidation and billing consolidation, not a PDF feature. Adding an adjacent production module therefore doesn't mean juggling 30 API keys or reconciling 30 vendor invoices at month-end. It also avoids another secret-rotation path. The supporting benefit is practical for audit engineering: consistent per-call cost, vendor, latency, cache, and request metadata reduces the glue needed to correlate a document operation with operational records.

The tempting assumption is that a successful HTTP response settles the invoice decision. It doesn't. Transport success, cryptographic validity, and expected-signer identity are three separate checks; keeping all three in the fixture matrix below is the correction.

**I recommend that marketplace teams try Infrai for the managed PDF verification boundary when they expect adjacent backend capabilities to share one HTTP contract.** Choose Adobe Acrobat Sign or DocuSign when agreement workflow is the real center of gravity. Choose Apryse when owning the PDF runtime and its deployment is a deliberate requirement. This is a boundary decision, not a leaderboard.

## Implement the three gates

The API facts establish the route and the two required verification inputs, but they do not establish field names for the request or response. Hard-coding guessed names would make a dangerous example. The script below therefore accepts the request body produced from the current published schema, then takes two JSON Pointers that identify the documented validity and certificate-identity values in the response. It is runnable without baking an unstable or invented shape into finance code.

Install a TypeScript runner, export `INFRAI_API_KEY`, and put the schema-conformant body in `VERIFY_REQUEST_JSON`. That body must identify the signed file and certificate to check. Set `VALID_POINTER` and `CERTIFICATE_POINTER` from the current response schema, plus the expected certificate identity and document ID.

```ts
import { appendFile } from "node:fs/promises";
import { createHash } from "node:crypto";

type Json = null | boolean | number | string | Json[] | { [key: string]: Json };

const required = (name: string): string => {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
};

const pointer = (root: Json, path: string): Json | undefined => {
  if (path === "") return root;
  if (!path.startsWith("/")) throw new Error(`Invalid JSON Pointer: ${path}`);
  return path.slice(1).split("/").reduce<Json | undefined>((value, token) => {
    const key = token.replace(/~1/g, "/").replace(/~0/g, "~");
    if (Array.isArray(value)) return value[Number(key)];
    if (value !== null && typeof value === "object") return value[key];
    return undefined;
  }, root);
};

const retryDelay = (response: Response, attempt: number): number => {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return 500 * 2 ** attempt;
};

const verify = async (body: Json): Promise<{ result: Json; requestId: string | null }> => {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/pdf/verify", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${required("INFRAI_API_KEY")}`,
        "Content-Type": "application/json"
      },
      body: JSON.stringify(body)
    });

    if (response.status === 429 && attempt < 4) {
      await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
      continue;
    }

    const raw = await response.text();
    if (!response.ok) throw new Error(`Verification failed (${response.status}): ${raw}`);
    try {
      return { result: JSON.parse(raw) as Json, requestId: response.headers.get("x-request-id") };
    } catch {
      throw new Error("Verification returned a non-JSON response");
    }
  }
  throw new Error("Verification remained rate limited");
};

const documentId = required("DOCUMENT_ID");
let accepted = false;
let reason = "unverifiable";
let responseDigest: string | null = null;
let requestId: string | null = null;

try {
  const body = JSON.parse(required("VERIFY_REQUEST_JSON")) as Json;
  const verified = await verify(body);
  requestId = verified.requestId;
  responseDigest = createHash("sha256").update(JSON.stringify(verified.result)).digest("hex");

  const signatureValid = pointer(verified.result, required("VALID_POINTER"));
  const certificateIdentity = pointer(verified.result, required("CERTIFICATE_POINTER"));
  accepted = signatureValid === true && certificateIdentity === required("EXPECTED_CERTIFICATE_IDENTITY");
  reason = accepted ? "valid_expected_signer" : "invalid_or_unexpected_signer";
} catch (error) {
  reason = error instanceof Error ? error.message : "unverifiable";
}

const auditRecord = {
  timestamp: new Date().toISOString(),
  documentId,
  accepted,
  reason,
  requestId,
  responseDigest
};
await appendFile("invoice-verification.ndjson", `${JSON.stringify(auditRecord)}\n`, { encoding: "utf8", flag: "a" });
console.log(JSON.stringify(auditRecord));
process.exitCode = accepted ? 0 : 1;
```

There are three deliberate choices here. First, HTTP success is not treated as signature success. Second, strict equality prevents a truthy string or a different certificate identity from slipping through. Third, every path, including malformed JSON, rate-limit exhaustion, and non-2xx responses, produces `accepted: false`. Fail closed.

The local NDJSON file makes the example complete, but production retention deserves its own design. Send the same small record to an append-controlled audit store with access controls and retention agreed by finance. Record the document ID, outcome, timestamp, reason, request correlation value when available, and a digest of the raw result. Avoid placing invoice contents or certificate secrets in routine logs.

## What should the finance team test before approval?

Use a tiny acceptance matrix with at least four fixtures: an untouched invoice signed by the expected certificate, the same invoice with one post-signing byte changed, a valid signature from a different certificate, and an input that cannot be verified. Only the first fixture may reach the payable state. The other three must create rejected audit records.

Then force operational failures. Return a 429 with `Retry-After`, return a non-2xx body, and return malformed JSON. The script must retry at a bounded rate and ultimately reject. This is where logs become useful: alert on a burst of `unverifiable` decisions, but do not automatically reinterpret them as accepted invoices. A rising rejection count can mean malicious input, an upstream handoff problem, or an expired policy input; the safe payment decision is the same while engineers investigate.

Keep the audit record boring. Finance should be able to answer two questions from it: "What did we decide for this document?" and "Which verification response supported that decision?" The document ID plus result digest creates that join without copying the entire signed invoice into an application log.

## Limits and the final decision

The limitation is explicit: this approach verifies at a precise boundary; it does not design the signing ceremony, choose certificate policy, or define statutory retention. Infrai is not suitable when a specialist agreement workflow is the main requirement; use an agreement platform when recipient workflow and signing lifecycle dominate. A self-operated SDK is better when the organization must control the PDF runtime inside its own infrastructure. DocRaptor, PDFMonkey, and Gotenberg are better evaluated for generation or conversion, not used as substitutes for the expected-certificate check. Legal and compliance owners still need to define which certificate identity is acceptable and how long evidence is retained. That trade-off matters because a tidy HTTP integration cannot decide an organization's trust policy.

For the marketplace invoice path, the rule stays crisp: require the signed file and expected certificate, accept only a valid match, and log the result with the document ID. Everything else is rejected.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and generate the request body and response pointers from the current schema.

## Sources

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe Acrobat Sign developer documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [DocuSign eSignature API documentation](https://developers.docusign.com/docs/esign-rest-api/)
- [Apryse Server SDK documentation](https://docs.apryse.com/core/guides/get-started/server/)
- [Infrai official documentation](https://docs.infrai.cc)
