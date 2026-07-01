---
feature: cantor-multicloudj
phase: requirements
skill: spec-driven-development
---

# Requirements: cantor-multicloudj

## Introduction

Cantor currently provides cloud object storage through the `cantor-s3` module, which is hard-coded against the AWS SDK (`com.amazonaws:aws-java-sdk-s3`). To support additional cloud providers (Alibaba Cloud OSS, GCP Cloud Storage, future providers) without forking the module per cloud, we will introduce a new Maven module **`cantor-multicloudj`** that implements Cantor's storage interfaces (`Cantor`, `Objects`, `Events`) on top of the [`com.salesforce.multicloudj`](https://github.com/salesforce/multicloudj) blob abstraction.

The new module sits alongside `cantor-s3` and does **not** modify it. Consumers select a provider (`"aws"`, `"gcp"`, `"ali"`) and configuration (region, bucket, credentials) when constructing the Cantor instance. The implementation translates Cantor's namespace/key model into MultiCloudJ blob operations.

### Scope

- New Maven module `cantor-multicloudj` registered in the root `pom.xml` modules list.
- Java implementation under package `com.salesforce.cantor.multicloudj` (mirroring the `com.salesforce.cantor.s3` package convention).
- Implements `Cantor`, `Objects`, and `Events` (the same surface that `cantor-s3` provides). Mirrors `cantor-s3`'s decision to throw `UnsupportedOperationException` for `Sets`.
- Cloud-agnostic build-time dependency on `com.salesforce.multicloudj:blob-client`; provider runtime artifacts (`blob-aws`, `blob-gcp`, `blob-ali`) are declared as `provided` / `optional` so consumers add the runtime they need.
- Unit / integration tests use MultiCloudJ's `blob-inmemory` provider so tests run hermetically without a real cloud.

### Non-goals

- Modifying or deprecating `cantor-s3`. The existing AWS-SDK-based module remains.
- Implementing the `Sets` interface on object storage. (Matches the `cantor-s3` decision.)
- Implementing `cantor-multicloudj`-specific gRPC/HTTP wiring. Server modules (`cantor-grpc-service`, `cantor-http-service`) pick up the new implementation through their existing Cantor wiring once the module is on the classpath.
- Migration tooling between `cantor-s3` and `cantor-multicloudj` storage layouts.

## Requirements

### Requirement 1: New Maven Module

**User Story:** As a Cantor maintainer, I want a new Maven module `cantor-multicloudj` registered in the parent POM, so that the project builds the new code as a first-class artifact alongside `cantor-s3`.

#### Acceptance Criteria

1. WHEN the root `pom.xml` is processed THEN the build SHALL include a `<module>cantor-multicloudj</module>` entry, sequenced after `cantor-s3` in the modules block.
2. WHEN a developer runs `mvn -pl cantor-multicloudj clean install` from the repository root THEN the module SHALL build, run unit tests, and produce a JAR artifact named `cantor-multicloudj-<version>.jar`.
3. WHEN the module is built THEN its `pom.xml` SHALL inherit from `cantor-parent` (`com.salesforce.cantor:cantor-parent`) with `relativePath` set to `../pom.xml`, matching the convention used by `cantor-s3`.
4. WHEN the module is built THEN its `groupId` SHALL be `com.salesforce.cantor` and its `artifactId` SHALL be `cantor-multicloudj`.
5. WHEN the module is built THEN the source/target Java version SHALL be Java 11 or higher (because `com.salesforce.multicloudj` requires Java 11+), even though the rest of Cantor uses Java 8. The module's `pom.xml` SHALL override `<source>` and `<target>` accordingly without affecting other modules.

### Requirement 2: Dependency Declaration

**User Story:** As a consumer of `cantor-multicloudj`, I want to pull in only the cloud SDK I actually need (AWS, GCP, or Alibaba), so that my application doesn't ship transitive dependencies for clouds I'm not using.

#### Acceptance Criteria

1. WHEN the module's `pom.xml` declares dependencies THEN it SHALL include `com.salesforce.multicloudj:blob-client` as a compile-scope dependency.
2. WHEN the module's `pom.xml` declares provider dependencies THEN it SHALL include `com.salesforce.multicloudj:blob-aws`, `blob-gcp`, and `blob-ali` as `<optional>true</optional>` compile-scope dependencies so consumers can opt-in by mirroring the dependency in their own POM.
3. WHEN the module is built THEN it SHALL depend on `cantor-common` (compile) and `cantor-common` `test-jar` (test) - matching `cantor-s3`'s pattern - so the existing `AbstractBaseObjectsTest` can be reused.
4. WHEN the module's tests run THEN `com.salesforce.multicloudj:blob-inmemory` SHALL be on the test classpath so tests use the in-memory provider rather than a real cloud.
5. WHEN the module declares versions THEN the MultiCloudJ version SHALL be parameterised as a `<multicloudj.version>` property in the module POM (initial value: latest stable release, currently `0.4.0`).
6. IF the consumer adds `cantor-multicloudj` to their classpath WITHOUT adding a provider runtime (`blob-aws`, `blob-gcp`, or `blob-ali`) THEN attempting to build a `BucketClient` for that provider SHALL fail at runtime with a clear error indicating the missing provider module - this is delegated to MultiCloudJ's existing `ProviderSupplier` behaviour and the module's docs SHALL note the requirement.

