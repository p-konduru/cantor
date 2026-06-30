---
feature: cantor-multicloudj
phase: tasks
depends_on:
  - requirements.md
  - design.md
skill: spec-driven-development
---

# Implementation Plan: cantor-multicloudj

> Every task uses `- [ ]` checkbox syntax (required by the autonomous runner). Tasks are sequenced so each builds on the previous. Tests precede implementation wherever a task produces testable logic; scaffolding-only tasks are tagged `no-test`.
>
> Working directory for all paths below is the repository root (`/Users/p.konduru/cantor-os`). All new source lives under `cantor-multicloudj/` except for one line added to the root `pom.xml`.

---

- [x] 1. Scaffold the `cantor-multicloudj` Maven module (`no-test`: pure build config)
  - Create directory `cantor-multicloudj/` at the repo root.
  - Create `cantor-multicloudj/pom.xml` exactly as specified in `design.md` §Architecture/Module pom.xml (artifactId `cantor-multicloudj`, parent `cantor-parent` 0.5.25-SNAPSHOT with `relativePath=../pom.xml`, `<multicloudj.version>0.4.0</multicloudj.version>`, Java 11 override via `<source.version>/<target.version>` properties AND a re-declared `maven-compiler-plugin` block, `<description>` matching Req 7.3).
  - Declare compile deps: `cantor-common` (managed version), `com.salesforce.multicloudj:blob-client`.
  - Declare optional compile deps: `blob-aws`, `blob-gcp`, `blob-ali` (each with `<optional>true</optional>`).
  - Declare runtime deps: `logback-classic`, `gson` (versioned via inherited `${gson.version}`), `guava`.
  - Declare test deps: `testng`, `cantor-common` `test-jar` scope=test, `blob-inmemory` scope=test.
  - Create empty `cantor-multicloudj/src/main/java/com/salesforce/cantor/multicloudj/` and `cantor-multicloudj/src/test/java/com/salesforce/cantor/multicloudj/` directories.
  - Create `cantor-multicloudj/src/test/resources/logback-test.xml` (copy from `cantor-s3/src/test/resources/logback-test.xml`).
  - Verify: `mvn -pl cantor-multicloudj -am compile` succeeds (no Java sources yet, so this validates the pom only).
  - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 2.1, 2.2, 2.3, 2.4, 2.5, 7.3_
  - _Commit: `feat(cantor-multicloudj): scaffold maven module with multicloudj dependencies`_

- [x] 2. Register the new module in the root `pom.xml` (`no-test`: pure build config)
  - Edit `pom.xml` at the repo root: insert `<module>cantor-multicloudj</module>` on the line immediately after `<module>cantor-s3</module>` (currently line 117) and before `<module>cantor-misc</module>`.
  - Do not modify any other file in any other module.
  - Verify: `mvn -pl cantor-multicloudj clean install` builds successfully and produces `cantor-multicloudj/target/cantor-multicloudj-0.5.25-SNAPSHOT.jar`. Also verify `mvn -pl cantor-s3 test` still passes (Req 10.2).
  - _Requirements: 1.1, 10.1, 10.2_
  - _Commit: `build(cantor-multicloudj): register module in parent pom`_

- [x] 3. Add the `StreamingObjects` interface (`no-test`: pure interface, no logic)
  - Copy `cantor-s3/src/main/java/com/salesforce/cantor/s3/StreamingObjects.java` to `cantor-multicloudj/src/main/java/com/salesforce/cantor/multicloudj/StreamingObjects.java`.
  - Change the `package` declaration to `com.salesforce.cantor.multicloudj`.
  - Keep the contract identical: extends `com.salesforce.cantor.Objects`; adds `void store(String ns, String key, InputStream stream, long length)` and `InputStream stream(String ns, String key)`.
  - Verify: `mvn -pl cantor-multicloudj compile` passes.
  - _Requirements: 4.1, 10.1_
  - _Commit: `feat(cantor-multicloudj): add StreamingObjects interface`_

- [x] 4. Add the in-memory `BucketClient` test factory (`no-test`: pure test scaffolding)
  - Create `cantor-multicloudj/src/test/java/com/salesforce/cantor/multicloudj/support/InMemoryBucketClients.java` (package `com.salesforce.cantor.multicloudj.support`).
  - Expose a single static factory: `public static BucketClient fresh()` returning `BucketClient.builder("inmemory").withBucket("test-" + java.util.UUID.randomUUID()).build()`.
  - This ensures every test gets a fresh, isolated in-memory bucket (per design §Testing Strategy).
  - Verify: `mvn -pl cantor-multicloudj test-compile` passes.
  - _Requirements: 8.1_
  - _Commit: `test(cantor-multicloudj): add in-memory BucketClient test factory`_

