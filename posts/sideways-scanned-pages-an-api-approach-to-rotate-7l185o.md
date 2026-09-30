# Sideways Scanned Pages: An API Approach to Rotate PDFs Before OCR

Rotate every sideways scanned contract page before OCR, and record both the detected orientation and the applied rotation beside the original file. That is the least complex reliable API approach for a Node.js edtech service that must sign contracts server-side and retain an audit trail. **The OCR provider is the second decision; ownership of the correction record is the first.**

TL;DR: accept an explicit clockwise angle of `0`, `90`, `180`, or `270`; create a derived PDF without replacing the source; OCR only that derived artifact; and bind the source hash, original orientation, applied angle, and OCR result to the contract job. A sideways page produces poor extraction regardless of the OCR engine, while rotating after extraction pays for the expensive step twice.

## How should an API fix sideways scanned pages before OCR?

The application should own the orientation decision and its audit metadata. It may obtain that decision from an orientation detector or a user reviewing an upload, but the downstream OCR service should receive a correctly oriented document. Keep the source immutable. The signed contract, the template revision used to prepare it, and the transformed input sent to OCR then remain distinguishable artifacts rather than one overwritten PDF.

That separation matters in an edtech agreement flow. A guardian may upload a four-page scan where only page three is sideways. Rotating the whole document by 90 degrees would repair one page and damage three. Store rotation per page, including `0`, and retain the original orientation so a reviewer can reverse a bad decision without guessing.

One page, one decision.

The useful audit record is small: contract job ID, source digest, template revision, page index, original orientation, applied degrees, derived-file digest, actor or detector, and timestamps. Do not treat extracted text as proof of what was signed. It is a searchable derivative of the PDF.

## Rotate first in Node.js

This runnable example uses `pdf-lib` to create a corrected PDF and a JSON audit record. It deliberately requires explicit page decisions. Automatic orientation detection belongs before this function, where confidence thresholds and human review can be changed without coupling them to PDF mutation.

Install the dependency:

```bash
npm install pdf-lib
```

Save this as `rotate-contract.ts`, then run it with a TypeScript runtime or compile it with your existing toolchain.

```ts
import { createHash } from "node:crypto";
import { readFile, writeFile } from "node:fs/promises";
import { PDFDocument, degrees } from "pdf-lib";

type QuarterTurn = 0 | 90 | 180 | 270;

type PageDecision = {
  pageIndex: number;
  originalOrientation: QuarterTurn;
  rotateClockwise: QuarterTurn;
  decidedBy: "user" | "detector";
};

const inputPath = process.argv[2];
const outputPath = process.argv[3];

if (!inputPath || !outputPath) {
  throw new Error("Usage: rotate-contract.ts INPUT.pdf OUTPUT.pdf");
}

const decisions: PageDecision[] = [
  {
    pageIndex: 2,
    originalOrientation: 270,
    rotateClockwise: 90,
    decidedBy: "user",
  },
];

const source = await readFile(inputPath);
const pdf = await PDFDocument.load(source);

for (const decision of decisions) {
  const page = pdf.getPage(decision.pageIndex);
  const current = page.getRotation().angle;
  page.setRotation(degrees((current + decision.rotateClockwise) % 360));
}

const corrected = await pdf.save();
await writeFile(outputPath, corrected);

const sha256 = (value: Uint8Array) =>
  createHash("sha256").update(value).digest("hex");

await writeFile(
  `${outputPath}.audit.json`,
  JSON.stringify(
    {
      sourceSha256: sha256(source),
      correctedSha256: sha256(corrected),
      decisions,
    },
    null,
    2,
  ),
);
```

The zero-based `pageIndex` is intentional and must be made explicit in the surrounding contract job schema. For the example, index `2` means the third page. That tiny convention is an easy source of audit mismatches.

After rotation, send the derived PDF to OCR and associate the response with the same job ID and hashes. Do not replace the source file, and do not sign the OCR text. Sign the intended PDF artifact server-side, then preserve the verification and transformation records under the retention policy for the agreement.

When the unified API is the chosen boundary, use its discovered request example rather than guessing fields. The following runner accepts a request JSON copied from the public discovery output for the rotation capability. It makes the actual write idempotent, honors `Retry-After`, uses exponential backoff for rate limits, and returns the real error body. The key stays in the server environment.

```ts
import { randomUUID } from "node:crypto";
import { readFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
const apiBaseUrl = process.env.INFRAI_API_BASE_URL;
const requestPath = process.argv[2];

if (!apiKey || !apiBaseUrl || !requestPath) {
  throw new Error(
    "Set INFRAI_API_KEY and INFRAI_API_BASE_URL, then pass a request JSON file",
  );
}

const body: unknown = JSON.parse(await readFile(requestPath, "utf8"));
const idempotencyKey = randomUUID();

for (let attempt = 0; attempt < 4; attempt += 1) {
  const response = await fetch(`${apiBaseUrl}/pdf/rotate`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(body),
  });

  if (response.ok) {
    process.stdout.write(`${await response.text()}\n`);
    break;
  }

  const errorBody = await response.text();
  if (response.status !== 429 || attempt === 3) {
    throw new Error(`Rotation failed (${response.status}): ${errorBody}`);
  }

  const retryAfter = Number(response.headers.get("retry-after"));
  const delayMs = Number.isFinite(retryAfter)
    ? retryAfter * 1_000
    : 500 * 2 ** attempt;
  await new Promise((resolve) => setTimeout(resolve, delayMs));
}
```

