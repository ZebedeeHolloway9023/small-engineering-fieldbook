# Diagnosing Browser Avatar CORS Authorization Before a Presigned Object PUT Begins

Use direct upload only when the browser origin, request method, and request headers can be declared as a small, stable contract. **Short answer:** a presigned PUT authorizes the storage request, while CORS separately decides whether browser JavaScript may send it; an avatar upload fails at preflight when the bucket's CORS policy does not admit the exact origin, method, or headers the browser proposes.

That distinction determines the architecture. An authenticated application endpoint chooses an opaque object key and returns a short-lived upload intent. The browser then sends the bytes to object storage and reports that the transfer finished. Tenant ownership remains in application data, even though the application never relays the file body. For a solo team, this is attractive because application compute doesn't become a file pipe. The catch is operational: two independent controls must agree, and the browser reports many disagreements as the same vague CORS symptom.

Keep the first version boring.

## What exactly does the browser propose before the upload begins?

A presigned URL and a CORS rule answer different questions. The signature lets the storage service verify that a request was authorized under particular constraints. The browser's same-origin protections cause it to check whether JavaScript from one origin may call another origin. For a cross-origin `PUT`, the browser normally sends an `OPTIONS` preflight containing the intended origin, method, and requested header names. If the response doesn't grant that combination, the browser withholds the real upload.

No avatar bytes moved yet.

This is why changing the signature's lifetime rarely fixes a preflight rejection. Start with the request visible in the browser network panel and compare four values literally: the page's origin, including scheme and port; `PUT`; the header names in `Access-Control-Request-Headers`; and the response's CORS allow values. The hosts `app-us.example.test` and `app-eu.example.test` belong to different origins when used under the same HTTPS scheme. So do a local HTTP origin and its deployed HTTPS counterpart. A US/EU rollout often exposes this boundary because the application acquires a second frontend origin while the upload code stays identical. There is another boundary after preflight: the actual PUT still has to match what was signed. If the signer included a content type, the browser must send that same value; casually adding a header in a request wrapper can also change the proposed preflight header set. Don't debug both layers at once. First prove that `OPTIONS` grants the browser's proposed request. Then inspect the PUT response and compare the signed method, key, expiry, and signed headers. The response must grant the requesting origin rather than merely look plausible in a dashboard. When credentials are part of a cross-origin browser request, wildcard policies have additional restrictions; for a presigned upload, avoid adding cookies or an authorization header unless the design truly requires them. The URL already carries scoped authorization, and a same-origin application endpoint can use the user's normal session when it creates the upload intent. This order is faster than repeatedly minting new URLs and hoping one survives.

| Network observation | Boundary to inspect | Next comparison |
| --- | --- | --- |
| `OPTIONS` appears, but PUT does not | Browser CORS admission | Proposed origin, method, and header names versus allow values |
| `OPTIONS` succeeds, then PUT is rejected | Signed storage request | Method, object key, expiry, content type, and other signed headers |
| PUT succeeds, but no avatar appears | Application completion | Expected-upload record, ownership, validation, and active reference |
| One regional host works and another does not | Deployment target | Frontend origin and storage-policy revision for each region |

## How does a browser direct avatar upload use CORS and presigned PUT safely?

The browser needs a narrow contract, not storage credentials. The example below assumes a same-origin application endpoint returns an upload intent with a presigned URL, an opaque object key, an expiry timestamp, and the exact content type the signer bound. It deliberately sends only one content header. The final confirmation contains metadata, never a public object URL; the application can later authorize a download and issue an expiring link.

```ts
type UploadIntent = {
  uploadUrl: string;
  objectKey: string;
  expiresAt: string;
  contentType: string;
};

type AvatarFile = File & { type: "image/jpeg" | "image/png" };

async function uploadAvatar(file: AvatarFile): Promise<void> {
  const intentResponse = await fetch("/api/avatar-upload-intents", {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({
      contentType: file.type,
      size: file.size,
    }),
  });

  if (!intentResponse.ok) {
    throw new Error(`Upload intent failed: ${intentResponse.status}`);
  }

  const intent = (await intentResponse.json()) as UploadIntent;
  if (intent.contentType !== file.type) {
    throw new Error("Signed content type does not match the selected file");
  }

  const uploadResponse = await fetch(intent.uploadUrl, {
    method: "PUT",
    headers: { "content-type": intent.contentType },
    body: file,
  });

  if (!uploadResponse.ok) {
    throw new Error(`Object upload failed: ${uploadResponse.status}`);
  }

  const confirmResponse = await fetch("/api/avatar-upload-completions", {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({ objectKey: intent.objectKey }),
  });

  if (!confirmResponse.ok) {
    throw new Error(`Upload confirmation failed: ${confirmResponse.status}`);
  }
}
```