- [x] 5. Implement `MulticloudjUtils` (static helpers) — test-first
  - [x] 5.1 Write `MulticloudjUtilsTest`
    - Create `cantor-multicloudj/src/test/java/com/salesforce/cantor/multicloudj/MulticloudjUtilsTest.java` (TestNG).
    - Cover `matchesMetadata(metadata, query)` with a `@DataProvider` exercising operators `=`, `!=`, `~` (LIKE with `*`), `!~` (NOT LIKE) — at least 8 cases including empty query, single-key match, single-key miss, mixed-AND across two keys.
    - Cover `matchesDimensions(dimensions, query)` with operators `=`, `!=`, `>`, `>=`, `<`, `<=`, `..` (BETWEEN inclusive) — at least 8 cases.
    - Add `listKeysOrdered_pagination_returnsStableOrder`: upload 1000 keys via `InMemoryBucketClients.fresh()`, page through with `(start=0, count=100)`, `(start=100, count=100)`, …, assert no duplicates, no gaps, and total = 1000. This explicitly guards against the C1 regression (`S3Utils.getKeys` double-advance + HashSet ordering).
    - Add `listKeysOrdered_countMinusOne_returnsAll`: `count = -1` returns every key.
    - Add `getMatchingKeys_minimalPrefixCover`: for windows that span an hour boundary, assert the set of prefixes covers all in-window minutes and no out-of-window minutes (per `EventsOnS3.getMatchingKeys` algorithm).
    - Add `getMatchingKeys_rejectsStartAfterEnd`: assert `IllegalArgumentException` (fix for M5).
    - Add `deleteBatched_chunksAtBatchSize`: insert 250 keys, call `deleteAllUnderPrefix`, verify all gone; use a mock or counting decorator if needed to assert chunking into ≤100-key batches.
    - Add `wrapAsIOException_includesOpAndNamespace`: assert the wrapped message contains both the op name and the namespace.
    - Tests must fail to compile until 5.2 lands.
    - _Requirements: 6.2, 4.6, 4.9, 5.4, 5.5, 5.6, 5.7, 8.1, 8.5_
    - _Commit: `test(cantor-multicloudj): add MulticloudjUtils tests covering filters and pagination`_
  - [x] 5.2 Implement `MulticloudjUtils`
    - Create `cantor-multicloudj/src/main/java/com/salesforce/cantor/multicloudj/MulticloudjUtils.java` (package-private final class with private constructor).
    - Methods (all per design §MulticloudjUtils): `doesObjectExist`, `upload(byte[])`, `upload(InputStream)`, `uploadFile(File)`, `download`, `downloadRange`, `listKeys`, `listKeysOrdered(prefix, start, count)`, `countKeys`, `deleteAllUnderPrefix`, `deleteBatched(Collection<String>)`, `wrapAsIOException(op, ns, cause)`, `getMatchingKeys(prefix, startMs, endMs)`, `matchesMetadata`, `matchesDimensions`.
    - `upload` uses `UploadRequest.builder().withKey(k).build()` (builder-only; there is no `new UploadRequest(key)`).
    - `download` uses `bucketClient.download(DownloadRequest.builder().withKey(k).build(), baos)` and returns `baos.toByteArray()`.
    - `downloadRange` uses `DownloadRequest.builder().withKey(k).withRange(startInclusive, endInclusive).build()`; on `SubstrateSdkException` whose message contains `"range"` (case-insensitive) or `UnsupportedOperationException`, fall back to full download + in-memory slice and log a `WARN` once per JVM per provider (use an `AtomicBoolean` flag).
    - `listKeys` / `listKeysOrdered` iterate the `Iterator<BlobInfo>` from `bucketClient.list(...)` directly — no manual `start`/`count` pagination on the SDK call; the slicing is done in-process.
    - `deleteBatched` chunks into batches of 100 (`EventsOnMulticloudj.DELETE_BATCH_MAX`) and calls `bucketClient.delete(Collection<BlobIdentifier>)`; each `BlobIdentifier` constructed as `new BlobIdentifier(key, null)`.
    - `deleteAllUnderPrefix` iterates `listKeys(prefix)` and delegates to `deleteBatched`.
    - `getMatchingKeys(prefix, startMs, endMs)`: precondition `startMs <= endMs` (else throw `IllegalArgumentException`). Expand `[startMs, endMs]` into the minimal set of `yyyy/MM/dd/HH/` and `yyyy/MM/dd/HH/mm` prefixes using UTC, matching `EventsOnS3.getMatchingKeys`'s algorithm.
    - `matchesMetadata` / `matchesDimensions` are pure predicates over `Map<String,String>` / `Map<String,Double>`; operators are parsed from the query value's leading token: `=` (no prefix), `!=`, `~`, `!~`, `>`, `>=`, `<`, `<=`, `..` (BETWEEN). For LIKE patterns, translate trailing `*` to suffix-match, leading `*` to prefix-match, both to substring-match.
    - `wrapAsIOException(op, ns, cause)` returns `new IOException(String.format("exception during %s on namespace '%s'", op, ns), cause)`.
    - All catches of `SubstrateSdkException` rethrow via `wrapAsIOException`; nothing swallows or returns null on error.
    - Tests from 5.1 must pass.
    - Verify: `mvn -pl cantor-multicloudj test -Dtest=MulticloudjUtilsTest`.
    - _Requirements: 4.10, 6.2, 6.3_
    - _Commit: `feat(cantor-multicloudj): implement MulticloudjUtils helpers`_
    - _Implementation note for downstream tasks: `InMemoryBucketClients.fresh()` uses reflection to seed `InMemoryBlobStore.BUCKETS` because `BucketClient` doesn't expose `createBucket` and going through `BlobClient.builder("memory")` fails when the optional blob-aws/blob-gcp/blob-ali providers are on the classpath (their `AbstractBlobClient` SPI entries lack public no-arg ctors). The provider id is `"memory"` (not `"inmemory"`). Use `JAVA_HOME=/Users/p.konduru/Applications/IntelliJ\ IDEA.app/Contents/jbr/Contents/Home` and `mvn` at `/Users/p.konduru/Applications/IntelliJ\ IDEA.app/Contents/plugins/maven/lib/maven3/bin/mvn` to run tests. Pass `-Dgpg.skip=true` to bypass the parent-pom GPG plugin._

