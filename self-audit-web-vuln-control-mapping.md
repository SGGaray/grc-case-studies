# Auditing My Own Tool
### A Self-Audit of web-vuln-control-mapping, Four Real Findings and What They Taught Me About Risk

**Case Study | Sebastian Garay | GRC, Human Risk & Shadow AI**

> This is a genuine self-audit, not a demonstration exercise. Every finding below was discovered while actively building and maintaining this project, and every fix was applied to the real, live repository. Dates and commit messages are accurate. Nothing here is staged.

## Why Audit Your Own Code

Most portfolio case studies analyze someone else's breach. That is useful, but it carries a risk: it is easy to sound wise about failures you did not live through. This one is different. `web-vuln-control-mapping` is my own tool, a small Next.js app that maps common web vulnerabilities to risk and governance controls. Over the course of building and maintaining it, four real issues surfaced. I am documenting them the way I would document findings in a client engagement: condition, criteria, cause, effect, and recommendation, because the format matters more when the subject is your own work. It is easier to be honest about someone else's mistakes than your own.

## Scope and Method

The audit covers the application code, its dependency posture, and its data layer, as they existed across several weeks of active development. Findings were discovered through a mix of manual code review, dependency scanning (`npm audit`), and hands-on testing of both API endpoints. I did not go looking for problems to pad this report. Every finding here reflects a real gap, not a hypothetical one.

## Finding 1: API Endpoints Accepted Unbounded Input

**Condition:** The two server endpoints, `/api/hash` and `/api/headers`, validated that input fields were the correct type (string), but placed no limit on their length.

**Criteria:** Any endpoint that accepts user-supplied input should bound that input's size before processing it. This is a basic control under NIST CSF's Protect function (PR.PS, Platform Security) and is standard practice for any public-facing API.

**Cause:** The original validation logic checked correctness (is this a string?) but not scale (how large is this string?). This is a common omission because the happy path, someone pasting a normal password or a normal set of HTTP headers, never exercises the missing boundary.

**Effect:** A caller could submit an arbitrarily large payload (megabytes of text) to either endpoint, forcing the server to hash or parse it, tying up CPU with no cost to the caller. A trivial, unauthenticated denial-of-service vector, on a tool whose entire purpose is teaching people to think about this exact class of risk.

**Recommendation and remediation:** Added explicit maximum length checks (100,000 characters for `/api/hash`, 20,000 for `/api/headers`) returning HTTP 413 when exceeded. Covered both limits with unit tests asserting the 413 response. Fixed in commit `fix(security): add input length limits on API routes and security headers`.

## Finding 2: The Tool That Audits Security Headers Had None of Its Own

**Condition:** `web-vuln-control-mapping` includes a Header Analyzer that checks a target site for six standard security headers (HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy, and CSP). The app's own deployment set none of them.

**Criteria:** ISO/IEC 27001:2022 Annex A.8.9 (Configuration Management) and basic web security baselines expect an application to apply the protections it is capable of applying to itself, not only to describe them for others.

**Cause:** Next.js does not set these headers by default, and adding them requires an explicit `headers()` function in the config file, a step easy to skip when the visible feature (the analyzer tool) already works correctly in isolation.

**Effect:** Beyond the direct exposure (clickjacking, MIME sniffing, referrer leakage), there was a credibility problem: anyone technical enough to run my own tool against my own site would find it failing its own check. That is a worse outcome than the missing headers themselves.

**Recommendation and remediation:** Added five of the six headers directly in `next.config.mjs` (Strict-Transport-Security, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy). Deliberately did not add Content-Security-Policy yet, a misconfigured CSP can break the app's own scripts and styles, and shipping a broken CSP is worse than shipping none. That remains an open, documented item rather than a rushed fix. Fixed in the same commit as Finding 1.

## Finding 3: Framework Dependency Carried Fourteen Unpatched Advisories

**Condition:** The project ran Next.js 14.2.35, the last version in its release line, against which `npm audit` reported fourteen separate published advisories (denial of service, cache poisoning, request smuggling, cross-site scripting, and server-side request forgery variants), aggregated under a single high-severity entry.

**Criteria:** Dependencies with known, disclosed vulnerabilities and no available patch in their current line should be triaged and upgraded on a defined cycle, not left indefinitely because the current version still runs. This maps to NIST CSF's Identify function (ID.RA, Risk Assessment) applied to third-party components.

**Cause:** The 14.x line had reached its final release; every fix for these advisories only existed in the next major version. Upgrading a major framework version carries its own risk of breaking changes, which is why it had been deferred.

**Effect:** Before triage, the raw count (fourteen advisories, one rated high) looked alarming. After actually reading each one against the codebase, eight of the fourteen turned out to be inapplicable: they required features this project does not use at all (the image optimizer, middleware, i18n routing, rewrites, WebSocket upgrades, CSP nonces). The real residual exposure was narrower than the count suggested, but not zero.

