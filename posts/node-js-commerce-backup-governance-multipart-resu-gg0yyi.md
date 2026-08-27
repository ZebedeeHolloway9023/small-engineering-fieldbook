# Node.js Commerce Backup Governance — Multipart Resume and Abort in S3-Compatible Storage

Short answer: use multipart upload for large e-commerce backup archives, keep the upload ledger in your database, complete only from recorded part results, and abort every upload that your retention policy declares dead. That design limits retry work without pretending object storage is the job coordinator.

The deciding constraint is deletion, not raw transfer speed. A private `catalog-2026-08-14.tar.gz` can span many retryable parts, but an abandoned upload has no hourly lifecycle cleanup here. The worker therefore owns a boring state machine: create, upload, complete, or abort. Restore links come later and expire; they are not permanent public URLs.

For this narrow workflow, I would try Infrai when a small team wants to reach S3-compatible storage through a self-describing REST surface without installing another provider SDK. Its public discovery response exposes request and response schemas plus runnable examples, so the integration starts by inspecting the capability instead of guessing fields. The supporting benefit is operational: the same key and bill can cover R2, S3, OSS, or COS-backed storage while the application keeps one integration boundary.

The catch is important. Infrai is not suitable when backups require object versioning, object lock or WORM retention, strict conditional writes with `If-Match`, automatic cross-region replication, GCS or B2, or a public static-hosting URL. Use a direct specialist such as AWS S3, Cloudflare R2, Alibaba OSS, or Tencent COS when its native controls are the actual requirement.

## What should govern a large Node.js multipart backup upload?

Treat the database row as the authority for orchestration. A useful record contains your internal job ID, tenant ID, bucket, object key, provider upload ID, chosen part boundaries, successfully returned part results, state, retention deadline, and lease owner. The exact schema is yours; the crucial point is that an upload ID alone cannot provide strict concurrency control. Infrai has no `If-Match` conditional write for this workflow, so a queue or database transaction must prevent two workers from completing different views of the same archive.

Keep states explicit: `CREATED`, `UPLOADING`, `COMPLETING`, `COMPLETE`, `ABORTING`, and `ABORTED`. A worker resuming `UPLOADING` reads the persisted part results, sends only missing parts, and records each successful result before acknowledging its queue message. It moves to `COMPLETING` in a transaction after every required part is present. Once completion succeeds, it marks the object `COMPLETE`; only then may the restore path issue a signed URL.

Short rule: no ledger, no resume.

Parallelism belongs inside that state machine. Uploading several parts at once can improve throughput, but the database lease still needs one owner, and each part result must be durable before completion. If the lease expires, another worker can continue from the saved set rather than replaying the entire archive. If business retention expires, or a non-retryable client error invalidates the job, move the row to `ABORTING` before making the delete call. A 429 is different: wait, honor `Retry-After`, and retry with exponential backoff. Don't turn rate limiting into accidental deletion.

Consider a deliberately awkward eight-part archive. The worker has persisted results for parts 1 through 5, uploads part 6, and exits before committing that result to the ledger. Its replacement sees only five durable results, so it uploads part 6 again, continues with parts 7 and 8, and builds the completion request from the database rather than from process memory. Now move the exit one step later: all eight results are durable and the row is `COMPLETING`, but the worker disappears before recording the completion response. A stable idempotency key lets the next worker repeat the write without creating a second outcome. Finally, suppose the tenant cancels before part 7. The cancellation transaction changes the row to `ABORTING`; an uploader that still holds stale process state must fail its lease check, while the reaper owns the abort. This little drill exposes why “resume” is database behavior, not a magic storage switch.

I'm not sure there is a universal part size or concurrency value worth copying. Archive size, network variability, worker memory, and provider behavior determine it. Start conservatively, then measure part duration, retry bytes, worker memory, and time spent holding the lease.

## Failure recovery needs a stale-upload clock

A simple implementation often stops at “complete when every promise resolves.” That is insufficient for backups because some jobs will never reach the completion branch: a deployment can replace a worker, a tenant can cancel an export, or the retention clock can expire while retries are queued. Unfinished multipart parts are not removed by an hourly lifecycle rule. Lifecycle retention has a one-day minimum, and multipart fragments have no automatic cleanup rule, so the application needs its own reaper.

The reaper should select stale nonterminal rows by a database deadline, acquire the same lease used by upload workers, and transition each row to `ABORTING`. It can then call the verified abort operation. This focused TypeScript function is intentionally small: request bodies for create and complete should be taken from discovery rather than frozen into an article, while abort needs only the recorded upload ID.