- [x] 6. Implement `AbstractBaseMulticloudjNamespaceable` — test-first
  - [x] 6.1 Write `AbstractBaseMulticloudjNamespaceableTest`
    - Create `cantor-multicloudj/src/test/java/com/salesforce/cantor/multicloudj/AbstractBaseMulticloudjNamespaceableTest.java`.
    - Define a `Probe` inner class extending `AbstractBaseMulticloudjNamespaceable` with `getObjectKeyPrefix(ns) = "cantor-test/" + trim(ns)`.
    - `testConstructor_validatesBucketReachable`: assert constructor succeeds against `InMemoryBucketClients.fresh()`.
    - `testCreate_writesNamespaceMarkerAtExpectedKey`: after `create("foo")`, assert `doesObjectExist("cantor-test/foo-<hash>/.namespace", null)` is true.
    - `testCreate_isIdempotent`: calling `create` twice is a no-op (no exception, marker not duplicated).
    - `testDrop_removesAllKeysUnderPrefix`: stage 5 extra blobs under the namespace prefix, call `drop`, assert all 5 gone AND the marker gone.
    - `testTrim_lowercasesStripsAndAppendsHash`: assert `trim("My.NS/123")` equals the exact `cantor-s3` output for the same input (byte-for-byte parity per Req 4.11).
    - `testTrim_truncatesAt64Chars`: input longer than 64 chars yields a name whose pre-hash portion is exactly 64.
    - Tests must fail to compile until 6.2 lands.
    - _Requirements: 4.11, 6.1, 8.1_
    - _Commit: `test(cantor-multicloudj): add AbstractBaseMulticloudjNamespaceable tests`_
  - [x] 6.2 Implement `AbstractBaseMulticloudjNamespaceable`
    - Create `cantor-multicloudj/src/main/java/com/salesforce/cantor/multicloudj/AbstractBaseMulticloudjNamespaceable.java` (public abstract class implementing `Namespaceable`).
    - Fields: `protected static final String NAMESPACE_IDENTIFIER = ".namespace"`, `protected final BucketClient bucketClient`, `protected final String bucketName`. Read `bucketName` from `bucketClient.getBucket()` so callers cannot mis-pair the two.
    - Constructor `protected AbstractBaseMulticloudjNamespaceable(BucketClient bucketClient, String type)`: validate `bucketClient != null` via `CommonPreconditions.checkArgument`; call `bucketClient.doesBucketExist()` and on `false`/`SubstrateSdkException` throw `IOException("bucket '<name>' is not reachable", cause)`.
    - `create(namespace)`: `CommonPreconditions.checkCreate(namespace)`; compute marker key `getObjectKeyPrefix(namespace) + "/" + NAMESPACE_IDENTIFIER`; if `MulticloudjUtils.doesObjectExist(...)` returns true, log and return; else upload the bytes of `("namespace=" + namespace).getBytes(UTF_8)` via `MulticloudjUtils.upload`. Wrap `SubstrateSdkException` via `MulticloudjUtils.wrapAsIOException`.
    - `drop(namespace)`: `CommonPreconditions.checkDrop(namespace)`; `MulticloudjUtils.deleteAllUnderPrefix(bucketClient, getObjectKeyPrefix(namespace))`; wrap exceptions.
    - `protected static String trim(String namespace)`: identical to `AbstractBaseS3Namespaceable.trim` — lowercase, strip `[^A-Za-z0-9_\-/]`, truncate to 64 chars, append `-<abs(namespace.hashCode())>`. Copy the implementation verbatim from `cantor-s3` so the hash is bit-identical (do not modify `cantor-s3` to share code; Req 10.1).
    - `abstract protected String getObjectKeyPrefix(String namespace)`.
    - Tests from 6.1 must pass.
    - _Requirements: 4.1, 4.8, 4.11, 6.1_
    - _Commit: `feat(cantor-multicloudj): implement AbstractBaseMulticloudjNamespaceable`_