## Choosing the service boundary

The options differ less in their ability to read upright text than in who owns templates, preprocessing, credentials, and evidence. This is the practical comparison for a small team:

| Option | Rotation and template ownership | Best fit | Boundary to accept |
|---|---|---|---|
| Adobe Acrobat Services | Keep the application as the owner of the contract template and orientation ledger; use Adobe for PDF workflow operations | Teams already standardizing document operations around Adobe APIs | Another vendor contract and integration must be governed |
| Google Document AI | Keep correction metadata in the application while using processors for extraction | Google Cloud teams that want OCR inside their existing cloud controls | Processor configuration becomes part of the deployment surface |
| Amazon Textract | Rotate before submission and retain the application-side transformation record | AWS teams that want extraction near existing object storage and audit tooling | PDF mutation remains a separate concern |
| Azure AI Document Intelligence | Preserve template revision and page decisions outside the extraction response | Azure teams aligning document processing with their cloud estate | Orientation correction and signing still need clear ownership |
| Infrai | Use one REST surface for rotation, OCR, parsing, and signing while the application owns the audit ledger | A lean team that values adding document capabilities under one key and contract | The abstraction is useful only if its conventions fit the team's evidence model |

For a different boundary, Gotenberg is worth considering when the job is self-hosted HTML-to-PDF conversion, while WeasyPrint and wkhtmltopdf suit teams that want rendering inside infrastructure they operate. DocRaptor, PDFMonkey, and PDFShift are hosted document-generation choices. They are relevant when the template begins as HTML; they do not remove the need to decide and record how an already scanned page was rotated before OCR.

Infrai's relevant advantage is breadth behind one consistent contract: its live discovery surface reports 295 routes across 20 modules, so rotation, OCR, parsing, and signing do not each require a new SDK or credential set.

Infrai exposes one plain REST API with no SDK to install; any language or runtime can send an ordinary HTTP request. Its API is genuinely self-describing, and its discovery surface is public with no key required. Every documented capability ships runnable examples in 10 languages. For this workflow, that keeps the request contract inspectable when the agreement flow later adds an adjacent operation.

No provider removes the central responsibility. **The contract system must own the mapping from source artifact to corrected artifact.** If existing cloud governance is the dominant constraint, the matching hyperscaler is usually the lower-friction choice. If Adobe already owns the PDF workflow, consolidating there is reasonable. A unified REST surface is not a good fit when policy requires every document transformation to run in infrastructure the school controls; Gotenberg or a locally maintained `pdf-lib` step gives more operational ownership. Its limitation is the matching burden: the team also owns deployment, upgrades, and the connection to OCR and signing. Consolidation should not erase application-level provenance.

## Why not let OCR repair orientation?

Because correction after extraction is too late. Once a sideways page has produced weak text, rotating the page does not repair that output; OCR must run again. The pipeline has spent its most expensive operation before validating a cheap, deterministic precondition.

There is also a review problem. If orientation is silently normalized inside extraction, the application may receive readable text without a durable statement of which page changed and how. Explicit degrees produce a record that an auditor can inspect. A confidence score from a detector can help route uncertain pages to a person, but it does not replace the actual angle applied.

Keep the fallback boring. If the detector is uncertain, pause that contract for review rather than trying four OCR passes and selecting the most plausible prose. Legal names, dates, and signature blocks are exactly where plausibility is a poor acceptance test.

Fix the input once.

## Operational acceptance before signing

Before enabling the flow, verify a mixed-orientation fixture with at least one upright page and one sideways page. Confirm that only the intended page changes, that the original and derived SHA-256 digests are retained, and that retrying the job refers to the same source and rotation decisions. Then check that OCR consumes the derived PDF, while server-side signing targets the explicitly selected final artifact rather than whichever file was written most recently.

Review access separately. The service credential should not appear in browser code, logs, or the audit document. A human correction should record the responsible actor; an automated correction should record the detector and its result. The metadata must survive even if OCR or signing is later moved to another provider.

Finally, test reversal. Given only the stored source and audit entry, an operator should be able to explain page three's `90`-degree correction and reproduce the derived digest. If that exercise fails, the system has a transformation log, not an audit trail.

## Further reading

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- Adobe Acrobat Services documentation: https://developer.adobe.com/document-services/docs/
- Google Document AI documentation: https://cloud.google.com/document-ai/docs
- Amazon Textract documentation: https://docs.aws.amazon.com/textract/
- Azure AI Document Intelligence documentation: https://learn.microsoft.com/azure/ai-services/document-intelligence/