### Requirement 3: CantorOnMulticloudj Facade

**User Story:** As an application developer, I want a single Java class that constructs a Cantor instance backed by MultiCloudJ, so that I can wire a multi-cloud Cantor with one line of code.

#### Acceptance Criteria

1. WHEN the module is consumed THEN it SHALL expose a public class `com.salesforce.cantor.multicloudj.CantorOnMulticloudj` that implements `com.salesforce.cantor.Cantor`.
2. WHEN a caller instantiates `CantorOnMulticloudj(BucketClient bucketClient)` THEN the constructor SHALL accept a fully-configured MultiCloudJ `BucketClient`, defer namespace validation to the underlying provider, and store the client for use by Objects and Events implementations.
3. WHEN a caller instantiates `CantorOnMulticloudj(String providerId, String region, String bucket)` (convenience constructor) THEN the constructor SHALL internally build a `BucketClient` via `BucketClient.builder(providerId).withRegion(region).withBucket(bucket).build()` and use it as in (2).
4. WHEN a caller invokes `cantor.objects()` THEN the facade SHALL return a non-null `Objects` instance bound to the configured bucket.
5. WHEN a caller invokes `cantor.events()` THEN the facade SHALL return a non-null `Events` instance bound to the configured bucket.
6. WHEN a caller invokes `cantor.sets()` THEN the facade SHALL throw `UnsupportedOperationException("Sets are not implemented on multicloudj")`, matching `CantorOnS3`'s behaviour.
7. WHEN the `CantorOnMulticloudj` instance is no longer needed THEN it SHALL implement `AutoCloseable` (or a `close()` method) that calls `bucketClient.close()` and releases any executor services, so callers can use try-with-resources.

### Requirement 4: ObjectsOnMulticloudj Implementation

**User Story:** As a Cantor user, I want object storage that works identically across S3, GCS, and OSS, so that I can switch providers without changing application code.

#### Acceptance Criteria

