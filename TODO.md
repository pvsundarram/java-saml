# java-saml Modernization TODO

Goal: run cleanly on **JRE 25**, target **Java 21** (LTS), deploy under
**Spring Boot 4** (Jakarta EE 11), and clear the outstanding CVEs — while
keeping the public API and the way callers consume it as close to the
current shape as possible.

Legend: `[ ]` pending · `[x]` done · `[~]` in progress · `[!]` blocked / needs a decision

---

## Phase 0 — Baseline (done during survey)

- [x] Locate a JDK 25 (Homebrew OpenJDK 25.0.2) and confirm Maven uses it
- [x] Baseline build on JDK 25 — **BUILD SUCCESS**
- [x] Baseline test run — 372/373 pass; 1 failure is a JDK-version-specific
      assertion on an exception string, not a real defect
- [x] Full dependency tree extracted (52 third-party artifacts)
- [x] OSV vulnerability scan — **65 advisories: 1 critical, 27 high, 31 moderate, 6 low**
- [x] Manual security review of the SAML-critical code paths
- [x] Confirm target: Java 21, jakarta.servlet, API shape preserved

## Phase 1 — Build toolchain

- [x] 1.1 Parent pom: `source/target 1.8` → `maven.compiler.release 21`
- [x] 1.2 Enforcer: require Java 21+, bump Maven floor
- [x] 1.3 Drop the obsolete `evaluateBeanshell` JCE-strength rule in `core`
      (unlimited crypto policy has been the default since Java 9) and retire
      maven-enforcer-plugin 1.4.1
- [x] 1.4 Bump all Maven plugins to current releases
- [x] 1.5 Fix surefire config: `inputEncoding`/`outputEncoding` are not
      surefire parameters and warn on every build
- [x] 1.6 `.java-version` 1.8 → 21
- [x] 1.7 Refresh CI (GitHub Actions); retire the dead `.travis.yml`

## Phase 2 — Dependency CVEs

Direct, security-critical for a SAML library:

- [x] 2.1 **xmlsec 3.0.2 → 4.0.4** — CVE-2023-44483, private key disclosure
- [x] 2.2 **commons-lang3 3.13.0 → 3.20.0** — CVE-2025-48924, uncontrolled recursion
- [x] 2.3 commons-codec 1.16.0 → 1.22.1
- [x] 2.4 slf4j 1.7.36 → 2.0.18
- [x] 2.5 **logback 1.2.12 → 1.5.38** — 8 advisories incl. CVE-2023-6378 (HIGH)

Optional Azure Key Vault HSM chain (source of ~55 of the 65 advisories):

- [x] 2.6 **azure-security-keyvault-keys 4.7.0 → 4.11.2** — CVE-2026-33117 (**CRITICAL**)
- [x] 2.7 **azure-identity 1.10.1 → 1.18.6** — CVE-2024-35255; also drags
      netty / jackson / reactor forward, clearing the transitive backlog

Test-scope:

- [x] 2.8 mockito 3.12.4 → 5.23.0 (3.x cannot instrument JDK 25 bytecode)
- [x] 2.9 hamcrest 2.2 → 3.0

## Phase 3 — Jakarta EE 11 migration (required by Spring Boot 4)

- [x] 3.1 `javax.servlet-api 4.0.1` → `jakarta.servlet-api 6.1.0`
- [x] 3.2 Rewrite `javax.servlet.*` imports in `Auth` and `ServletUtils`
      — package swap only, signatures unchanged
- [x] 3.3 Update toolkit tests to the jakarta namespace
- [x] 3.4 Migrate the JSP sample webapp (imports, `web.xml` to Servlet 6.0)
- [x] 3.5 Confirm `core` stays servlet-free (it should need no change)

## Phase 4 — Code-level security hardening

- [x] 4.1 Fix `UtilsTest.testGetNameIdDataWrongKey` — asserts on a JDK-internal
      exception message that changed after JDK 8
