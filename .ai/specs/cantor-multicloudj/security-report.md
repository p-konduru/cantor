# Security Re-Audit — cantor-multicloudj (post Fix Cycle)

Date: 2026-06-30
Auditor: `security-auditor` teammate
Skill applied: `salesforce-trust-foundations:java-security`
Scope: **re-audit only** — limited to the four files changed in the Fix Cycle. The original full audit (which produced F1/F2/F3 + medium/low findings) has been superseded by this report. The full audit history is preserved in commits 5725edc, 0519e37, 66f8bf5, 6b05a1b.

Files re-audited (Fix Cycle change set):
1. `cantor-multicloudj/src/main/java/com/salesforce/cantor/multicloudj/MulticloudjUtils.java`
2. `cantor-multicloudj/src/main/java/com/salesforce/cantor/multicloudj/EventsOnMulticloudj.java`
3. `cantor-multicloudj/src/main/java/com/salesforce/cantor/multicloudj/AbstractBaseMulticloudjNamespaceable.java`
4. `cantor-multicloudj/src/main/java/com/salesforce/cantor/multicloudj/ObjectsOnMulticloudj.java`

Severity legend: **HIGH** — actionable now; **MEDIUM** — fix before broad rollout; **LOW** — defence-in-depth; **INFO** — design note.

---

## Section 1 — `salesforce-trust-foundations:java-security`

### Verification of prior HIGH findings

#### F1 — `download()` / `downloadRange()` unbounded heap (CWE-400 / CWE-770) — **CONFIRMED FIXED**

**Where:** `MulticloudjUtils.java` lines 33–214.

**Evidence the fix holds:**

- `download()` (line 98) and `downloadRange()` (line 119) now stream into `BoundedByteArrayOutputStream` (line 171), a `final` subclass of `ByteArrayOutputStream` with a 256 MiB default cap configurable via `setMaxDownloadSizeBytes()`.
- All three mutating entry points are overridden — `write(int)`, `write(byte[], int, int)`, `writeBytes(byte[])` (lines 181–196) — so providers cannot pick a different write variant to bypass the cap. `BAOS.write(byte[])` delegates to `write(byte[], int, int)` and is therefore also covered.
- `checkSize()` (line 198) uses `(long) size() + (long) incoming`, preventing int-overflow in the comparison itself.
- `downloadRange()` adds an **upfront** range-size check on line 121 (`requested = endInclusive - startInclusive + 1 > maxDownloadSizeBytes`) in addition to the bounded stream — defence in depth.
- `SubstrateSdkException` path uses `isBoundedSizeLimitCause()` (line 155) to walk the cause chain in case a provider wraps the underlying limit exception.
- The `UnsupportedOperationException` / "range not supported" provider fallback (line 137) delegates to `download(bc, key)`, which is itself bounded — fallback does not introduce an unbounded path.

**Bypass vectors considered and ruled out:**
- Subclassing the bounded stream to override `checkSize` — class is `final`.
- A provider that calls a hypothetical `write(ByteBuffer)` — `ByteArrayOutputStream` does not expose one.
- Multiple concurrent downloads each near the cap — that is a workload concern, not a bypass; cap is per-download.

#### F2 — Buffer directory readable by other local users (CWE-276 / CWE-200) — **CONFIRMED FIXED**

**Where:** `EventsOnMulticloudj.java` lines 640–675 (`createBufferRootWithRestrictivePermissions`).

**Evidence the fix holds:**
- POSIX branch creates parents *without* attrs (line 649) so existing parent perms are not clobbered, then creates the leaf with `rwx------` (700) via `PosixFilePermissions.asFileAttribute` — atomic at creation. If the leaf already exists, falls back to `Files.setPosixFilePermissions(...)` (line 655).
- Non-POSIX (Windows) branch (line 658) does best-effort `setReadable/setWritable/setExecutable` (false-for-all then true-for-owner). This is the strongest portable JDK API. Acceptable per java-security skill guidance.
- Buffered files written under the dir use the default umask, but parent dir = 700 prevents directory traversal by other unprivileged local users on POSIX — sufficient for the threat model (PII in transient buffer).

#### F3 — Path traversal on buffer directory (CWE-22) — **CONFIRMED FIXED**

**Where:** `EventsOnMulticloudj.java` lines 607–630 (`validateAndCanonicalizeBufferDirectory`).

**Evidence the fix holds:**
- Per-segment `..` rejection (line 618–623) runs **before** normalization — catches `../etc/passwd`, `/tmp/../etc`, `a/../b`, etc.
- `Paths.get(bufferDirectory)` throws `InvalidPathException` on NUL bytes / invalid sequences and is caught at line 615 → re-thrown as `IllegalArgumentException`.
- `.normalize().toAbsolutePath()` canonicalizes for downstream use.
- Rejects when the canonical path resolves to an existing non-directory (line 625) — prevents pointing at a regular file, device, or named pipe.
- Empty/null is rejected upstream by `checkString(bufferDirectory, ...)` (line 76).
- URL-encoded `..`, percent-encoding, and Unicode normalisation tricks do **not** apply — `Paths.get()` does not decode them.

#### F4 — `flushBeforeRead()` added to `expire()` (correctness fix)

Non-security. Verified at line 201 — pattern matches `get()` / `metadata()` / `dimension()`. No security regression.

---

### New findings introduced by the Fix Cycle

#### MEDIUM — none.

#### LOW — residual hardening opportunities