1. WHEN `ObjectsOnMulticloudj` is constructed with a `BucketClient` THEN it SHALL validate the bucket exists by calling `bucketClient.doesBucketExist()` and throw `IOException` if the bucket is unreachable or missing.
2. WHEN `store(namespace, key, bytes)` is called THEN the implementation SHALL upload the bytes to the underlying bucket at key `cantor-objects/<trimmed-namespace>/<key>` using `bucketClient.upload(new UploadRequest(objectKey), bytes)`.
3. WHEN `get(namespace, key)` is called AND the object exists THEN the implementation SHALL return the object's bytes via `bucketClient.download(new DownloadRequest(objectKey))`.
4. WHEN `get(namespace, key)` is called AND the object does NOT exist THEN the implementation SHALL return `null` (matching `ObjectsOnS3`'s contract).
5. WHEN `delete(namespace, key)` is called THEN the implementation SHALL invoke `bucketClient.delete(objectKey, null)` and return `true` if the object existed (via a prior `doesObjectExist` check), `false` otherwise.
6. WHEN `keys(namespace, prefix, start, count)` is called THEN the implementation SHALL list blobs via `bucketClient.list(new ListBlobsRequest().withPrefix(objectKeyPrefix + prefix))`, filter out the `.namespace` marker, skip `start` entries, and return at most `count` keys with the namespace prefix stripped (`-1` means no limit).
7. WHEN `size(namespace)` is called THEN the implementation SHALL iterate the listing for the namespace prefix and return the count (excluding the namespace marker).
8. WHEN `create(namespace)` is called THEN the implementation SHALL place a `.namespace` marker object at `cantor-objects/<trimmed-namespace>/.namespace` (idempotent - skip if it already exists).
9. WHEN `drop(namespace)` is called THEN the implementation SHALL list all objects under the namespace prefix and delete them in batched `delete(Collection<BlobIdentifier>)` calls.
10. WHEN any underlying MultiCloudJ call throws `SubstrateSdkException` THEN the implementation SHALL wrap it in an `IOException` with a message indicating the operation and namespace, matching `cantor-s3`'s error-handling pattern.
11. WHEN the namespace is sanitized THEN the implementation SHALL reuse the trimming logic from `AbstractBaseS3Namespaceable.trim()` (lowercase, alphanumeric/_/-/, max 64 chars + hash) - moved into a shared helper inside the new module.

### Requirement 5: EventsOnMulticloudj Implementation

**User Story:** As a Cantor user storing time-series events, I want the events API to work over MultiCloudJ blob storage, so that I can keep my events pipeline cloud-agnostic.

#### Acceptance Criteria

1. WHEN `EventsOnMulticloudj` is constructed with a `BucketClient` and optional buffer directory + flush interval THEN it SHALL validate the bucket and start a buffer-flush scheduler, mirroring `EventsOnS3`'s constructor shape.
2. WHEN `store(namespace, batch)` is called THEN the implementation SHALL append each event to local JSON-lines buffer files keyed by `cantor-events/<trimmed-namespace>/yyyy/MM/dd/HH/mm.<cycle>.json`, encode payloads as base64 in a sibling `.b64` file, and rely on the periodic flush task to upload completed buffer directories.
3. WHEN the flush cycle runs THEN it SHALL upload buffer files to the underlying bucket using `bucketClient.upload(...)` (one call per file) and delete the local buffer directory after a successful upload.
4. WHEN `get(namespace, start, end, metadataQuery, dimensionsQuery, ...)` is called THEN the implementation SHALL list matching blob keys for the timestamp range, download each `.json` blob via `bucketClient.download(...)`, parse the JSON-lines content, apply metadata/dimensions filtering in-process (since MultiCloudJ does not expose S3 Select), and return matching events sorted ascending or descending per the `ascending` flag, capped by `limit`.
5. WHEN `metadata(namespace, metadataKey, start, end, ...)` is called THEN the implementation SHALL stream the same matching blobs and return the distinct set of values for `metadataKey`.
6. WHEN `dimension(namespace, dimensionKey, start, end, ...)` is called THEN the implementation SHALL stream the same matching blobs and return events containing only the requested dimension and timestamp.
7. WHEN `expire(namespace, endTimestampMillis)` is called THEN the implementation SHALL list event blobs older than the timestamp and delete them in batched `delete(Collection<BlobIdentifier>)` calls.
8. WHEN any underlying MultiCloudJ call throws `SubstrateSdkException` or an interrupted exception THEN the implementation SHALL wrap it in an `IOException`.
9. WHEN `create(namespace)` / `drop(namespace)` is called THEN the implementation SHALL behave as Objects does (namespace marker + prefix delete) but under the `cantor-events/` prefix.
10. WHEN payload retrieval is needed AND `includePayloads=true` AND a stored event has `.cantor-payload-offset` / `.cantor-payload-length` dimensions THEN the implementation SHALL fetch the byte range from the sibling `.b64` blob (using MultiCloudJ's `DownloadRequest` range support if available; otherwise a full download + slice) and base64-decode the payload.

### Requirement 6: Shared Helpers

**User Story:** As a developer reading the module, I want shared boilerplate (preconditions, namespace trimming, blob-key building) factored into a single class, so that Objects and Events stay focused on their domain logic.

#### Acceptance Criteria

1. WHEN the module is built THEN it SHALL contain an `AbstractBaseMulticloudjNamespaceable` class that implements `Namespaceable.create` and `drop` using `BucketClient` operations, mirroring `AbstractBaseS3Namespaceable`.
2. WHEN the module is built THEN it SHALL contain a `MulticloudjUtils` class (or equivalent) with helpers for: bucket existence checks, listing keys with pagination, batched deletes by prefix, and the namespace trimming function.
3. WHEN the helpers throw exceptions THEN they SHALL throw `IOException` (or a checked subtype) consistent with `cantor-s3`'s `S3Utils` patterns.

### Requirement 7: Configuration Surface

**User Story:** As an operator deploying Cantor on different clouds, I want a single configuration knob to switch providers, so that swapping clouds is a config change, not a code change.

#### Acceptance Criteria

1. WHEN a `BucketClient` is constructed via the convenience constructor (Requirement 3.3) THEN the `providerId` parameter SHALL accept `"aws"`, `"gcp"`, `"ali"`, or `"inmemory"` (the test provider) and SHALL NOT enforce a hard-coded enum - validation is delegated to MultiCloudJ's `ProviderSupplier`.
2. WHEN advanced configuration is needed (credentials override, endpoint override, retry config, parallel upload settings, tracing policy) THEN callers SHALL be able to pass a pre-built `BucketClient` to the primary constructor (Requirement 3.2). The module SHALL NOT re-expose every builder option - we use the MultiCloudJ builder directly.
3. WHEN documentation is shipped in the module THEN the module's `pom.xml` `<description>` SHALL state "Cantor on top of MultiCloudJ - cloud-agnostic object storage backend supporting AWS S3, GCP Cloud Storage, Alibaba OSS, and other MultiCloudJ providers."

### Requirement 8: Testing

**User Story:** As a maintainer, I want unit and integration tests that exercise the module without requiring real cloud credentials, so that CI is hermetic and fast.

#### Acceptance Criteria

1. WHEN unit tests run THEN they SHALL use MultiCloudJ's `blob-inmemory` provider (`BucketClient.builder("inmemory").withBucket("test").build()`) so no network calls are made.
2. WHEN `ObjectsOnMulticloudjTest` runs THEN it SHALL extend `com.salesforce.cantor.common.AbstractBaseObjectsTest` (the shared test from `cantor-common`'s test-jar) and exercise the full Objects contract against the in-memory provider.
3. WHEN `EventsOnMulticloudjTest` runs THEN it SHALL exercise store/get/metadata/dimension/expire against the in-memory provider with a deterministic timestamp seed.
4. WHEN `CantorOnMulticloudjTest` runs THEN it SHALL verify the facade returns non-null `objects()` / `events()` and throws `UnsupportedOperationException` from `sets()`.
5. WHEN tests run THEN they SHALL use TestNG (matching the rest of the project) and SHALL produce results compatible with the existing Surefire/Failsafe configuration inherited from the parent POM.
6. IF the `blob-inmemory` provider does not fully support every MultiCloudJ operation used by `EventsOnMulticloudj` (e.g., range downloads, multipart upload) THEN tests for unsupported paths SHALL be marked with TestNG `@Test(enabled = false)` and the limitation SHALL be documented in the test class Javadoc - matching how `ObjectsOnS3Test` is currently commented out.

### Requirement 9: Documentation

**User Story:** As a new consumer, I want a short README in the module that shows me how to wire `cantor-multicloudj` for each supported cloud, so that I can adopt it without reading the source.

#### Acceptance Criteria

1. WHEN the module ships THEN it SHALL include a `cantor-multicloudj/README.md` with: (a) Maven coordinates, (b) the required provider-runtime dependency table (`blob-aws` / `blob-gcp` / `blob-ali`), (c) a 5-line Java quick-start snippet for AWS, (d) notes on Sets being unsupported, (e) a pointer to MultiCloudJ docs for advanced configuration.
2. WHEN the project's MkDocs site is built (`mkdocs.yml`) THEN the new module SHALL be referenced under the same section as `cantor-s3` (the planner SHALL leave the MkDocs nav edit out-of-scope but flag it in the design doc).

### Requirement 10: Backward Compatibility

**User Story:** As a current user of `cantor-s3`, I want the new module's introduction to leave my existing code untouched, so that I have a no-op upgrade path.

#### Acceptance Criteria

1. WHEN `cantor-multicloudj` is added to the build THEN no file under `cantor-s3/`, `cantor-base/`, `cantor-common/`, or any other existing module SHALL be modified except:
   - The root `pom.xml` (add `<module>` entry only)
   - Optionally, `mkdocs.yml` and `docs/` (out-of-scope for tasks; flag in design)
2. WHEN existing `cantor-s3` tests run (`mvn -pl cantor-s3 test`) THEN they SHALL pass with the same behaviour as before this change.

## Edge Cases & Open Questions

- **MultiCloudJ version drift.** MultiCloudJ is on an active 0.x release line (latest `0.4.0`). We pin to a specific minor version via the `<multicloudj.version>` property and accept that upgrading the property is a maintenance task.
- **Java version mismatch.** Cantor is Java 8; MultiCloudJ requires Java 11. The new module overrides source/target to 11. Downstream consumers must run on Java 11+. Document this prominently in the module README.
- **S3 Select feature gap.** `EventsOnS3` uses S3 Select for server-side filtering. MultiCloudJ does not expose Select. `EventsOnMulticloudj` falls back to client-side filtering, which is correct but slower for large event volumes. Acceptable trade-off for v1; performance optimization is out-of-scope.
- **In-memory provider coverage.** `blob-inmemory` may not implement every method (range downloads, multipart, presigned URLs). Tests that exercise those paths must be skipped or use mocks. The design phase should confirm which methods are exercised.
- **`drop()` semantics on listing pagination.** Large namespaces require multi-page listing + batched deletes. Design should specify page size and batch size limits (MultiCloudJ delete-batch limits vary by provider).
- **Sets.** Like `cantor-s3`, we throw `UnsupportedOperationException`. If a future PR wants to implement Sets on object storage, it lives in a different spec.

---

**Approval question:** Do these requirements look good? If so, I'll move to design (Phase 2) where I'll dispatch the code-explorer, code-architect, and code-reviewer agents in parallel to produce `design.md`.