- [x] 7. Implement `ObjectsOnMulticloudj` — test-first
  - [x] 7.1 Write `ObjectsOnMulticloudjTest`
    - Create `cantor-multicloudj/src/test/java/com/salesforce/cantor/multicloudj/ObjectsOnMulticloudjTest.java`.
    - Mirror the `cantor-h2` pattern: `public class ObjectsOnMulticloudjTest extends com.salesforce.cantor.common.AbstractBaseObjectsTest` overriding only `protected Cantor getCantor() throws IOException { return new CantorOnMulticloudj(InMemoryBucketClients.fresh()); }`. The class must NOT be `@Test(enabled = false)` — D9 explicitly rejects the cantor-s3 commented-out-tests anti-pattern.
    - Optionally override `getStoreMagnitude()` to `1.0` (default); leave at default unless the in-memory provider proves too slow.
    - Add focused unit tests in the same package for failure paths the abstract base does not cover:
      - `testStore_propagatesSubstrateException`: use a `BucketClient` decorator that throws `SubstrateSdkException` on upload; assert `store` throws `IOException` (fix vs. H1 `ObjectsOnS3.store(stream,length)` which silently swallows).
      - `testStream_propagatesSubstrateException`: same for `stream(...)`; assert `IOException` (fix vs. H1 `ObjectsOnS3.stream` returning `null`).
      - `testStreamStore_propagatesSubstrateException`: same for `store(stream, length)`.
      - `testDelete_returnsFalseWhenAbsent`: assert `delete(ns, "missing-key")` returns `false`.
      - `testKeys_filtersOutNamespaceMarker`: after `create(ns)`, assert `keys(ns, "", 0, -1)` does not include `.namespace`.
    - Tests must fail to compile until 7.2 lands.
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 4.8, 4.10, 4.11, 8.2, 8.5_
    - _Commit: `test(cantor-multicloudj): add ObjectsOnMulticloudj tests`_
  - [x] 7.2 Implement `ObjectsOnMulticloudj`
    - Create `cantor-multicloudj/src/main/java/com/salesforce/cantor/multicloudj/ObjectsOnMulticloudj.java` (public final class extending `AbstractBaseMulticloudjNamespaceable` implementing `StreamingObjects`).
    - Constant `private static final String OBJECT_KEY_PREFIX = "cantor-objects"`.
    - `protected String getObjectKeyPrefix(String ns)` returns `OBJECT_KEY_PREFIX + "/" + trim(ns)`.
    - Helper `private String getObjectKey(String ns, String key)` returns `getObjectKeyPrefix(ns) + "/" + key`.
    - Every public method begins with the matching `ObjectsPreconditions` check (`checkStore`, `checkGet`, `checkDelete`, `checkKeys`, `checkSize`), then catches `SubstrateSdkException` and rewraps via `MulticloudjUtils.wrapAsIOException("<op>", namespace, e)`.
    - `store(ns, key, bytes)`: `MulticloudjUtils.upload(bucketClient, getObjectKey(ns, key), bytes)`.
    - `store(ns, key, stream, length)`: `MulticloudjUtils.upload(bucketClient, getObjectKey(ns, key), stream)` — rethrows on failure (fix H1).
    - `get(ns, key)`: if `MulticloudjUtils.doesObjectExist(...)` is false, return `null`; else `MulticloudjUtils.download(...)`.
    - `stream(ns, key)`: returns `new ByteArrayInputStream(get(ns, key))` when present; throws `IOException` on download failure (fix H1; document the memory trade-off in Javadoc).
    - `delete(ns, key)`: `existed = doesObjectExist(...); bucketClient.delete(getObjectKey(ns, key), null); return existed;` (MultiCloudJ `delete` returns `void`).
    - `keys(ns, start, count)` delegates to `keys(ns, "", start, count)`.
    - `keys(ns, prefix, start, count)`: list under `getObjectKeyPrefix(ns) + "/" + prefix` via `MulticloudjUtils.listKeysOrdered`, drop the `.namespace` marker, strip the `getObjectKeyPrefix(ns) + "/"` prefix from each returned key.
    - `size(ns)`: `MulticloudjUtils.countKeys(bucketClient, getObjectKeyPrefix(ns))` minus 1 when the marker is present.
    - Inherited `create`/`drop` come from `AbstractBaseMulticloudjNamespaceable`.
    - Tests from 7.1 must pass.
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 4.8, 4.9, 4.10, 4.11_
    - _Commit: `feat(cantor-multicloudj): implement ObjectsOnMulticloudj`_

