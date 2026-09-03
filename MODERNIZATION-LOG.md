# Modernization Log

Running log of the JRE 25 upgrade + security hardening work on this fork of
the OneLogin/SAML-Toolkits `java-saml` library.

Format: newest entries at the bottom. Each entry notes what was done, why, and
what the observed result was.

---

## 2026-09-03

### Session start — environment survey

- Repo: `java-saml-toolkit` 2.9.1-SNAPSHOT, multi-module (`core`, `toolkit`, `samples`).
- `.java-version` pins **1.8**; parent pom compiles with `source/target 1.8`.
- No JDK on `PATH`. Homebrew has **OpenJDK 25.0.2** at
  `/opt/homebrew/Cellar/openjdk/25.0.2/libexec/openjdk.jdk/Contents/Home`,
  which Maven 3.9.15 already picks up as its runtime.
- Action: exporting `JAVA_HOME` to that JDK for all build commands.
- Local Maven repo is cold, so the first build must run online.

### Baseline on JDK 25

Ran the existing build unchanged against OpenJDK 25.0.2:

- `mvn -DskipTests package` → **BUILD SUCCESS**. The 1.8 source/target still
  compiles; JDK 25 accepts `-source 8` but warns it "is obsolete and will be
  removed in a future release".
- `mvn test` → **372 of 373 pass**. The single failure is
  `UtilsTest.testGetNameIdDataWrongKey`, which asserts the exception message
  contains `"algid parse error, not a sequence"`. Modern JDKs report
  `"Unable to decode key"` instead. A test-only issue, not a library defect.

So the code itself is already largely JDK 25 clean. The work is in the
dependencies and the build.

### Probing the risky bits

