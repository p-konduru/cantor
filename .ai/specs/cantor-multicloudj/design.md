---
feature: cantor-multicloudj
phase: design
depends_on:
  - requirements.md
skill: spec-driven-development
---

# Design: cantor-multicloudj

## Overview

`cantor-multicloudj` is a new Maven module that implements Cantor's `Objects` and `Events` interfaces on top of the `com.salesforce.multicloudj:blob-client` SDK, replacing the AWS-SDK-only `cantor-s3` for users who need cloud-agnostic blob storage (AWS S3, GCP Cloud Storage, Alibaba OSS). The module sits alongside `cantor-s3` without modifying it, mirrors the `cantor-s3` package layout and object-key conventions, and corrects several latent bugs in the reference implementation rather than replicating them.

**Sets** are unsupported (matches `cantor-s3`). **Azure** is not supported because MultiCloudJ 0.4.0 has no Azure adapter.

## System Context

```
┌────────────────────────────────────────────────────────────────────┐
│                          Cantor consumer                           │
│           (cantor-server / cantor-grpc-service / library)          │
└──────────────┬─────────────────────────────────────────────────────┘
               │  Cantor / Objects / Events  (interfaces from cantor-base)
               │
┌──────────────▼─────────────────────────────────────────────────────┐
│                 cantor-multicloudj  (new module, Java 11)          │
│                                                                    │
│   CantorOnMulticloudj ── ObjectsOnMulticloudj ── (StreamingObjects)│
│        │                       │                                   │
│        │                       ├── AbstractBaseMulticloudjNamespaceable
│        │                       │                                   │
│        └─── EventsOnMulticloudj                                    │
│                                │                                   │
│                                └── MulticloudjUtils (pkg-private)  │
└──────────────┬─────────────────────────────────────────────────────┘
               │  BucketClient (com.salesforce.multicloudj:blob-client 0.4.0)
               │
   ┌───────────┼───────────────┬───────────────┬───────────────┐
   │           │               │               │               │
   ▼           ▼               ▼               ▼               ▼
blob-aws   blob-gcp        blob-ali       blob-inmemory   (future providers)
(optional) (optional)     (optional)      (test scope)
   │           │               │               │
   ▼           ▼               ▼               ▼
 AWS S3   GCP Cloud Storage  Alibaba OSS    in-process map
```

**Boundaries:**
- The module depends on `cantor-base` (interfaces) and `cantor-common` (preconditions), exactly as `cantor-s3` does.
- The module exposes only `BucketClient` for configuration — it does NOT re-export every MultiCloudJ builder option.
- No new gRPC/HTTP wiring; server modules pick the impl through their existing `Cantor` wiring.
- No edits to `cantor-s3`, `cantor-base`, `cantor-common`, or any other existing module. The ONLY change outside `cantor-multicloudj/` is one `<module>` entry in the root `pom.xml`.

## Architecture

### Package & Module Layout

```
cantor-multicloudj/
├── pom.xml
├── README.md
└── src/
    ├── main/java/com/salesforce/cantor/multicloudj/
    │   ├── CantorOnMulticloudj.java                   (public, facade, AutoCloseable)
    │   ├── AbstractBaseMulticloudjNamespaceable.java  (public abstract)
    │   ├── ObjectsOnMulticloudj.java                  (public, implements StreamingObjects)
    │   ├── EventsOnMulticloudj.java                   (public, AutoCloseable)
    │   ├── StreamingObjects.java                      (public interface, copied verbatim from cantor-s3)
    │   └── MulticloudjUtils.java                      (package-private, static helpers)
    └── test/java/com/salesforce/cantor/multicloudj/
        ├── support/InMemoryBucketClients.java         (package-private test factory)
        ├── CantorOnMulticloudjTest.java
        ├── ObjectsOnMulticloudjTest.java              (extends AbstractBaseObjectsTest)
        ├── EventsOnMulticloudjTest.java               (extends AbstractBaseEventsTest)
        ├── AbstractBaseMulticloudjNamespaceableTest.java
        └── MulticloudjUtilsTest.java
```

Package is `com.salesforce.cantor.multicloudj` (lower-case `multicloudj`, matching the upstream artifact id). All class names use the form `XOnMulticloudj` for consistency with `XOnS3`.

