# Identity Verification Photos: Enforcing Scheduled Deletion Across Every Derived Crop

Short answer: assign one retention deadline when an identity-verification upload is accepted, record every original and smart-cropped derivative under the same asset group, and let an idempotent worker delete due objects before it closes the group. For a fintech flow that needs several aspect ratios, crop at upload when the required variants are known; use on-demand crops only when new dimensions are genuinely unpredictable, because each late derivative must still inherit the original deadline.

The scheduler is a wake-up mechanism, not the retention control. The durable control is the database claim, deletion state, and evidence that no known object remains.

## How should scheduled deletion remove all identity verification photos?

An identity photo rarely stays one file. A capture may produce a normalized original, a square reviewer thumbnail, and a wider case-review image. Image formats also differ in browser support and characteristics, so the inventory must store the actual media type and object key rather than infer either from a filename [1].

The easy mistake is to put `delete_after` only on the upload row. A cleanup job removes that object while the crop table, retry output, or cached rendition survives. Deleting by a guessed prefix is also brittle: keys change, two asset groups can share a human-readable identifier, and a failed crop can be written before its database row is committed.

Use an immutable `asset_group_id` as the ownership boundary. Every write must create or update a manifest entry containing the group, object key, kind, and inherited deadline. The parent group is complete only when each manifest entry is deleted or explicitly recorded as already absent. Keep the evidence record free of image bytes and signed access URLs.

No crop gets a fresh clock.

## A runnable TypeScript worker

The example below models storage and persistence as narrow interfaces, so the retention logic is independent of a cron package, queue, database, or object store. A production adapter should claim rows transactionally and enforce one active lease per group. The worker can then run every few minutes without treating exact firing time as correctness.

```ts
interface StoredObject {
  key: string;
  kind: "original" | "crop";
  deletedAt?: Date;
}

interface AssetGroup {
  id: string;
  deleteAfter: Date;
  objects: StoredObject[];
}

interface Repository {
  claimDue(now: Date, leaseUntil: Date, limit: number): Promise<AssetGroup[]>;
  markObjectDeleted(groupId: string, key: string, at: Date): Promise<void>;
  closeGroup(groupId: string, at: Date): Promise<void>;
  release(groupId: string): Promise<void>;
}

interface ObjectStore {
  delete(key: string): Promise<"deleted" | "absent">;
}

export async function deleteExpiredPhotos(
  repo: Repository,
  store: ObjectStore,
  now = new Date(),
): Promise<void> {
  const leaseUntil = new Date(now.getTime() + 5 * 60_000);
  const groups = await repo.claimDue(now, leaseUntil, 100);

  for (const group of groups) {
    try {
      for (const object of group.objects) {
        if (object.deletedAt) continue;

        // Treat a missing object as success so retries converge.
        await store.delete(object.key);
        await repo.markObjectDeleted(group.id, object.key, now);
      }
      await repo.closeGroup(group.id, now);
    } catch (error) {
      await repo.release(group.id);
      throw error;
    }
  }
}
```

Call `deleteExpiredPhotos` from a single scheduled entry point or a queue consumer. Do not use an in-process `setInterval` as the only trigger: process restarts erase its schedule, while overlapping replicas can duplicate work. Duplicate execution is acceptable here because deletion is idempotent and claims are leased; silent omission is not. Node's timer documentation also notes that timer callbacks are not guaranteed to run at an exact time [2].

There is one subtle failure window. The object store may accept a delete, then the process may stop before `markObjectDeleted` commits. The retry sees the object as absent and records success. That is why `absent` belongs to the success path, while authentication failures, timeouts, and rate limits must leave the manifest entry open for retry.

That race matters.

## Crop at upload or on demand?

For a fixed identity-review UI, upload-time processing is the cleaner retention design. The service writes the original and known aspect ratios, registers all keys, and exposes the group only after the manifest is complete. It costs processing and storage before every rendition is viewed, but deletion has a bounded inventory. This is the trade I would choose when the square and wide crops are contractual parts of the review flow.

The limitation is equally concrete: upload-time cropping is a poor fit when dimensions change frequently or most renditions are never opened. On-demand processing is the better alternative in that case, provided crop creation and expiry use the same group-level lock. Its trade-off is more latency on the first request and a larger race surface during deletion; upload-time processing pays earlier and stores more, but makes the inventory finite before review begins. My rule is to accept that cost for two or three stable review shapes, then move rare exports to the on-demand path.

On-demand processing avoids unused renditions. It also opens a race: a request can try to create a crop while the group is expiring. The crop service must read the group state, reject creation after expiry begins, and insert the derivative with the parent's existing `delete_after` value. Never restart the retention clock from crop creation time.

The cost equation should include storage duration, transformation work, request latency, and cleanup operations. It should not be reduced to a vendor's current unit price. Measure the number of crops actually requested and the number declared at upload; that ratio tells you whether speculative processing is buying useful latency or generating waste.

A hybrid can work when only two sizes are stable: create those during upload and generate uncommon reviewer exports on demand. The invariant does not change. Every derivative belongs to exactly one asset group and can never outlive it.

Keep that invariant boring.

## Test the gaps, not just the happy path

A useful test fixture has one original, three crops, a deadline in the past, and one storage key that is already absent. Run two workers against the same group and assert that one lease wins, all four manifest entries reach a deleted state, and the group closes once. Then inject a failure after the first storage deletion but before its database update; the second run should converge without extending the deadline.

Use the same fixture to test a 5-minute lease and a claim batch of 100 groups. Worker A claims the group and deletes the original. Its database update fails. After the lease expires, worker B claims the row, receives `absent` for the original, records that outcome, and continues through the three crops. This sequence checks the awkward boundary that a green-path unit test skips: storage and the database cannot commit atomically, so the evidence row may briefly lag reality. The retry must close that gap without manufacturing a new retention deadline, and the final record must distinguish successful absence from an unattempted delete.

Also test the boundary conditions: a crop request arriving as expiry starts, a worker stopping midway, a lease expiring, and a newly discovered derivative attached to a closed group. The last case should raise an integrity alert rather than quietly reopening the group. Keep clocks injectable, as in the example, so tests do not wait for wall time.

Operationally, track the oldest overdue open group, due groups claimed, object deletions by outcome, retry age, and groups closed with zero objects. Alert on age, not merely error count: repeated retries can look busy while one photo remains past its deadline. Logs should carry the asset group ID and a one-way representation of the object key, but no biometric image, access token, or downloadable URL.

Before deployment, reconcile a sample of manifests against storage listings, verify that caches and temporary processing locations follow the same lifecycle, and confirm that backups have a separately documented expiration path. After deployment, pause crop creation for an expiring group, exercise the retry window, and inspect closure evidence. The job is finished only when the original, every registered crop, and every known temporary copy are gone.

Nothing else counts as done.

## Further reading

1. MDN, Image file type and format guide: https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
2. Node.js, Timers: https://nodejs.org/api/timers.html
3. Node.js, File system `rm`: https://nodejs.org/api/fs.html#fspromisesrmpath-options