**Recommendation and remediation:** Upgraded to Next.js 15 and React 19 together, using a dedicated branch, a full local build and test pass, and a manual smoke test before merging. Verified the fix with a follow-up `npm audit`, which dropped from fourteen high-severity advisories to a single unrelated moderate one (an internal PostCSS dependency, not exploitable in this project's build process). Fixed in commit `chore: upgrade to Next.js 15, React 19, and lucide-react latest`.

## Finding 4: The Tool Named for Control Mapping Never Populated Its Controls

**Condition:** The project is named `web-vuln-control-mapping`, and its README and GitHub description both promise that each payload maps to "a related control (NIST 800-53, ISO 27001 Annex A, OWASP ASVS)." The data layer told a different story. In `lib/explain.ts`, the `owasp`, `control`, and `mitigation` fields were declared as optional and left empty on all 23 entries, and the Explain panel in `PayloadGenerator.tsx` rendered only `summary`, `why`, and `when`. The control mapping the project is named for did not exist in the data or the UI.

**Criteria:** The name and stated purpose of an artifact must match what it delivers. This is the honesty-first standard I apply to my own case studies, that where I interpret rather than report fact, I say so, and it applies most strictly to a tool whose entire premise is connecting a technical finding to a governance control.

**Cause:** The type schema made the three governance fields optional (`owasp?`, `control?`, `mitigation?`), so an entry was structurally valid with them empty. The README described the intended design rather than the shipped state. And unlike Findings 1 through 3, this gap never produced a runtime error, no build failed and no page broke, so nothing forced it to surface. It was a content gap, invisible to every test.

**Effect:** This is the most consequential place a gap could sit. The differentiator of the whole project is connecting the technical layer to governance, and a reviewer who opened the repo named for that exact mapping would have found the mapping absent. A tool that names a capability it does not deliver undercuts its own premise more than a missing feature elsewhere would.

**Recommendation and remediation:** Made `owasp`, `control`, and `mitigation` required fields in the `PayloadExplanation` type, so an empty one is now a compile error rather than a silent omission. Populated all 23 entries with their real OWASP Top 10 2021 category, a mapped control (NIST SP 800-53 SI-10 / ISO 27001:2022 Annex A.8.28 and related), and a concrete per-technique mitigation. Updated `PayloadGenerator.tsx` to render the three governance fields in the Explain panel, visually separated from the conceptual fields. Verified with a clean `tsc --noEmit`, a passing production build, and the existing test suite. Fixed in commit `feat(explain): populate owasp/control/mitigation fields across all 23 payloads`.

## What These Four Findings Have in Common

Findings 1 through 3 are friction-driven shortcuts, not knowledge gaps. I knew input should be bounded; I just had not gotten to it yet. I knew a tool that checks for security headers should set its own; it was easy to skip once the visible feature worked in isolation. I knew the framework was overdue for a major upgrade; the breaking-change risk kept getting deferred.

Finding 4 is a different animal, and worth naming as such: not a shortcut taken under pressure, but a promise written before the work behind it was finished. The README described where the tool was going, and the data never caught up. That is its own failure mode, the artifact whose stated scope runs ahead of its shipped state, and it is easy to miss precisely because, unlike the first three, it never breaks anything loudly.

This is the same pattern I wrote about in the Uber 2022 case study: the gap between what a control should do and what a shortcut actually gets shipped, under time pressure, when nobody is watching that specific line yet. Controls do not fail because people do not understand them. They fail because the secure path was slower than the shortcut, in the moment the shortcut got chosen.

## Key Takeaways

- **Bounding input size is a basic control the happy path never forces you to add.** A normal-sized paste never exercises the missing limit; it takes a deliberately large payload to expose it, and by then it is a live denial-of-service vector.
- **A tool that teaches security should meet its own bar.** The header analyzer finding was not a technical failure, it was a credibility one, and those compound faster than technical ones.
- **A raw vulnerability count is not a risk assessment.** Fourteen advisories sounded severe. Reading each one against actual feature usage cut the real exposure by more than half before a single line of code changed.
- **A name is a promise, and the data has to keep it.** The control mapping the whole project is named for was declared optional in the type and left empty. Making it a required field turned an easy-to-miss content gap into a compile error, so the promise can no longer drift from what ships.

## What I'm Learning From This

Auditing your own work is harder than it looks, not technically, but psychologically. Every one of these findings existed for days or weeks before I wrote them down, because in the moment they did not feel like findings, they felt like reasonable shortcuts I would clean up later. Writing them up in this format, condition, criteria, cause, effect, forced a level of honesty that just fixing the code quietly never would have.

That is probably the most transferable lesson here, more than any individual fix. GRC work is not mainly about knowing the frameworks. It is about being willing to write down, in a document someone else will read, the exact moment your own judgment chose speed over rigor. I did that four times in one project. I expect to find it again in the next one.

---

*Sources: git commit history and `npm audit` output of the `web-vuln-control-mapping` repository, before and after remediation. This is a first-party self-audit, not an independent third-party assessment.*