### Module pom.xml (full)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <artifactId>cantor-multicloudj</artifactId>
    <packaging>jar</packaging>
    <name>cantor-multicloudj</name>
    <description>Cantor on top of MultiCloudJ - cloud-agnostic object storage backend supporting AWS S3, GCP Cloud Storage, Alibaba OSS, and other MultiCloudJ providers.</description>

    <parent>
        <groupId>com.salesforce.cantor</groupId>
        <artifactId>cantor-parent</artifactId>
        <version>0.5.25-SNAPSHOT</version>
        <relativePath>../pom.xml</relativePath>
    </parent>

    <properties>
        <multicloudj.version>0.4.0</multicloudj.version>
        <!-- Override parent's Java 8 baseline: MultiCloudJ requires Java 11+ -->
        <source.version>11</source.version>
        <target.version>11</target.version>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.salesforce.cantor</groupId>
            <artifactId>cantor-common</artifactId>
        </dependency>

        <dependency>
            <groupId>com.salesforce.multicloudj</groupId>
            <artifactId>blob-client</artifactId>
            <version>${multicloudj.version}</version>
        </dependency>

        <!-- Provider runtimes: opt-in per consumer -->
        <dependency>
            <groupId>com.salesforce.multicloudj</groupId>
            <artifactId>blob-aws</artifactId>
            <version>${multicloudj.version}</version>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>com.salesforce.multicloudj</groupId>
            <artifactId>blob-gcp</artifactId>
            <version>${multicloudj.version}</version>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>com.salesforce.multicloudj</groupId>
            <artifactId>blob-ali</artifactId>
            <version>${multicloudj.version}</version>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
        </dependency>

        <dependency>
            <groupId>com.google.code.gson</groupId>
            <artifactId>gson</artifactId>
            <version>${gson.version}</version>
        </dependency>

        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
        </dependency>

        <!-- TEST -->
        <dependency>
            <groupId>org.testng</groupId>
            <artifactId>testng</artifactId>
            <version>${testng.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.salesforce.cantor</groupId>
            <artifactId>cantor-common</artifactId>
            <version>${project.version}</version>
            <type>test-jar</type>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.salesforce.multicloudj</groupId>
            <artifactId>blob-inmemory</artifactId>
            <version>${multicloudj.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Re-declare the compiler config to lock source/target to 11
                 even if the parent ever inlines its 1.8 default. Defense in depth. -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>${mvn.plugins.compiler.version}</version>
                <configuration>
                    <source>11</source>
                    <target>11</target>
                </configuration>
            </plugin>
            <!-- Skip the inherited aggregate javadoc (configured in parent) for this module
                 because it inlines source=${source.version} from the parent and would attempt
                 to aggregate Java 11 source with the parent-level source=8. We still attach
                 a per-module javadoc jar via the inherited attach-javadocs execution, which
                 uses the module-local source/target=11. -->
        </plugins>
    </build>
</project>
```

### Root pom.xml change (the only edit outside the new module)

Insert a single line in `cantor-os/pom.xml` after `<module>cantor-s3</module>` and before `<module>cantor-misc</module>`:

```xml
<module>cantor-s3</module>
<module>cantor-multicloudj</module>    <!-- NEW -->
<module>cantor-misc</module>
```

### Class-by-class design

#### `CantorOnMulticloudj implements Cantor, AutoCloseable`

```java
public final class CantorOnMulticloudj implements Cantor, AutoCloseable {
    private final BucketClient bucketClient;
    private final boolean ownsClient;            // true when we built the BucketClient internally
    private final ObjectsOnMulticloudj objects;
    private final EventsOnMulticloudj events;

    public CantorOnMulticloudj(BucketClient bucketClient) throws IOException;                            // Req 3.2
    public CantorOnMulticloudj(String providerId, String region, String bucket) throws IOException;      // Req 3.3

    @Override public Objects objects();
    @Override public Events  events();
    @Override public Sets    sets();   // throws UnsupportedOperationException("Sets are not implemented on multicloudj")
    @Override public void    close() throws Exception;
}
```

The convenience constructor delegates to the primary constructor after building the `BucketClient` via `BucketClient.builder(providerId).withRegion(region).withBucket(bucket).build()`, sets `ownsClient=true`, and wraps any `IllegalStateException` / `SubstrateSdkException` thrown by MultiCloudJ when a provider runtime is missing in an `IOException("failed to construct BucketClient for provider '<id>': ensure blob-<id> dependency is on the runtime classpath", cause)` (Req 2.6).

`close()` first calls `events.close()` (which shuts down the flush executor and upload pool), then — only when `ownsClient` is `true` — `bucketClient.close()`. Caller-owned clients are never closed by us.

#### `AbstractBaseMulticloudjNamespaceable implements Namespaceable`

```java
public abstract class AbstractBaseMulticloudjNamespaceable implements Namespaceable {
    protected static final String NAMESPACE_IDENTIFIER = ".namespace";
    protected final BucketClient bucketClient;
    protected final String bucketName;

    protected AbstractBaseMulticloudjNamespaceable(BucketClient bucketClient, String type) throws IOException;

    @Override public void create(String namespace) throws IOException;
    @Override public void drop(String namespace) throws IOException;

    protected abstract String getObjectKeyPrefix(String namespace);

    protected static String trim(String namespace);   // byte-identical to AbstractBaseS3Namespaceable.trim
}
```

Constructor calls `bucketClient.doesBucketExist()`; on `false` or a thrown `SubstrateSdkException`, wraps in `IOException("bucket '<name>' is not reachable", cause)`. `bucketName` is read from `bucketClient.getBucket()` — no separate `bucketName` parameter (one less place for callers to mis-configure).

`drop(namespace)` calls `MulticloudjUtils.deleteAllUnderPrefix(bucketClient, getObjectKeyPrefix(namespace))`. The helper batches deletes through `bucketClient.delete(Collection<BlobIdentifier>)` with a batch size of **100** (the floor across AWS=1000, GCP=100, OSS=1000) — explicitly fixing the per-key delete loop seen in `S3Utils.deleteObjects`.

`trim()` is intentionally byte-for-byte identical to the S3 version (Req 4.11). Known limitation: `String.hashCode()` is theoretically collidable at scale (32-bit), inherited from the S3 design.

#### `ObjectsOnMulticloudj extends AbstractBaseMulticloudjNamespaceable implements StreamingObjects`

```java
public final class ObjectsOnMulticloudj extends AbstractBaseMulticloudjNamespaceable implements StreamingObjects {
    private static final String OBJECT_KEY_PREFIX = "cantor-objects";

    public ObjectsOnMulticloudj(BucketClient bucketClient) throws IOException;

    @Override public void   store (String ns, String key, byte[] bytes) throws IOException;
    @Override public byte[] get   (String ns, String key) throws IOException;     // null if missing
    @Override public boolean delete(String ns, String key) throws IOException;
    @Override public Collection<String> keys(String ns, int start, int count) throws IOException;
    @Override public Collection<String> keys(String ns, String prefix, int start, int count) throws IOException;
    @Override public int    size  (String ns) throws IOException;

    // StreamingObjects
    @Override public void store (String ns, String key, InputStream stream, long length) throws IOException;
    @Override public InputStream stream(String ns, String key) throws IOException;

    @Override protected String getObjectKeyPrefix(String ns) {
        return String.format("%s/%s", OBJECT_KEY_PREFIX, trim(ns));
    }

    private String getObjectKey(String ns, String key) {
        return String.format("%s/%s", getObjectKeyPrefix(ns), key);
    }
}
```

Method-to-MultiCloudJ mapping:

| Cantor method | MultiCloudJ call | Notes |
|---|---|---|
| `store(ns, key, bytes)` | `bucketClient.upload(UploadRequest.builder().withKey(k).build(), bytes)` | wrap `SubstrateSdkException` → `IOException` |
| `store(ns, key, stream, length)` | `bucketClient.upload(UploadRequest.builder().withKey(k).build(), stream)` | **rethrows on failure** (fix vs. `ObjectsOnS3.store(stream)` which silently swallows) |
| `get(ns, key)` | if `bucketClient.doesObjectExist(k, null)` → `bucketClient.download(DownloadRequest.builder().withKey(k).build(), baos)`; return `baos.toByteArray()`; else `null` |  |
| `stream(ns, key)` | downloads fully into a `ByteArrayInputStream` and returns it; **throws `IOException` on failure** (fix vs. `ObjectsOnS3.stream` which returns `null`). Documented memory trade-off. |  |
| `delete(ns, key)` | `existed = doesObjectExist(...); bucketClient.delete(k, null); return existed;` | |
| `keys(ns, prefix, start, count)` | uses `MulticloudjUtils.listKeysOrdered(bc, getObjectKey(ns, prefix), start, count)` returning a `List<String>` (deterministic order, no double-page advance — explicitly fixes `S3Utils.getKeys` C1) | |
| `size(ns)` | `MulticloudjUtils.countKeys(bc, getObjectKeyPrefix(ns))` minus 1 for the `.namespace` marker (only when present) | |

All paths catch `SubstrateSdkException` and rewrap via `MulticloudjUtils.wrapAsIOException("<op>", ns, e)`. **No path swallows or returns `null` on error.**

#### `EventsOnMulticloudj extends AbstractBaseMulticloudjNamespaceable implements Events, AutoCloseable`

**Decision: buffer-and-flush (Option A).** Direct write per `store()` would issue one PUT per event, which is unacceptable cost on real providers. We mirror `EventsOnS3`'s buffer-then-flush pattern and key layout so operators can inspect blobs cross-module, but with explicit fixes for the latent bugs the reviewer flagged.

```java
public final class EventsOnMulticloudj extends AbstractBaseMulticloudjNamespaceable
        implements Events, AutoCloseable {
    static final String OBJECT_KEY_PREFIX = "cantor-events";
    static final int    DELETE_BATCH_MAX = 100;          // floor across AWS/GCP/Ali
    private static final long DEFAULT_FLUSH_SECONDS = 60;

    // INSTANCE-SCOPED state (no statics — fixes C3, H3)
    private final String instanceId;                     // UUID per construction
    private final Path   bufferRoot;                     // <bufferDir>/<instanceId>/
    private final ScheduledExecutorService flushExecutor;
    private final ExecutorService          uploadPool;
    private final Map<String, ReentrantLock> namespaceLocks = new ConcurrentHashMap<>();
    private final LoadingCache<String, AtomicLong> payloadOffsets;
    private final AtomicReference<String> currentFlushCycle;
    private final Gson parser = new GsonBuilder().create();

    public EventsOnMulticloudj(BucketClient bc) throws IOException;
    public EventsOnMulticloudj(BucketClient bc, String bufferDirectory) throws IOException;
    public EventsOnMulticloudj(BucketClient bc, String bufferDirectory, long flushIntervalSeconds) throws IOException;

    @Override public void store(String ns, Collection<Event> batch) throws IOException;
    @Override public List<Event> get(String ns, long start, long end,
                                    Map<String,String> mq, Map<String,String> dq,
                                    boolean includePayloads, boolean ascending, int limit) throws IOException;
    @Override public Set<String>  metadata (String ns, String key, long start, long end,
                                            Map<String,String> mq, Map<String,String> dq) throws IOException;
    @Override public List<Event>  dimension(String ns, String key, long start, long end,
                                            Map<String,String> mq, Map<String,String> dq) throws IOException;
    @Override public void expire(String ns, long endTimestampMillis) throws IOException;
    @Override public void close();

    // package-private test hook
    void forceFlushNow();

    @Override protected String getObjectKeyPrefix(String ns) {
        return String.format("%s/%s", OBJECT_KEY_PREFIX, trim(ns));
    }
}
```

Internals:

- **Buffering.** `store(ns, batch)` validates inputs (`EventsPreconditions.checkStore`) and, **without a static sifting logger**, appends JSON-encoded lines to `<bufferRoot>/<currentCycle>/cantor-events/<trim(ns)>/yyyy/MM/dd/HH/mm.<cycle>.json` using `Files.write(..., APPEND, CREATE)`, guarded by `namespaceLocks.computeIfAbsent(ns, k -> new ReentrantLock())`. Payloads are base64-encoded and appended to a sibling `.b64` file; the `(offset, length)` pair is recorded on the event as `.cantor-payload-offset` and `.cantor-payload-length` dimensions. Dropping Logback's SiftingAppender removes the cross-instance singleton issue (H3) and the unbounded appender cache leak.

- **Cycle rollover.** `currentFlushCycle` is an instance-scoped `AtomicReference<String>` initialised to `<formattedNow>.<UUID>`. `flush()` (scheduled every `flushIntervalSeconds`) does: `oldCycle = currentFlushCycle.getAndSet(newCycle)`; sleeps 3 s for in-flight writes; then walks `<bufferRoot>/<oldCycle>/` and submits each file to `uploadPool` via `MulticloudjUtils.uploadFile(bucketClient, blobKey, file)`. After all uploads succeed, the cycle directory is recursively deleted. **Failures keep the directory on disk** so the next cycle retries — flush is idempotent on the wire (provider overwrites identical key).

- **Listing for `get`/`metadata`/`dimension`.** `MulticloudjUtils.getMatchingKeys(bucketClient, getObjectKeyPrefix(ns), start, end)` expands the `[start, end]` interval to the minimal set of `yyyy/MM/dd/HH/mm` and `yyyy/MM/dd/HH/` prefixes (identical algorithm to `EventsOnS3.getMatchingKeys`) and lists each prefix via `bucketClient.list(ListBlobsRequest.builder().withPrefix(p).build())`. A precondition rejects `start > end` (fix vs. M5 — `EventsOnS3` silently treats it as `[end, end]`).

- **Filtering.** For each matching `.json` blob, `doGetOnObject` downloads the full content with `bucketClient.download(DownloadRequest.builder().withKey(k).build(), baos)`, then parses line-by-line through Gson and applies `MulticloudjUtils.matchesMetadata(meta, mq)` and `matchesDimensions(dims, dq)` in-process. **No S3 Select.** This is the load-bearing trade-off; see "Known Limitations / Trade-offs" below.

- **Payload retrieval (`includePayloads=true`).** For each matched event with `.cantor-payload-offset` / `.cantor-payload-length`, build `DownloadRequest.builder().withKey(b64Key).withRange(offset, offset + length - 1).build()`, download the range to a `ByteArrayOutputStream`, base64-decode, and attach as the event's payload. MultiCloudJ 0.4.0 supports `DownloadRequest.Builder.withRange(Long start, Long end)`. **Fallback:** if a future provider throws (`UnsupportedOperationException`/`SubstrateSdkException` whose message mentions "range"), `MulticloudjUtils.downloadRange` retries without a range and slices the buffer; the retry is logged at `WARN` once per JVM per provider.

- **`expire(ns, endTs)`.** Replaces the broken `EventsOnS3.doExpire` (C2 — would scan up to 28 M minute-prefixes from epoch). Algorithm: `listKeys(bucketClient, getObjectKeyPrefix(ns))` (paginated iterator, no per-minute enumeration), filter to `.json`/`.b64` keys whose embedded timestamp is `< endTs`, group into batches of `DELETE_BATCH_MAX` (=100), and call `bucketClient.delete(Collection<BlobIdentifier>)` per batch.

- **Concurrency.** `flushExecutor`: single-thread scheduled. `uploadPool`: 32 named fixed threads. `get`/`metadata`/`dimension` per-call fan-out reuses `uploadPool` via `CompletableFuture` (do **not** spin up fresh executors per call — small fix vs. the S3 pattern, avoids churn).

- **Lifecycle.** `close()` shuts `flushExecutor` and `uploadPool`, `awaitTermination(30s)`, then forcibly terminates if needed. It does **not** close `bucketClient` — the facade owns that.

#### `MulticloudjUtils` (package-private)

```java
final class MulticloudjUtils {
    private MulticloudjUtils() {}

    static boolean doesObjectExist(BucketClient bc, String key) throws IOException;
    static void    upload    (BucketClient bc, String key, byte[] bytes) throws IOException;
    static void    upload    (BucketClient bc, String key, InputStream in) throws IOException;
    static void    uploadFile(BucketClient bc, String key, File file) throws IOException;
    static byte[]  download  (BucketClient bc, String key) throws IOException;
    static byte[]  downloadRange(BucketClient bc, String key, long startInclusive, long endInclusive) throws IOException;

    /** Iterates the BucketClient's paginated list iterator into an ordered list of keys. */
    static List<String> listKeys(BucketClient bc, String prefix) throws IOException;

    /** Deterministic-order paging over list(): skip `start`, take `count`. Replaces the broken S3Utils.getKeys. */
    static List<String> listKeysOrdered(BucketClient bc, String prefix, int start, int count) throws IOException;

    static int  countKeys(BucketClient bc, String prefix) throws IOException;

    /** Batched delete; chunk size = EventsOnMulticloudj.DELETE_BATCH_MAX. */
    static void deleteAllUnderPrefix(BucketClient bc, String prefix) throws IOException;
    static void deleteBatched      (BucketClient bc, Collection<String> keys) throws IOException;

    /** Wrap any MultiCloudJ exception consistently. */
    static IOException wrapAsIOException(String op, String namespace, Throwable cause);

    /** Returns the set of yyyy/MM/dd/HH/[mm] prefixes that cover [startMs, endMs]. */
    static Set<String> getMatchingKeys(BucketClient bc, String namespaceKeyPrefix, long startMs, long endMs) throws IOException;

    // Filter operator helpers (= != ~ !~ > >= < <= ..)
    static boolean matchesMetadata  (Map<String,String> metadata,   Map<String,String> query);
    static boolean matchesDimensions(Map<String,Double> dimensions, Map<String,String> query);
}
```

#### `StreamingObjects`

Copied verbatim (with the package declaration switched) from `cantor-s3` so `ObjectsOnMulticloudj` exposes streaming for parity. Cross-module hoisting into `cantor-base` is out of scope (Req 10.1 forbids edits outside the new module).

### Bugs from cantor-s3 we explicitly DO NOT replicate

| ID | Reference bug | What we do instead |
|---|---|---|
| C1 | `S3Utils.getKeys` accumulates into a `HashSet` then paginates with `start/count` over unordered storage; and the "skip" branch double-advances the paginator | `MulticloudjUtils.listKeysOrdered` uses the `BucketClient.list(...)` iterator natively, returns a `List<String>` in iteration order, and increments cursor exactly once per advance |
| C2 | `EventsOnS3.doExpire` enumerates per-minute prefixes from epoch — up to ~28 M list calls for a 50-year retention | `expire()` lists the namespace prefix once via the paginated iterator, filters keys by embedded timestamp, batched deletes |
| C3 | `EventsOnS3` keeps a static `flushExecutorServices` map keyed only by `bucketName`; a second instance with the same bucket silently loses its flusher; the map is never cleaned up | All EventsOnMulticloudj state (executor, upload pool, buffer subdir, locks) is **instance-scoped**; `close()` shuts it down |
| H1 | `ObjectsOnS3.stream()` and `store(stream,length)` swallow `AmazonS3Exception` (one returns `null`, the other only logs) | Both rethrow as `IOException`. Tests assert propagation. |
| H3 | Static `siftingLogger` + Logback `SiftingAppender` is a JVM-wide singleton shared across instances; appender cache is unbounded | NIO append-mode writes guarded by a per-namespace `ReentrantLock`. No Logback runtime dependency on the hot path. |
| H4 | `flush()` reads `bufferDirectory.listFiles()` and processes every directory — cross-instance pollution if two `EventsOnS3` instances share a buffer dir | Buffer subtree is `<bufferDir>/<instanceId>/...`; flush only iterates that subtree |
| H5 | `S3Utils.deleteObjects` deletes one key at a time | `MulticloudjUtils.deleteBatched` uses `bucketClient.delete(Collection<BlobIdentifier>)` with chunk size 100 |
| M5 | `EventsOnS3.getMatchingKeys` silently mis-handles `start > end` | Precondition rejects `start > end` with `IllegalArgumentException` |

## Implementation Map

Build order (each step compiles on top of the previous):

1. `cantor-multicloudj/pom.xml` — module pom (as above).
2. `cantor-os/pom.xml` — add `<module>cantor-multicloudj</module>` after `cantor-s3` (only edit outside the new module).
3. `cantor-multicloudj/src/main/java/com/salesforce/cantor/multicloudj/StreamingObjects.java` — verbatim copy of the cantor-s3 interface with adjusted package.
4. `MulticloudjUtils.java` — static helpers, including the corrected `listKeysOrdered`, `deleteAllUnderPrefix`/`deleteBatched`, `getMatchingKeys`, `matchesMetadata`/`matchesDimensions`, `wrapAsIOException`.
5. `AbstractBaseMulticloudjNamespaceable.java` — bucket validation, `create`, `drop`, `trim`.
6. `ObjectsOnMulticloudj.java` — Objects + StreamingObjects implementation.
7. `EventsOnMulticloudj.java` — buffer-and-flush + client-side filter + range payload retrieval + correct `expire`.
8. `CantorOnMulticloudj.java` — facade + `AutoCloseable` + missing-provider error mapping.
9. `src/test/java/.../support/InMemoryBucketClients.java` — fresh-bucket-per-test factory.
10. `MulticloudjUtilsTest.java` — exhaustive coverage of `matchesMetadata`/`matchesDimensions` operators (`=`, `!=`, `~`, `!~`, `>`, `>=`, `<`, `<=`, `..`) and `listKeysOrdered` deterministic-pagination test (specifically guards against C1 regression).
11. `AbstractBaseMulticloudjNamespaceableTest.java` — namespace marker creation, idempotent create, prefix-scoped drop, `trim()` output.
12. `ObjectsOnMulticloudjTest.java extends AbstractBaseObjectsTest` — full Objects contract on `blob-inmemory`.
13. `EventsOnMulticloudjTest.java extends AbstractBaseEventsTest` — full Events contract on `blob-inmemory`, with `flushIntervalSeconds=1` plus `forceFlushNow()` hook for determinism.
14. `CantorOnMulticloudjTest.java` — facade-level: `sets()` throws UOE, missing-provider IOException with hint, idempotent `close()`.
15. `cantor-multicloudj/README.md` — Maven coords, provider table, AWS quick-start, **Known Limitations / Trade-offs** section.

## Data Models

### Object layout

```
<bucket>/
└── cantor-objects/
    └── <trim(namespace)>/
        ├── .namespace                  # marker, payload "namespace=<original>"
        └── <key>                       # raw bytes from store(ns, key, bytes)
```

### Event layout (post-flush, in the bucket)

```
<bucket>/
└── cantor-events/
    └── <trim(namespace)>/
        └── yyyy/MM/dd/HH/
            ├── mm.<cycle>.json         # one JSON-encoded Event per line
            └── mm.<cycle>.b64          # base64-encoded payload bytes,
                                        # indexed by .cantor-payload-{offset,length}
```

### Event layout (during buffering, on local disk)

```
<bufferDir>/
└── <instanceId>/
    └── <cycleName>/
        └── cantor-events/<trim(namespace)>/yyyy/MM/dd/HH/
            ├── mm.<cycle>.json
            └── mm.<cycle>.b64
```

### `Event` JSON shape (Gson defaults)

```json
{"timestampMillis": 1730000000000,
 "metadata": {"host":"web-01", "env":"prod"},
 "dimensions": {"cpu": 0.42, ".cantor-payload-offset": 0.0, ".cantor-payload-length": 2048.0},
 "payload": null}
```

`payload` is always `null` on disk; the bytes live in the sibling `.b64` file. `get(..., includePayloads=true)` rehydrates the field.

## Error Handling

| Failure mode | Behavior |
|---|---|
| Bucket unreachable at construction (`doesBucketExist` false or throws) | `IOException("bucket '<name>' is not reachable", cause)` |
| Object missing in `get(ns, key)` | returns `null` (matches `ObjectsOnS3`) |
| Object missing in `delete(ns, key)` | returns `false`, no exception |
| `start > end` in events query | `IllegalArgumentException("start must be <= end")` |
| Any `SubstrateSdkException` from MultiCloudJ on the hot path | `IOException("exception during <op> on namespace '<ns>'", e)` — never swallowed |
| Missing provider runtime (`blob-aws`/`blob-gcp`/`blob-ali` not on classpath) at facade-convenience-constructor | `IOException("failed to construct BucketClient for provider '<id>': ensure blob-<id> dependency is on the runtime classpath", cause)` |
| `InterruptedException` during flush/upload | thread interrupt flag reset, wrapped as `IOException` and rethrown |
| Flush upload failure for a cycle dir | cycle dir kept on disk; next flush retries; logged at `WARN` |
| Range download not supported by provider | one-time WARN log; full download + slice fallback |

## Testing Strategy

- **Provider.** All unit tests use `blob-inmemory` via `InMemoryBucketClients.fresh()` which returns a `BucketClient` built with `BucketClient.builder("inmemory").withBucket("test-" + UUID.randomUUID()).build()`. Each test gets a fresh bucket — no cross-test pollution.

- **Reuse contract tests.** Both `AbstractBaseObjectsTest` AND `AbstractBaseEventsTest` exist in `cantor-common`'s test-jar (verified). `ObjectsOnMulticloudjTest` and `EventsOnMulticloudjTest` extend them respectively. Tests run by default; **no `@Test(enabled=false)` unless documented with rationale**.

- **Module-specific tests.** `MulticloudjUtilsTest` covers each filter operator with TestNG `@DataProvider`, plus a `listKeysOrdered_pagination_returnsStableOrder` test that guards against the C1 regression (insert 1000 keys, page through, assert no duplicates and no gaps). `AbstractBaseMulticloudjNamespaceableTest` covers the namespace marker round-trip and `trim()`. `CantorOnMulticloudjTest` covers facade semantics.

- **Determinism for Events.** A package-private `EventsOnMulticloudj.forceFlushNow()` runs the flush synchronously on the calling thread; tests use it instead of sleeping for the flush interval.

- **In-memory provider gaps.** If `blob-inmemory` does not honor `DownloadRequest.withRange`, the payload-range test detects this via a probe (`MulticloudjUtilsTest.probeRangeSupport`) and falls through to the slice-in-memory path. The fallback is the production behavior for unsupported providers, so the test still exercises correct end-to-end behavior. If `blob-inmemory` does not support `upload(..., InputStream)`, `ObjectsOnMulticloudjTest.testStreamStore` is `@Test(enabled=false)` with a Javadoc rationale and a tracking-issue marker; this is the ONE place we accept disablement.

- **Run command.** `mvn -pl cantor-multicloudj test` (Surefire/TestNG inherited from parent). Tests must pass on JDK 11+.

## Known Limitations / Trade-offs

> **This section is mandatory in both this design doc and the module's `README.md`.** Performance-sensitive consumers must read it before adopting `cantor-multicloudj`.

1. **No S3 Select equivalent in MultiCloudJ — `EventsOnMulticloudj` uses client-side filtering.**
   `EventsOnS3` pushes metadata/dimension predicates to S3 via S3 Select; the server returns only matching rows, and only the matching bytes traverse the network. MultiCloudJ 0.4.0 exposes no equivalent server-side query API. `EventsOnMulticloudj` therefore:
   1. lists matching time-prefix blobs,
   2. downloads each `.json` blob **in full**,
   3. parses every line with Gson,
   4. applies metadata/dimension predicates in-process.
   This is correct but slower than `EventsOnS3` and incurs higher network egress proportional to the unfiltered event volume. For a 1 GB-per-namespace-per-day workload the difference is negligible; for 100 GB+ workloads with selective predicates it can be significant. Acceptable trade-off for v1. **Future work:** evaluate columnar/Parquet event storage, an external secondary index (e.g., manifest blobs per namespace/time-window with materialized metadata), or push for a `query`/`scan` API in MultiCloudJ.
2. **Java 11+ runtime.** MultiCloudJ requires Java 11. The rest of Cantor targets Java 8. `cantor-multicloudj` overrides source/target to 11 in its own pom; consumers must run on Java 11+.
3. **`Sets` unsupported.** `cantor.sets()` throws `UnsupportedOperationException` — same as `cantor-s3`. Out of scope for this module; would require a separate spec.
4. **Azure not supported.** MultiCloudJ 0.4.0 ships AWS, GCP, and Alibaba adapters only. No Azure adapter exists upstream as of June 2026.
5. **Provider runtime is opt-in.** `blob-aws` / `blob-gcp` / `blob-ali` are declared `<optional>true</optional>`. Consumers must add the runtime they need to their own POM. The facade's convenience constructor surfaces a clear `IOException` if the runtime is missing.
6. **Delete batch size capped at 100.** AWS supports 1000, GCP 100, Alibaba 1000. We use the floor for portability. Large drops/expires are paginated.
7. **In-memory provider feature coverage.** `blob-inmemory` may not implement every method identically to real providers (notably range downloads). Tests are written defensively: where the in-memory provider diverges from a real provider, the test exercises the production fallback path and a Javadoc records the limitation.
8. **`trim()` namespace collisions.** Namespace sanitisation uses `String.hashCode()` (32-bit). Collisions are theoretically findable at scale; inherited from `cantor-s3`'s design and kept for layout parity.
9. **Aggregate javadoc / Java mixed-mode.** The parent POM aggregates javadoc with `source=${source.version}` (=1.8). `cantor-multicloudj`'s sources are Java 11. The module's per-module javadoc jar uses source=11 (correct); if running `mvn site` aggregates javadoc fails on Java-11 sources, the parent's aggregate javadoc execution must either bump to `source=11` (requires JDK 11 to build the site) or exclude this module. **Out of scope for this spec; flagged for the maintainer who runs `mvn site` / `mvn release`.**
10. **Concurrent writes are serialised per-namespace.** Buffer writes are guarded by a `ReentrantLock` per namespace. Stores to different namespaces run in parallel; stores to the same namespace serialise. This matches `EventsOnS3` semantics.

## Decisions

| # | Decision | Rationale |
|---|---|---|
| D1 | Mirror `cantor-s3` layout / key conventions verbatim | Operators can inspect blobs across modules; minimises migration surface |
| D2 | Drop the static `siftingLogger` and use NIO append-mode | Fixes H3 (cross-instance singleton, unbounded appender cache) |
| D3 | Use `BucketClient.list(...)` paginated iterator directly, no manual page tracking | Fixes C1 (S3Utils.getKeys double-advance + HashSet ordering) |
| D4 | Replace `doExpire`'s per-minute prefix walk with a paginated prefix listing + timestamp filter | Fixes C2 (28M-minute DoS path) |
| D5 | Scope all Events state to the instance; `close()` shuts it down | Fixes C3 (static map leak, second-instance silent loss) |
| D6 | Per-namespace `ReentrantLock` over a single JVM-wide lock | Allows concurrent stores to different namespaces |
| D7 | Reject `start > end` instead of silently coercing | Fixes M5; surfaces caller bugs |
| D8 | Use the in-memory provider for ALL tests | Hermetic, no credentials, no flakiness; matches Req 8 |
| D9 | Extend BOTH `AbstractBaseObjectsTest` and `AbstractBaseEventsTest` | Both exist in cantor-common test-jar (verified); maximises shared contract coverage. Don't inherit the "commented-out test" anti-pattern from `cantor-s3` |
| D10 | `CantorOnMulticloudj` is `AutoCloseable` and tracks `ownsClient` | Lets callers pass externally-managed `BucketClient` without double-closing |
| D11 | Module overrides source/target to 11 in its own pom; root pom stays at 8 | Minimises blast radius; downstream consumers explicitly opt-in |
| D12 | Delete batch = 100 (provider floor) | Portable across AWS/GCP/Ali without provider-specific code |

---

**Approval question:** Does the design look good? If so, I'll generate the implementation plan in `tasks.md`.