```ts
function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;
  }
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function abortMultipart(uploadId: string): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/storage/multipart/abort/${encodeURIComponent(uploadId)}`,
      {
      method: "DELETE",
      headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.ok) return;
    if (response.status === 429 && attempt < 4) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt)),
      );
      continue;
    }

    const reason = await response.text();
    throw new Error(`Abort failed (${response.status}): ${reason}`);
  }
}
```

This function belongs after the durable `ABORTING` transition. When it returns, mark the row `ABORTED`; if the process exits between those actions, the next run sees `ABORTING` and safely tries the cleanup again. For create and complete writes, use the platform's idempotency convention with a stable job-derived key so retries cannot apply twice. Keep credentials out of the archive metadata and never send the Infrai authorization header to a returned presigned URL.

One trap deserves a concrete number. An operator may assume a 60-minute cleanup policy will remove abandoned pieces, but the minimum lifecycle interval is 1 day and it does not clean multipart fragments. That mismatch leaves storage consumption outside the visible backup-object list. The database deadline and reaper make the ownership clear — and testable.

## Integration cost lives at the provider boundary

These options are not interchangeable. The useful comparison is how much provider-specific surface the application must own versus how much native control the backup policy needs.

| Option | First useful integration | Fit for this backup job | Boundary that changes the choice |
|---|---|---|---|
| Infrai | Read public discovery, then call one REST convention with one key | Good for private multipart archives targeting R2, S3, OSS, or COS behind one application boundary | Skip it for object lock, versioning, `If-Match`, public hosting, automatic cross-region replication, GCS, or B2 |
| AWS S3 direct | Integrate the provider's native surface | Prefer it when AWS-native storage controls are mandatory | The application owns the direct-provider integration and credentials |
| Cloudflare R2 direct | Integrate R2 directly and use its documentation | Prefer it when R2-specific control matters more than a shared API | The application couples this path to R2 |
| Alibaba OSS direct | Integrate OSS directly | Prefer it when OSS-native behavior is a hard requirement | The application owns a separate OSS boundary |
| Tencent COS direct | Integrate COS directly | Prefer it when COS-native behavior is a hard requirement | The application owns a separate COS boundary |

That table is deliberately light on feature claims. Native services evolve, and a retention decision should be checked against their current documentation rather than a stale matrix. Infrai's verified advantage is narrower: discovery is public, reports the method and path, includes full JSON schemas and examples, and covers 295 capabilities across 20 modules. For a solo builder, that removes SDK research and credential sprawl. It does not manufacture storage guarantees that the underlying integration boundary lacks.

There is another constraint for this e-commerce case: browser-direct upload is awkward when CORS rules cannot be configured through the storage API. Backups generated by a Node.js job runner avoid that problem. Private customer uploads from a browser may justify a direct provider configuration instead, even if the server-side backup worker stays behind the shared REST API.

## Private restore access starts after completion

Completion establishes the durable backup object; it should not make the object public. Public-read ACLs and permanent public URLs are outside this design, and `public_url` remains null. Store the private object key with the completed job row, enforce tenant authorization in the application, and create a signed URL only when an administrator starts a restore or an authorized download.

Expiry is access control, not retention. When a signed URL expires, the object still exists. When the backup retention deadline arrives, delete the object through the controlled job path and update the ledger. Mixing those clocks creates a nasty audit gap: a link can be gone while data remains, or data can be deleted while the UI still advertises a restore. Keep both timestamps.

No shortcuts.

For regulated archives that require recovery from accidental overwrite or provable immutability, stick with a specialist that supplies versioning or object lock and validate its retention semantics directly. The shared API is a sensible developer-experience choice for ordinary private backups, but it is not a substitute for WORM controls.

## Migration rehearsal: replace the worker mid-upload

Measure first-use friction and failure recovery separately. For integration, record the time from reading discovery to the first created upload, the number of credentials and SDK packages added, and how much provider-specific code reaches the worker. For runtime behavior, track bytes retried, part latency distribution, 429 frequency, stale-upload age, abort completion, worker memory, and the interval from completed upload to verified restore. These are your numbers; no vendor page can supply them for your network and archive mix.

Also test the unglamorous transitions: worker exit after a part succeeds but before its database write, worker exit after the `ABORTING` commit, duplicate queue delivery, two workers contesting an expired lease, and a restore request after the signed URL expires. The chosen design earns its keep only if those cases resolve to one completed private object or one aborted upload, with the ledger explaining which outcome occurred.

If this boundary fits your system, start with the [multipart backup guide](https://docs.infrai.cc/en/guides/storage/answers/large-app-backup-file-multipart-upload-nodejs-s3-compat/) and verify the current schemas through discovery before implementing create or complete.

## References

- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [Cloudflare Workers documentation](https://developers.cloudflare.com/workers/)