- [x] 8. Implement `EventsOnMulticloudj` — test-first
  - [x] 8.1 Write `EventsOnMulticloudjTest`
    - Create `cantor-multicloudj/src/test/java/com/salesforce/cantor/multicloudj/EventsOnMulticloudjTest.java` extending `com.salesforce.cantor.common.AbstractBaseEventsTest`. Override `getCantor()` to return `new CantorOnMulticloudj(InMemoryBucketClients.fresh())` configured with `flushIntervalSeconds=1` and a per-test buffer directory (use TestNG `@BeforeMethod` with a `Files.createTempDirectory(...)` and `@AfterMethod` cleanup).
    - In each test's flow, after `store(...)` and before `get(...)`/`metadata(...)`/`dimension(...)`/`expire(...)`, invoke the package-private `EventsOnMulticloudj.forceFlushNow()` (test hook) so reads see what was written deterministically.
    - Add focused tests:
      - `testStore_propagatesSubstrateException`: assert `store` throws `IOException` when the underlying upload fails (fix vs. cantor-s3 silent-swallow pattern).
      - `testExpire_doesNotScanFromEpoch`: stage one event at `now-1d`, call `expire(ns, now-1h)` with a counting `BucketClient` decorator; assert the helper made `O(prefix-pages)` `list` calls, not `O(minutes-since-epoch)` (fix C2).
      - `testGetMatchingKeys_rejectsStartAfterEnd`: assert `get(ns, end=10, start=20, ...)` throws `IllegalArgumentException` (fix M5; via the underlying util).
      - `testIncludePayloads_usesRangeDownload`: stage an event with a payload, retrieve with `includePayloads=true`, assert the payload round-trips byte-for-byte; on the in-memory provider, if range download is unsupported, the fallback slice path must still produce the correct bytes.
      - `testTwoInstances_dontShareState`: construct two `EventsOnMulticloudj` against the same bucket with the same buffer dir root; assert each flushes its own subtree and does not delete the other's data (fix C3, H4).
      - `testStore_concurrentWritesDifferentNamespaces`: two threads store to different namespaces concurrently; assert no data is lost (per-namespace lock allows parallelism across namespaces).
    - Tests must fail to compile until 8.2 lands.
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 5.8, 5.9, 5.10, 8.3, 8.5, 8.6_
    - _Commit: `test(cantor-multicloudj): add EventsOnMulticloudj tests`_
  - [x] 8.2 Implement `EventsOnMulticloudj`
    - Create `cantor-multicloudj/src/main/java/com/salesforce/cantor/multicloudj/EventsOnMulticloudj.java` (public final class extending `AbstractBaseMulticloudjNamespaceable` implementing `com.salesforce.cantor.Events` and `AutoCloseable`).
    - Constants per design: `OBJECT_KEY_PREFIX = "cantor-events"`, `DELETE_BATCH_MAX = 100`, `DEFAULT_FLUSH_SECONDS = 60`, default buffer dir `cantor-events-multicloudj-buffer`.
    - Fields (all instance-scoped — **no statics**, fixes C3 & H3): `instanceId` (UUID per construction), `bufferRoot` = `<bufferDir>/<instanceId>/` (created lazily), `flushExecutor` (single-thread `ScheduledExecutorService`), `uploadPool` (32-thread fixed `ExecutorService` with named threads), `namespaceLocks` (`ConcurrentHashMap<String, ReentrantLock>`), `payloadOffsets` (Guava `LoadingCache<String, AtomicLong>`), `currentFlushCycle` (`AtomicReference<String>` set to `<formattedNow>.<UUID>`), `parser` (`Gson`).
    - Three constructors: `(BucketClient)`, `(BucketClient, String bufferDirectory)`, `(BucketClient, String bufferDirectory, long flushIntervalSeconds)` — all delegate to the most-specific. Validate inputs via `CommonPreconditions.checkArgument`. Schedule `flush()` via `flushExecutor.scheduleWithFixedDelay(flushTask, flushIntervalSeconds, flushIntervalSeconds, SECONDS)` — first run is delayed, not immediate, so the constructor returns before any flush attempt.
    - `protected String getObjectKeyPrefix(String ns)` returns `OBJECT_KEY_PREFIX + "/" + trim(ns)`.
    - `store(ns, batch)`: `EventsPreconditions.checkStore(ns, batch)`. Acquire `namespaceLocks.computeIfAbsent(ns, k -> new ReentrantLock())`. For each event, compute the buffer file path `<bufferRoot>/<currentFlushCycle>/cantor-events/<trim(ns)>/yyyy/MM/dd/HH/mm.<cycle>.json` (UTC); append `gson.toJson(event)` + `\n` via `Files.write(path, line, APPEND, CREATE)`. If `payload != null`, base64-encode and append to a sibling `.b64` file, then record `(offset, length)` on the event as `.cantor-payload-offset` and `.cantor-payload-length` dimensions. Track per-file offset via `payloadOffsets.get(b64Path).getAndAdd(length)`. Release the lock in a `finally`. Wrap any `IOException` / `SubstrateSdkException` via `MulticloudjUtils.wrapAsIOException("store", ns, e)`.
    - `flush()`: `oldCycle = currentFlushCycle.getAndSet(<newCycle>)`. Sleep 3 s (`SECONDS.sleep(3)`) for in-flight writes to drain (interruptible). Walk `<bufferRoot>/<oldCycle>/` for `.json` and `.b64` files; for each file, submit `MulticloudjUtils.uploadFile(bucketClient, <blobKey>, <file>)` to `uploadPool`; wait for all futures. After successful upload, recursively delete the cycle directory. On any upload failure: do NOT delete; log a `WARN`; the next flush picks it up (idempotent — provider overwrites identical key). Catch `InterruptedException` → reset interrupt flag and exit. Catch all `RuntimeException` so the scheduler thread is not killed.
    - `get(ns, start, end, mq, dq, includePayloads, ascending, limit)`: `EventsPreconditions.checkGet(ns, start, end, mq, dq)`. `Set<String> prefixes = MulticloudjUtils.getMatchingKeys(bucketClient, getObjectKeyPrefix(ns), start, end)` (rejects `start > end` per M5 fix). For each prefix, list keys under it, then for each `.json` key submit a task to `uploadPool` to: download via `MulticloudjUtils.download(...)`, split on `\n`, parse each line with Gson, filter by `(timestampMillis in [start,end]) && matchesMetadata(meta, mq) && matchesDimensions(dims, dq)`. Collect into a `ConcurrentLinkedQueue<Event>`. After all futures complete, sort by `timestampMillis` (ascending or descending per flag), then `subList(0, min(limit, size))`. If `includePayloads`, for each event with `.cantor-payload-offset` / `.cantor-payload-length`, build `DownloadRequest.builder().withKey(<b64Key>).withRange(offset, offset + length - 1).build()` via `MulticloudjUtils.downloadRange`, base64-decode, attach as `Event` payload (construct a new `Event` since payload is final).
    - `metadata(ns, key, start, end, mq, dq)`: same prefix-list + per-blob download/parse/filter, return distinct values of `metadata.get(key)`.
    - `dimension(ns, key, start, end, mq, dq)`: same, return `Event` instances containing only `timestampMillis` and `{key: dimensions.get(key)}`.
    - `expire(ns, endTimestampMillis)`: `EventsPreconditions.checkExpire(ns, endTimestampMillis)`. `MulticloudjUtils.listKeys(bucketClient, getObjectKeyPrefix(ns))` (paginated iterator — does **not** enumerate per-minute prefixes from epoch; fixes C2). For each key whose embedded time is `< endTimestampMillis`, add to deletion list. `MulticloudjUtils.deleteBatched(bucketClient, deletionList)` (batches of 100).
    - `close()`: `flushExecutor.shutdown()`; `awaitTermination(30s)`; if not terminated, `shutdownNow()`. Repeat for `uploadPool`. Do **not** close `bucketClient` (the facade owns it).
    - Package-private `forceFlushNow()`: synchronously executes the `flush()` body on the caller's thread (test hook only).
    - Tests from 8.1 must pass.
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 5.8, 5.9, 5.10, 6.1, 6.2_
    - _Commit: `feat(cantor-multicloudj): implement EventsOnMulticloudj with buffer-and-flush`_