- [x] 4.2 **Reject SHA-1 signatures by default.** `rejectDeprecatedAlg`
      currently defaults to `false`, so `rsa-sha1`/`dsa-sha1` assertions are
      accepted out of the box. Flip the default; the existing
      `onelogin.saml2.security.reject_deprecated_alg` property remains the
      escape hatch for an IdP that still needs SHA-1.
- [x] 4.3 Enable `FEATURE_SECURE_PROCESSING` on the `XPathFactory`
- [x] 4.4 Resolve deprecated-API warnings — build is now warning-free by
      default. Fixed: `new URL(String)` (deprecated since Java 20) → `new
      URI(..).toURL()`, `new Base64(int)` → `Base64.builder()`, and a missing
      `@Deprecated` on `Auth.logout(..)`. Two deprecations deliberately left
      in place — see "Deferred" below.
- [x] 4.5 Re-check XXE hardening still holds under the new xmlsec

## Phase 5 — Verification

- [x] 5.1 Clean build on JDK 25
- [x] 5.2 Full test suite green
- [x] 5.3 Re-run the OSV scan; confirm the tree is clear
- [x] 5.4 Verify the built jar's bytecode is class-file major 65 (Java 21)
- [x] 5.5 Smoke-test that the artifact loads and runs on JRE 25
- [x] 5.6 Update README (Java baseline, jakarta, changed defaults)

---

## Security review findings (code, not dependencies)

Recorded during the survey. The library's SAML handling is in better shape
than expected — the items below are the gaps, not a full list of what it does.

**Holds up well:**
- XXE defence in `Util.parseXML` is thorough — `disallow-doctype-decl`,
  external entities off, secure processing, plus a belt-and-braces
  `<!ENTITY` string check in `loadXML`
- Signature-wrapping (XSW) defence is genuinely solid: exactly one Assertion
  document-wide, every `//ds:Signature` must parent to Response or Assertion,
  each Reference URI must equal its parent's `ID`, duplicate IDs and
  References rejected, exactly one Reference per Signature, and the signature
  position is pinned by XPath
- Santuario secure validation is on (`new XMLSignature(el, "", true)`)
- All text extraction uses `getTextContent()`, so the CVE-2017-11427
  comment-truncation attack does not apply

**Gaps to close:** see 4.2 and 4.3 above.


---

## Deferred, with reasons

**`org.apache.commons.lang3.text.StrSubstitutor` (13 call sites) and
`org.apache.commons.lang3.StringEscapeUtils` (1 call site).** Both are
deprecated in commons-lang3 in favour of `commons-text`. Left as-is
deliberately:

- They are deprecated, not removed, and commons-lang3 3.x keeps them.
- The successor would be a **new runtime dependency** on `commons-text` for
  every consumer, which cuts against keeping consumption unchanged.
- `commons-text`'s `StringSubstitutor` is the exact class behind Text4Shell
  (CVE-2022-42889). The plain map-backed constructor is not affected, but
  adding that dependency to a security library to resolve a cosmetic
  deprecation is a poor trade.
- `StringEscapeUtils.escapeXml10` does XML escaping. Hand-rolling a
  replacement would be strictly worse for security than keeping the
  deprecated-but-correct implementation.

Revisit when commons-lang3 4.0 actually removes the package.

**`this-escape` lint warnings (7 sites).** A Java 21 lint about calling
overridable methods from constructors, in `Metadata`, `LogoutRequest`,
`LogoutResponse`, `SamlResponse` and `AuthnRequest`. Real but latent — it
would only bite a subclass, and fixing it means restructuring constructors in
the SAML message classes. Not worth the churn inside this scope; noted for a
follow-up.

**`Util.getXPathFactory()` asks for the JDK-internal
`com.sun.org.apache.xpath.internal.jaxp.XPathFactoryImpl` by name.** Verified
working on JDK 25 — JAXP's FactoryFinder special-cases the built-in
implementations, so the module system does not block it. Fragile rather than
broken, and it already falls back to `XPathFactory.newInstance()`. Left alone.
