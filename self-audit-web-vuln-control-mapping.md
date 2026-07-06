# Auditing My Own Tool
### A Self-Audit of web-vuln-control-mapping, Five Real Findings and What They Taught Me About Risk

**Case Study | Sebastian Garay | GRC, Human Risk & Shadow AI**

> This is a genuine self-audit, not a demonstration exercise. Every finding below was discovered while actively building and maintaining this project, and every fix was applied to the real, live repository. Dates and commit messages are accurate. Nothing here is staged.

## Why Audit Your Own Code

Most portfolio case studies analyze someone else's breach. That is useful, but it carries a risk: it is easy to sound wise about failures you did not live through. This one is different. `web-vuln-control-mapping` is my own tool, a small Next.js app that maps common web vulnerabilities to risk and governance controls. Over the course of building and maintaining it, five real issues surfaced. I am documenting them the way I would document findings in a client engagement: condition, criteria, cause, effect, and recommendation, because the format matters more when the subject is your own work. It is easier to be honest about someone else's mistakes than your own.

## Scope and Method

The audit covers the application code, its build and deployment pipeline, and its dependency posture, as they existed across several weeks of active development. Findings were discovered through a mix of manual code review, dependency scanning (`npm audit`), and live production incidents (a build that silently failed, a case study that would not render). I did not go looking for problems to pad this report. Every finding here caused a real, observable failure first.

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

## Finding 3: A Hardcoded File List Silently Broke Production Deploys

**Condition:** The build script copied static assets into the deploy folder using an explicit, hardcoded list of filenames: `cp index.html cerro-sosneado.jpg og-image.png profile.jpg dist/`. When the images were later converted from JPG to WebP (for a 65 to 92 percent file size reduction) and the old JPG files were removed, this command started failing.

**Criteria:** Build scripts should not encode specific filenames as an implicit contract. NIST CSF's Govern function (GV.SC, Supply Chain and dependency practices, extended here to internal tooling) implies that changing an artifact should not require remembering every place its old name is hardcoded elsewhere.

**Cause:** The original script was written once, worked, and was never revisited when the asset set changed. This is the same failure mode a security control has when it is written for one specific condition and never re-validated against a changed environment.

**Effect:** The build step failed outright (`cp: cannot stat 'cerro-sosneado.jpg': No such file or directory`), which meant the GitHub Action exited with an error and the last successful deploy stayed live. For a period, the production site was silently serving a stale build while every push after the image swap failed invisibly unless someone checked the Actions tab.

**Recommendation and remediation:** Replaced the hardcoded filenames with glob patterns (`cp index.html *.png *.webp dist/`), so the copy step now works with whatever image files actually exist, without needing to be edited again when assets change. Fixed in commit `fix(build): copy static assets by glob instead of hardcoded names, automate cache-busting via commit SHA`.

## Finding 4: A Manual Cache-Busting Step Depended on Human Memory

**Condition:** To force browsers and the GitHub Pages CDN to fetch updated JavaScript after a deploy, script tags in `index.html` carried a manually incremented query string (`sections.js?v=2`, then `?v=3`). Each new deploy required remembering to bump that number by hand.

**Criteria:** A control that depends on a human remembering a manual step, every single time, with no reminder and no consequence for skipping it, is not a reliable control. This is the same principle behind why manual patch management fails at scale in any organization.

**Cause:** The fix was built reactively, in the middle of debugging a stale-cache incident, as the fastest way to unblock a specific deploy. It solved that one incident but did not remove the underlying dependency on memory.