- [x] 9. Implement `CantorOnMulticloudj` facade — test-first
  - [x] 9.1 Write `CantorOnMulticloudjTest`
    - Create `cantor-multicloudj/src/test/java/com/salesforce/cantor/multicloudj/CantorOnMulticloudjTest.java`.
    - `testObjects_returnsNonNull`: `new CantorOnMulticloudj(InMemoryBucketClients.fresh()).objects()` is non-null and an `instanceof ObjectsOnMulticloudj`.
    - `testEvents_returnsNonNull`: same for `events()`.
    - `testSets_throwsUnsupportedOperation`: `sets()` throws `UnsupportedOperationException` whose message contains `"Sets are not implemented on multicloudj"`.
    - `testConvenienceConstructor_buildsInMemoryClient`: `new CantorOnMulticloudj("inmemory", "any-region", "ctor-test")` succeeds and returns non-null objects/events.
    - `testConvenienceConstructor_missingProviderThrowsIOException`: pass an obviously-bogus provider id (`"this-provider-does-not-exist"`) and assert `IOException` whose message contains `"failed to construct BucketClient"` and the provider id (Req 2.6).
    - `testClose_externallyManagedClientNotClosed`: pass a `BucketClient` decorator that tracks `close()` calls; build `CantorOnMulticloudj(client)`; call `cantor.close()`; assert decorator's `close()` was NOT called (`ownsClient=false` path).
    - `testClose_internallyManagedClientIsClosed`: construct via the convenience constructor (any successful provider id such as `"inmemory"`); call `close()`; assert no exception. (Verifying client closure on `ownsClient=true` requires an injectable client; a smoke-only assertion is sufficient here.)
    - `testClose_isIdempotent`: call `close()` twice; second call must not throw.
    - Tests must fail to compile until 9.2 lands.
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 2.6, 8.4_
    - _Commit: `test(cantor-multicloudj): add CantorOnMulticloudj facade tests`_
  - [x] 9.2 Implement `CantorOnMulticloudj`
    - Create `cantor-multicloudj/src/main/java/com/salesforce/cantor/multicloudj/CantorOnMulticloudj.java` (public final class implementing `com.salesforce.cantor.Cantor` and `AutoCloseable`).
    - Fields: `private final BucketClient bucketClient`, `private final boolean ownsClient`, `private final ObjectsOnMulticloudj objects`, `private final EventsOnMulticloudj events`, `private final java.util.concurrent.atomic.AtomicBoolean closed = new AtomicBoolean(false)`.
    - Primary constructor `public CantorOnMulticloudj(BucketClient bucketClient) throws IOException`: validate non-null; set `ownsClient = false`; instantiate `objects = new ObjectsOnMulticloudj(bucketClient)`; instantiate `events = new EventsOnMulticloudj(bucketClient)`.
    - Convenience constructor `public CantorOnMulticloudj(String providerId, String region, String bucket) throws IOException`: validate args via `CommonPreconditions.checkString`; in a try/catch wrap `BucketClient.builder(providerId).withRegion(region).withBucket(bucket).build()`; on `SubstrateSdkException` / `IllegalStateException` / `RuntimeException` throw `IOException(String.format("failed to construct BucketClient for provider '%s': ensure blob-%s dependency is on the runtime classpath", providerId, providerId), cause)`. After successful build, set `ownsClient = true`, then call the primary-constructor logic by extracting it into a private `init(BucketClient)` helper.
    - `objects()` returns `this.objects`. `events()` returns `this.events`. `sets()` throws `new UnsupportedOperationException("Sets are not implemented on multicloudj")`.
    - `close()`: `if (!closed.compareAndSet(false, true)) return;` then `events.close()`; if `ownsClient`, call `bucketClient.close()` (note: `BucketClient.close()` declares `throws Exception`, so `CantorOnMulticloudj.close()` likewise declares `throws Exception`).
    - Tests from 9.1 must pass.
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 2.6_
    - _Commit: `feat(cantor-multicloudj): implement CantorOnMulticloudj facade`_