`Util.getXPathFactory()` asks for the JDK-internal
`com.sun.org.apache.xpath.internal.jaxp.XPathFactoryImpl` by name. Since that
package is not exported from the `java.xml` module, I expected this to fail
under the module system. Wrote a probe and ran it on 25 — it resolves fine
(JAXP's FactoryFinder special-cases the built-in implementations). Leaving it
alone, noting it as fragile rather than broken.

Also checked the three `DocumentBuilderFactory` instances in `Util` that skip
the XXE hardening (`copyDocument`, `addSign(Node…)`, `generateNameId`). All
three only ever call `newDocument()` — they never parse untrusted input, so
they are not XXE vectors. No change needed.

### Dependency vulnerability scan

Extracted the full resolved tree (52 third-party artifacts) and queried the
OSV database. Result: **65 advisories — 1 critical, 27 high, 31 moderate, 6 low.**

The one critical and most of the highs arrive through the *optional* Azure Key
Vault HSM dependency, which pulls in netty, jackson, reactor and woodstox. They
are optional to Maven but very much present on the classpath of anyone using
that feature.

Most security-relevant for a SAML library specifically:

| Artifact | Current | Advisory | Fixed in |
|---|---|---|---|
| `org.apache.santuario:xmlsec` | 3.0.2 | CVE-2023-44483 — private key disclosure | 3.0.3 / 4.0.4 |
| `com.azure:azure-security-keyvault-keys` | 4.7.0 | CVE-2026-33117 — **CRITICAL**, security feature bypass | 4.10.6+ |
| `com.azure:azure-identity` | 1.10.1 | CVE-2024-35255 — elevation of privilege | 1.12.2+ |
| `ch.qos.logback:*` | 1.2.12 | 8 advisories, incl. CVE-2023-6378 (HIGH) | 1.2.13+ / 1.5.x |
| `org.apache.commons:commons-lang3` | 3.13.0 | CVE-2025-48924 — uncontrolled recursion | 3.18.0+ |

### Manual security review of the SAML paths

Read `Util.validateSign`, `getSignatureData`, `validateSignNode`,
`SamlResponse.processSignedElements`, `validateSignedElements` and
`validateNumAssertions`.

The signature-wrapping defences are genuinely thorough — this fork has the
post-CVE hardening. Documented in detail in `TODO.md`. Two real gaps found:

1. **SHA-1 accepted by default.** `isAlgorithmWhitelisted` permits `rsa-sha1`
   and `dsa-sha1`, and `Saml2Settings.rejectDeprecatedAlg` defaults to
   `false` — so a SHA-1-signed assertion is accepted unless the deployer opts
   out. Given practical chosen-prefix collisions against SHA-1, this should
   fail closed.
2. **`XPathFactory` has no secure-processing feature set**, unlike the
   `DocumentBuilderFactory` alongside it.

### Direction confirmed with the user

- Deployment target is **Spring Boot 4** → Jakarta EE 11, so the
  `javax.servlet` → `jakarta.servlet` migration is required, not optional.
- Compile target **Java 21** (LTS, runs on 25).
- **Keep the public API and consumption mechanics as close as possible** —
  the servlet migration is a package swap, not a redesign.

Wrote `TODO.md` with the full plan. Starting Phase 1.

### Phase 1–3 complete — build, dependencies, Jakarta

**Build toolchain.** Parent pom now sets `maven.compiler.release 21` (replacing
`source`/`target` 1.8, which also removes the "source value 8 is obsolete"
warning). Enforcer requires Java 21+ and Maven 3.6.3+. All Maven plugins moved
to current releases: compiler 3.16.0, surefire 3.5.6, jar 3.5.1, enforcer
3.6.3, javadoc 3.12.0, source 3.4.0, gpg 3.2.8, release 3.3.1, jacoco 0.8.15,
dependency-check 13.0.0. `.java-version` → 21.

Two build-config cleanups along the way:

- The `core` module carried a `maven-enforcer-plugin` **1.4.1** execution whose
  only rule was a beanshell check that `Cipher.getMaxAllowedKeyLength("AES") > 128`.
  That guarded against the old restricted-strength JCE policy files, which
  have not been a thing since Java 9 — unlimited is the default. Removed the
  execution and with it the dependency on a decade-old plugin.
- surefire was passing `inputEncoding` and `outputEncoding`, which are not
  surefire parameters; they produced a `[WARNING] Parameter ... is unknown`
  on every single build. Dropped.

**Dependencies.** Versions are now centralised as properties in the parent pom
rather than scattered as literals across three modules:

| | from | to |
|---|---|---|
| xmlsec | 3.0.2 | **4.0.4** |
| commons-lang3 | 3.13.0 | **3.20.0** |
| commons-codec | 1.16.0 | 1.22.1 |
| slf4j-api | 1.7.36 | 2.0.18 |
| logback | 1.2.12 | **1.5.38** |
| azure-security-keyvault-keys | 4.7.0 | **4.11.2** |
| azure-identity | 1.10.1 | **1.18.6** |
| mockito-core | 3.12.4 | 5.23.0 |
| hamcrest | 2.2 | 3.0 |

xmlsec 4.0.4 was the upgrade I expected to hurt — it is a major version bump
on the library that does all the signature and encryption work. It compiled
and tested clean with no source changes at all.

**Jakarta EE 11.** `javax.servlet` appeared in exactly four files and only ever
as import statements — `Auth`, `ServletUtils`, and their two tests. Swapped to
`jakarta.servlet`, with `jakarta.servlet-api 6.1.0` replacing
`javax.servlet-api 4.0.1`. No method signature changed, so callers only need
to update their own imports. `core` never touched the servlet API, as expected.
The JSP sample's `web.xml` moved from Servlet 2.5 to 6.0; its JSPs only use
implicit objects, so they needed no edits.

**Two source fixes were required:**

1. `AuthTest` imported `org.mockito.Matchers`, removed in Mockito 3. Changed to
   `org.mockito.ArgumentMatchers` — same method, renamed class.
2. `UtilsTest.testGetNameIdDataWrongKey` asserted the exception *message*
   contained `"algid parse error, not a sequence"`. That string is a JDK
   implementation detail that changed after Java 8 (now "Unable to decode
   key"). Rewrote it to assert the exception *type* — `InvalidKeySpecException`
   — which is the actual contract: a malformed PKCS#8 key must be rejected.

**Result: BUILD SUCCESS, 464 tests, 0 failures** (373 core + 91 toolkit).

### Phase 4 — code-level hardening

**Signature algorithm defaults.** The manual review turned up something worse
than the inbound-SHA-1 issue I originally logged: `Saml2Settings` also defaulted
its *outbound* algorithms to SHA-1 —

```java
private String signatureAlgorithm = Constants.RSA_SHA1;
private String digestAlgorithm   = Constants.SHA1;
```

so the toolkit was signing its own AuthnRequests, LogoutRequests and metadata
with RSA-SHA1/SHA-1 unless the deployer knew to override it. Changed all three
defaults:

| Setting | Was | Now |
|---|---|---|
| `signatureAlgorithm` | `RSA_SHA1` | `RSA_SHA256` |
| `digestAlgorithm` | `SHA1` | `SHA256` |
| `rejectDeprecatedAlg` | `false` | `true` |

All three stay configurable through the existing properties, so an IdP that
still needs SHA-1 has a one-line escape hatch. **This is a behaviour change and
is called out at the top of the README** — an integration against a SHA-1 IdP
will start failing validation until it opts out.

**XPathFactory secure processing.** `parseXML` hardened its
`DocumentBuilderFactory` carefully but the `XPathFactory` next to it had no
equivalent. Added a `secured(..)` helper that sets
`FEATURE_SECURE_PROCESSING`, applied to both the primary and fallback factory,
degrading gracefully if an implementation does not support it.

**Deprecated APIs.** The build is now warning-free by default. Fixed
`new URL(String)` (deprecated since Java 20) → `new URI(..).toURL()`, which
also adds RFC 3986 validation that `URL` never did; `new Base64(int)` →
`Base64.builder().setLineLength(64).get()`; and a missing `@Deprecated`
annotation on `Auth.logout(..)`. Two deprecations were left in place on
purpose — reasoning recorded under "Deferred" in `TODO.md`.

### Fallout from the changed defaults, and how it was resolved

Hardening the defaults broke 34 tests. Rather than weaken the defaults, I
sorted the failures by what each test was actually for:

- **27 tests** were exercising something else entirely — expiry, destination,
  request IDs, subject confirmation — and merely happened to use SAML fixtures
  that were signed with rsa-sha1 years ago. Mapped each failing test back to
  the config file it loads; they came from exactly five configs
  (`config.my`, `config.adfs`, `config.mywithmulticert`, `config.mywithnocert`,
  `config.noreferenceuri`). Added
  `onelogin.saml2.security.reject_deprecated_alg = false` to those five, with a
  comment explaining that they represent SHA-1-era IdPs. The tests now keep
  testing what they were written to test.
- **7 tests** in `SettingBuilderTest` / `IdPMetadataParserTest` were asserting
  the default algorithm itself. Verified first that none of the configs they
  load actually set an algorithm — so they really were asserting the default —
  then updated the expected values to `RSA_SHA256` / `SHA256`.

Result: 464 tests, 0 failures.

### Phase 5 — verification

**Build.** Clean `mvn package` on JDK 25: BUILD SUCCESS, **464 tests, 0
failures**, and zero compiler warnings. Compiled classes are class-file major
version **65 = Java 21**, as intended.

**XXE defences re-verified under xmlsec 4.** Rather than trust that the
existing tests still meant something after a major-version bump, I wrote a
probe against the built library. Four attacks and one control:

| Attack | Blocked |
|---|---|
| External entity → `file:///etc/passwd` | yes |
| External DTD subset, **no `<!ENTITY` literal** | yes |
| Parameter entity → remote DTD (OOB exfiltration shape) | yes |
| Billion laughs | yes |
| Benign document (control) | parsed correctly |

The second case matters most: it contains no `<!ENTITY` string, so it slips
past the crude filter in `loadXML` and has to be stopped by
`disallow-doctype-decl` in the parser itself. It was — confirming the real
hardening works, not just the string check.

**End-to-end smoke test on JRE 25.** Ran the library against a live JVM 25
exercising the actual SAML paths — 14 checks, all passing: settings loading,
the three hardened defaults being in effect, AuthnRequest generation (deflate +
base64 + XML), SP metadata generation and XSD validation, signing with
RSA-SHA256 through xmlsec 4, verifying that signature, **rejecting a tampered
document**, and confirming the SHA-1 switch changes behaviour in *both*
directions (rejects when on, accepts when off).

One check failed on first run — "tampered document rejected". That turned out
to be a bug in my probe, not the library: I tampered with `entityId=` but SAML
metadata spells it `entityID=`, so nothing was actually modified and the
signature legitimately verified. Fixed the probe and added a guard that the
tampering actually changed the string; it then passed.

**Dependency re-scan.** Re-resolved the tree and re-queried OSV. One advisory
survived the upgrades: `netty-codec-dns` sat at 4.1.135 (CVE-2026-73508) while
the rest of Netty resolved to 4.1.137, because Azure's BOM pins the DNS modules
a couple of patches behind. Imported `io.netty:netty-bom` into
`dependencyManagement` to hold every Netty module at one version.

**Final scan: 0 advisories across 52 artifacts** (from 65).

**CI.** GitHub Actions now builds on Java 21 and 25 with Temurin and Maven
caching, on `actions/checkout@v4` / `setup-java@v4` (was v2 on the retired
`adopt` distribution, testing Java 8 and 11). Split dependency-check into its
own job so a slow NVD feed cannot break the build matrix. Deleted `.travis.yml`
— Travis has not run this project in years and the badge was dead.

**README** updated: Java 21 requirement, the Jakarta migration, the three
changed security defaults with the opt-out, and the obsolete JCE policy-file
instructions removed (unlimited strength has been the default since JDK 9).

### Done

All 39 planned items are closed. Three items were deliberately deferred with
reasoning recorded in `TODO.md`: the `commons-text` migration (would add a
runtime dependency to resolve a cosmetic deprecation), the `this-escape` lint
warnings, and the JDK-internal `XPathFactoryImpl` lookup (verified working).