Those application paths are illustrative internal routes, not storage API routes. On the server, creation of the intent must derive tenant and user identity from the authenticated session rather than accept either value from the browser. It should generate the object key, constrain content type and size according to application policy, persist an expected-upload record, and return only the capability needed for this one transfer. Confirmation should look up that expected record and verify object metadata before changing the active avatar reference.

One capability. One object.

## Treat upload completion as a governance event

The `File` subtype is compile-time guidance, not security validation. A browser-provided MIME type and filename are user-controlled input. OWASP recommends allowlisting extensions, validating the file type rather than trusting the `Content-Type` header, changing the filename, setting size limits, allowing only authorized users, and storing uploads outside the webroot or on a separate host. An avatar pipeline can apply those controls after upload and before publication. Until validation finishes, treat the object as quarantined and don't make it the user's active image.

This creates a small but important state machine: expected, uploaded, validated, and active, with rejected or expired records cleaned up separately. It also prevents a user from confirming an arbitrary key belonging to another tenant. The design costs a couple of metadata writes, but access control is a poor place to save writes.

## How can a second frontend origin break an otherwise valid upload?

Region is usually an indirect cause. A browser in Paris doesn't receive different CORS semantics just because it is in Europe; the meaningful change is commonly the origin it loaded, the storage endpoint selected, or the policy attached to that endpoint. If `app.example.test` routes uploads to one bucket and `app.eu.example.test` routes them to another, both buckets need the intended origin-method-header contract. A policy deployed to only one target produces the classic report: “US works, EU fails.”

I'm not sure which of those boundaries is wrong in a given system until the network trace and deployed policy are side by side. Your mileage may vary with proxies and storage products, but the diagnostic sequence stays useful. Capture the failed `OPTIONS` request. Record its `Origin`, `Access-Control-Request-Method`, and `Access-Control-Request-Headers`. Record the response status and allow headers. Then identify the exact storage endpoint and configuration revision serving that response. This evidence separates a browser admission failure from a later signature or object-policy rejection.

Do not “fix” the problem by reflecting every origin sent by a caller. Maintain an explicit environment-to-origin map and deploy it with the storage configuration. Preview deployments are awkward here — their hostnames may be unbounded — so either give previews a controlled origin, proxy preview uploads through an application endpoint, or provision and retire exact preview origins. A proxy is less simple for delivery, yet it can be the right trade when origins are numerous, upload volume is low, and centralized enforcement matters more than removing a hop.

The same discipline applies to downloads. Private avatars shouldn't quietly become permanent public objects just because browsers need to display them. Authorize the read through the application and issue an expiring download URL, or serve the bytes through an authenticated application path when strict control outweighs delivery simplicity. Cache behavior must be chosen deliberately: MDN documents that `private` permits storage in a private cache while preventing shared-cache storage, and `no-store` asks caches not to store the response. A personalized response and an immutable public asset don't deserve the same `Cache-Control` policy.

## Shipping and operating the boundary

Before deployment, test preflight and PUT as separate events from every allowed frontend origin, including the actual US and EU production hosts. Use a small valid image, a disallowed type, an oversized file, an expired intent, and a confirmation attempt for a key owned by another tenant. The goal isn't a huge test matrix. It is proof that each boundary fails closed and that a valid upload reaches active state only after server-side verification.

Logging needs correlation without leaking the signed query string. Give each expected upload a random internal ID and log that ID with the tenant, selected region, state transition, object key hash, and policy revision. In client telemetry, classify an absent PUT following a failed `OPTIONS` separately from a PUT that reached storage. Redact the upload URL. A presigned URL is a temporary capability, so dropping its full query into analytics or exception reports widens its audience for no operational gain.

Watch counts and latency for intent creation, preflight failure, completed PUT, validation rejection, abandoned expected uploads, and time to active state. Don't pretend a single “upload failed” counter is enough. A rise in preflight failures after an origin rollout points toward configuration; a rise in validation rejection points toward input or policy; expected records that never progress may indicate cancellation, expiry, or a client navigation. Those interpretations are hypotheses, not proof, so retain the correlation needed to check them.

The final release check is prose-sized because the ownership model matters more than checkbox volume: the application authenticates intent creation, creates tenant-scoped opaque keys, and never accepts ownership from client input; storage admits only the explicit production origins, required method, and required headers; the browser sends exactly the signed request; uploaded objects remain private and inactive until independent validation passes; download authorization is separate from upload authorization; logs omit signed URLs; cleanup removes abandoned and rejected objects; and a regional configuration change is tested against every production origin before traffic moves.

**Choose the proxy path instead** when files must be inspected synchronously before they enter object storage, when exact browser origins can't be bounded, or when one application enforcement point is worth the extra bandwidth and latency. Choose direct upload when origins are controlled, payload volume makes relaying wasteful, and the team can operate the two-layer authorization contract. Neither is universally cleaner. For private B2B avatars, the right answer is the least complicated path that still preserves tenant ownership, quarantine, and expiring reads.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