- [x] 10. Write the module `README.md` with the Known Limitations / Trade-offs section (`no-test`: documentation)
  - Create `cantor-multicloudj/README.md`.
  - Sections:
    - **Title & one-line description** — "Cantor on top of MultiCloudJ — cloud-agnostic object storage for AWS S3, GCP Cloud Storage, Alibaba OSS."
    - **Maven coordinates** — `com.salesforce.cantor:cantor-multicloudj:<version>`.
    - **Provider runtime table** — three rows: AWS S3 / `com.salesforce.multicloudj:blob-aws`, GCP Cloud Storage / `blob-gcp`, Alibaba OSS / `blob-ali`. Add a one-line note that `blob-inmemory` ships for test/dev use only.
    - **Java version requirement** — Java 11+ required at runtime (MultiCloudJ requirement).
    - **Quick-start (AWS)** — a 5-line snippet:
      ```java
      BucketClient bc = BucketClient.builder("aws").withRegion("us-west-2").withBucket("my-bucket").build();
      try (CantorOnMulticloudj cantor = new CantorOnMulticloudj(bc)) {
          cantor.objects().create("orders");
          cantor.objects().store("orders", "k1", "hello".getBytes(UTF_8));
      }
      ```
    - **Advanced configuration** — point readers to MultiCloudJ's `BucketClient.builder(...)` for credentials override, retry config, tracing, etc. Note that this module does NOT re-expose builder options (Req 7.2).
    - **Sets** — explicitly unsupported (matches `cantor-s3`); throws `UnsupportedOperationException`.
    - **Azure** — explicitly NOT supported because MultiCloudJ 0.4.0 has no Azure adapter.
    - **Known Limitations / Trade-offs** — copy each numbered item from `design.md` §Known Limitations / Trade-offs verbatim. **Item #1 (S3 Select / client-side filtering) MUST appear first and be prominently called out per team-lead's directive.** Specifically state: "`EventsOnMulticloudj` uses client-side filtering. MultiCloudJ does not expose an S3-Select-equivalent server-side query API, so metadata and dimension predicates are evaluated in-process after downloading each matching blob. This is correct but slower than `EventsOnS3` and incurs higher network egress proportional to the unfiltered event volume — acceptable for v1; performance-sensitive consumers should profile before adopting."
  - Do NOT touch `mkdocs.yml` or anything under `docs/` (Req 10.1 — flagged out-of-scope in design §Known Limitations item explicitly).
  - Verify: `cantor-multicloudj/README.md` renders cleanly on GitHub (visual check is fine; this is the only documentation task).
  - _Requirements: 9.1, 9.2_
  - _Commit: `docs(cantor-multicloudj): add module README with limitations and trade-offs`_