**Effect:** More than once, a real code change (including this project's own Uber breach case study reader) appeared broken to users because the version number was not bumped, even though the underlying code was correct. The failure was indistinguishable from a real bug without checking the build pipeline directly.

**Recommendation and remediation:** Added a build step that generates the cache-busting value automatically from the git commit hash (`sed -i -E "s/(src=\"(...)\.js)\"/\1?v=${SHA}\"/g" dist/index.html`), applied to every relevant script tag on every deploy. The control no longer depends on anyone remembering anything. Fixed in commit `fix(build): copy static assets by glob instead of hardcoded names, automate cache-busting via commit SHA`.

## Finding 5: Framework Dependency Carried Fourteen Unpatched Advisories

**Condition:** The project ran Next.js 14.2.35, the last version in its release line, against which `npm audit` reported fourteen separate published advisories (denial of service, cache poisoning, request smuggling, cross-site scripting, and server-side request forgery variants), aggregated under a single high-severity entry.

**Criteria:** Dependencies with known, disclosed vulnerabilities and no available patch in their current line should be triaged and upgraded on a defined cycle, not left indefinitely because the current version still runs. This maps to NIST CSF's Identify function (ID.RA, Risk Assessment) applied to third-party components.

**Cause:** The 14.x line had reached its final release; every fix for these advisories only existed in the next major version. Upgrading a major framework version carries its own risk of breaking changes, which is why it had been deferred.

**Effect:** Before triage, the raw count (fourteen advisories, one rated high) looked alarming. After actually reading each one against the codebase, eight of the fourteen turned out to be inapplicable: they required features this project does not use at all (the image optimizer, middleware, i18n routing, rewrites, WebSocket upgrades, CSP nonces). The real residual exposure was narrower than the count suggested, but not zero.

**Recommendation and remediation:** Upgraded to Next.js 15 and React 19 together (avoiding a two-major-version jump straight to 16), using a dedicated branch, a full local build and test pass, and a manual smoke test before merging. Verified the fix with a follow-up `npm audit`, which dropped from fourteen high-severity advisories to a single unrelated moderate one (an internal PostCSS dependency, not exploitable in this project's build process). Fixed in commit `chore: upgrade to Next.js 15, React 19, and lucide-react latest`.

## What These Five Findings Have in Common

Every single one of these is a friction-driven shortcut, not a knowledge gap. I knew input should be bounded; I just had not gotten to it yet. I knew hardcoding filenames was fragile; it was faster to write it that way the first time. I knew a manual version bump was not a real fix; it unblocked the incident I was debugging at 11pm faster than building the automated version would have.

This is the same pattern I wrote about in the Uber 2022 case study: the gap between what a control should do and what a shortcut actually gets shipped, under time pressure, when nobody is watching that specific line yet. The MFA-fatigued contractor and the hardcoded `cp` command are the same failure mode wearing different clothes. Controls do not fail because people do not understand them. They fail because the secure path was slower than the shortcut, in the moment the shortcut got chosen.

## Key Takeaways

- **A tool that teaches security should meet its own bar.** The header analyzer finding was not a technical failure, it was a credibility one, and those compound faster than technical ones.
- **Hardcoded values are a governance problem disguised as a coding style choice.** Both the filename list and the manual cache version were the same underlying pattern: a fact about the world encoded once, never re-validated.
- **A raw vulnerability count is not a risk assessment.** Fourteen advisories sounded severe. Reading each one against actual feature usage cut the real exposure by more than half before a single line of code changed.
- **Fixing something reactively during an incident is not the same as fixing it properly.** The first cache-busting fix (`?v=2`) solved the immediate outage. It took a second pass, done without time pressure, to remove the human-memory dependency entirely.

## What I'm Learning From This

Auditing your own work is harder than it looks, not technically, but psychologically. Every one of these findings existed for days or weeks before I wrote them down, because in the moment they did not feel like findings, they felt like reasonable shortcuts I would clean up later. Writing them up in this format, condition, criteria, cause, effect, forced a level of honesty that just fixing the code quietly never would have.

That is probably the most transferable lesson here, more than any individual fix. GRC work is not mainly about knowing the frameworks. It is about being willing to write down, in a document someone else will read, the exact moment your own judgment chose speed over rigor. I did that five times in one project. I expect to find it again in the next one.

---

*Sources: git commit history and GitHub Actions logs of the `web-vuln-control-mapping` repository, `npm audit` output before and after remediation. This is a first-party self-audit, not an independent third-party assessment.*