**L1 (LOW) — Long underflow/overflow in `downloadRange()` upfront size check.**
- **Where:** `MulticloudjUtils.java:121` — `final long requested = endInclusive - startInclusive + 1;`.
- **What:** A caller passing pathological bounds (e.g. `startInclusive = Long.MIN_VALUE`, `endInclusive = 0`) overflows `long`. After overflow `requested` may be a small positive value and the upfront guard is skipped.
- **Bypass impact:** **None for OOM** — the bounded stream still catches the actual byte count. The upfront check is defence in depth, not the only guard.
- **Fix:** Validate `0 <= startInclusive <= endInclusive` before subtracting, or use `Math.subtractExact` / `Math.addExact`.
- **Severity:** LOW — guard layered behind a hard cap.

**L2 (LOW) — Global mutable static `maxDownloadSizeBytes`.**
- **Where:** `MulticloudjUtils.java:41` — package-private `static volatile long`.
- **What:** Any class in `com.salesforce.cantor.multicloudj` can mutate the cap process-wide via `setMaxDownloadSizeBytes()`. There is no per-instance or per-tenant scoping.
- **Bypass impact:** Only callers inside the package can change it; not externally reachable. In a multi-tenant embedder, however, one tenant's config code could raise the cap and starve another.
- **Fix:** Document the static-config model in README; for multi-tenant use, plumb the cap through the constructor of `EventsOnMulticloudj` / `ObjectsOnMulticloudj`.
- **Severity:** LOW — by-design library config; acceptable for current single-tenant rollout.

**L3 (LOW) — Residual TOCTOU symlink-follow window when `bufferRoot` already exists.**
- **Where:** `EventsOnMulticloudj.java:651–657`.
- **What:** Between `Files.exists(bufferRoot)` and `Files.setPosixFilePermissions(bufferRoot, ...)`, a local attacker with write access to the *parent* directory could swap `bufferRoot` for a symlink to a sensitive path. `setPosixFilePermissions` follows symlinks by default and would chmod the target.
- **Mitigation already in place:** `bufferRoot` always includes a fresh `UUID.randomUUID()` segment (line 84), so an attacker cannot pre-create the path with a known name. Risk is residual only — requires parent-dir write + UUID prediction.
- **Fix:** JDK does not expose `NOFOLLOW_LINKS` for permission setting on the default filesystem; pragmatic acceptance. Could be tightened by `Files.readAttributes(..., NOFOLLOW_LINKS)` to verify it's not a symlink before `setPosixFilePermissions`.
- **Severity:** LOW — not exploitable without prior parent-dir access.

**L4 (LOW) — Symlink resolution at the caller-supplied bufferDirectory level is not enforced.**
- **Where:** `EventsOnMulticloudj.java:607–630`.
- **What:** A caller-supplied `bufferDirectory` like `/var/tmp/safe` where `/var/tmp/safe` is itself a symlink to `/etc/cantor-buf` would pass the `..`-segment check and the not-a-directory check. The code comment at line 610 acknowledges this ("the project has no standardized base").
- **Bypass impact:** The bufferDirectory is provided by the application embedder, not by an external attacker over the network. Trusted-embedder model — acceptable.
- **Fix:** If/when this module is embedded behind a multi-tenant API where bufferDirectory comes from external config, add `Files.readSymbolicLink` check or document a `cantor.multicloudj.buffer.root` system property as the only allowed parent.
- **Severity:** LOW.

#### Informational

- **I1** — `BoundedByteArrayOutputStream` is `final`. ✔ prevents subclass-based bypass of `checkSize`.
- **I2** — `SizeLimitExceededException extends RuntimeException` is justified by the comment ("propagates cleanly through SDK `OutputStream.write(...)` callers"). Callers convert to `IOException`; no leak.
- **I3** — `parent != null` guard before `Files.createDirectories(parent)` (line 648) handles bufferRoot at filesystem root.
- **I4** — `AbstractBaseMulticloudjNamespaceable.checkNamespaceExists` (lines 74–79) is hoisted into the base class and reused by `EventsOnMulticloudj` and `ObjectsOnMulticloudj`. No new security surface; uses existing `MulticloudjUtils.doesObjectExist`. ✔
- **I5** — `ObjectsOnMulticloudj` (re-audited) was not modified during the Fix Cycle other than to consume the hoisted `checkNamespaceExists`. No new findings in this file. Pre-existing MEDIUM/LOW notes from the original audit (stream contract, key sanitization, TOCTOU on doesObjectExist→download) remain out of Fix Cycle scope and are deferred.

---

## Section 2 — Skills not applied (with reason)

- `salesforce-trust-foundations:python-security` — no `.py` files in the Fix Cycle change set.
- `salesforce-trust-foundations:salesforce-security-coding-bestpractices` — only invoked when neither Java nor Python applies; Java applies, so it was not the controlling skill.

---

## Verdict

| Finding | Original Severity | Re-Audit Status |
|---------|-------------------|-----------------|
| F1 — OOM on blob downloads (CWE-400/770) | HIGH | **confirmed-fixed** |
| F2 — Buffer directory world-readable (CWE-276/200) | HIGH | **confirmed-fixed** |
| F3 — Path traversal on buffer directory (CWE-22) | HIGH | **confirmed-fixed** |
| F4 — `flushBeforeRead()` in `expire()` | correctness (non-sec) | confirmed-fixed |

**No new HIGH or MEDIUM findings introduced.** Four LOW residuals (L1–L4) are documented hardening opportunities, none of which materially bypass the fixes:

- L1: long-overflow in `downloadRange` precheck — still caught by bounded stream.
- L2: global mutable static for max cap — by-design library config.
- L3: TOCTOU symlink follow on existing bufferRoot — mitigated by UUID randomness.
- L4: no symlink check on caller-supplied bufferDirectory — trusted-embedder model.

**Recommendation:** safe to merge; file L1 as a follow-up hardening ticket. L2/L3/L4 are acceptable as-is.