- [x] 11. Final verification (`no-test`: build-only gate; no new code)
  - Run `mvn -pl cantor-multicloudj clean install` — must succeed end-to-end (compile, test, jar).
  - Run `mvn -pl cantor-s3 test` — must still pass with the same behavior as before (Req 10.2).
  - Run `mvn clean install -DskipTests=false` at the repo root — full multi-module build must succeed.
  - Confirm `cantor-multicloudj/target/cantor-multicloudj-0.5.25-SNAPSHOT.jar` exists and contains classes under `com/salesforce/cantor/multicloudj/`.
  - Confirm no file under `cantor-s3/`, `cantor-base/`, `cantor-common/`, or any other existing module has been modified (`git diff --name-only` should show only `pom.xml` and files under `cantor-multicloudj/`).
  - If any verification fails, fix forward in a follow-up task — do NOT amend prior commits.
  - _Requirements: 1.2, 8.2, 8.3, 8.4, 8.5, 10.1, 10.2_
  - _Commit: `chore(cantor-multicloudj): no-op verification commit (allow empty if no fixups needed)`_

---

## Fix Cycle Tasks

- [x] **F1. [Security HIGH] Add max-size guard to blob downloads (OOM prevention)**
  - Source: Security audit finding HIGH-1 (CWE-400)
  - Category: Security
  - Files: `cantor-multicloudj/src/main/java/com/salesforce/cantor/multicloudj/MulticloudjUtils.java`
  - Fix: Add a configurable `MAX_DOWNLOAD_SIZE_BYTES` constant (default 256 MB). Before writing to `ByteArrayOutputStream`, check `Content-Length` header / metadata if available. During streaming download, track bytes written and throw `IOException("blob exceeds maximum download size")` if the limit is exceeded. Add a constructor parameter or static config method to override the default.
  - Acceptance criteria: downloading a blob larger than the configured limit throws `IOException` with a clear message instead of OOMing. Unit test with a mock/stub that returns oversized data.
  - Commit message: `fix(cantor-multicloudj): add max-size guard to blob downloads to prevent OOM`

- [x] **F2. [Security HIGH] Set restrictive permissions on buffer directory (PII exposure)**
  - Source: Security audit finding HIGH-2 (CWE-276/CWE-200)
  - Category: Security
  - Files: `cantor-multicloudj/src/main/java/com/salesforce/cantor/multicloudj/EventsOnMulticloudj.java`
  - Fix: When creating the buffer directory, use `Files.createDirectories(path, PosixFilePermissions.asFileAttribute(PosixFilePermissions.fromString("rwx------")))` on POSIX systems. On non-POSIX (Windows), fall back to `File.setReadable(false, false); File.setReadable(true, true)` pattern. Add a comment explaining why.
  - Acceptance criteria: buffer directory is created with 700 permissions on POSIX. Unit test verifies permissions on POSIX systems (skip on Windows).
  - Commit message: `fix(cantor-multicloudj): set restrictive permissions on event buffer directory`

- [x] **F3. [Security HIGH] Validate and canonicalize buffer directory path (path traversal)**
  - Source: Security audit finding HIGH-3 (CWE-22)
  - Category: Security
  - Files: `cantor-multicloudj/src/main/java/com/salesforce/cantor/multicloudj/EventsOnMulticloudj.java`
  - Fix: In the constructor, after the non-empty check, canonicalize the path (`path.toRealPath()` or `path.normalize().toAbsolutePath()`). Reject paths containing `..` segments. Reject paths that resolve outside an expected base (or at minimum, log a WARN if the resolved path differs from the input). Add validation that the path is a directory (not a file, symlink to sensitive location, etc.).
  - Acceptance criteria: passing `"../../etc/shadow"` or `"/tmp/../etc/passwd"` as buffer dir throws `IllegalArgumentException`. Unit test covers traversal attempts.
  - Commit message: `fix(cantor-multicloudj): validate and canonicalize buffer directory path`

- [x] **F4. [Correctness] Add flushBeforeRead() to expire() method**
  - Source: Refactorer finding R-D
  - Category: Bug
  - Files: `cantor-multicloudj/src/main/java/com/salesforce/cantor/multicloudj/EventsOnMulticloudj.java`
  - Fix: Add `flushBeforeRead()` call at the start of the `expire()` method, before the try block — same pattern as `get()`, `metadata()`, and `dimension()`. This ensures buffered events within the expiry window are flushed to storage before deletion.
  - Acceptance criteria: events stored within the expiry window and still in the local buffer are correctly expired after flush. Unit test: store events, call expire immediately (before auto-flush), verify they are deleted.
  - Commit message: `fix(cantor-multicloudj): flush buffered events before expire to prevent data loss`
