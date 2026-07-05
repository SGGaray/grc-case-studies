# When the Human Becomes the Attack Surface
### The Uber 2022 Breach as a Failure of Behavioral System Design

**Case Study | Sebastian Garay | GRC, Human Risk & Shadow AI**

> This is an independent analysis based on public reporting, written as part of my learning path in GRC. I have no inside knowledge of Uber's systems or investigation. The analysis, and the opinions, are mine.

## Executive Summary

In September 2022, an attacker gained broad access to Uber's internal systems without exploiting a single software vulnerability. The entry point was a person: an external contractor worn down by repeated MFA push notifications and a convincing message from a fake "IT support" contact.

This breach matters because it shows how technically sound controls fail when they ignore how humans actually behave under fatigue, pressure, and ambiguity. This analysis focuses on the Human Risk dimension: why the control design made the wrong decision easy, and what that teaches us about a newer version of the same problem, Shadow AI, where users bypass controls not out of malice but because the system incentivizes it.

## Incident Overview

Simplified timeline, September 2022:

1. An attacker, later linked to the Lapsus$ group, obtains the VPN credentials of an external Uber contractor. Public reporting suggests they were purchased on a dark web marketplace after a malware infection on the contractor's personal device.
2. The credentials alone are not enough. Uber uses push-based MFA. The attacker triggers repeated login attempts, flooding the contractor with approval requests.
3. After roughly an hour, the attacker contacts the contractor on WhatsApp, posing as Uber IT support, and instructs them to accept the prompt to make the notifications stop.
4. The contractor approves. The attacker enrolls their own device for MFA and gains persistent VPN access.
5. Inside the network, the attacker scans internal shares and finds a PowerShell script containing hardcoded admin credentials for Uber's privileged access management (PAM) platform.
6. With PAM-level access, the attacker reaches AWS, Google Workspace, Slack, the HackerOne bug bounty program, and internal dashboards.
7. The attacker announces the breach in Uber's own Slack. Some employees initially think it is a joke.

No zero-days. No advanced malware inside Uber's perimeter. **The critical exploit was a tired human plus a script with secrets in plain text.**

## Attack Breakdown

**Step 1. Credential acquisition.** Contractor credentials stolen outside Uber's perimeter, through the contractor's personal device. Uber never saw this stage.

**Step 2. MFA fatigue.** Repeated push notifications convert a security control into a source of stress. Each prompt looks identical to a legitimate one.

**Step 3. Social engineering.** A WhatsApp message from "IT" reframes the situation: approving the prompt is no longer risky, it is presented as the solution. Authority plus relief.

**Step 4. Persistence.** The attacker registers their own device for MFA. The control that failed once now works for the attacker.

**Step 5. Privilege escalation.** Hardcoded PAM admin credentials found on a network share. One file converts contractor-level access into near-total access.

**Step 6. Broad compromise and self-disclosure.** The attacker moves across cloud, communication, and security tooling, then publicly announces the breach.

## Key Failure Domains

### 1. Identity & Access Management

**What failed:** Push-based MFA with no rate limiting and no number matching. Self-service MFA device enrollment after a single approval.

**Why it failed:** The control assumed every "approve" reflects an informed decision. It had no mechanism to distinguish a deliberate approval from a worn-down one.

**Insight:** **An MFA prompt is not a security decision. It is a security decision delegated to a human under unknown conditions.** Design has to account for the worst realistic condition, not the ideal one.

### 2. Human Risk Layer

**What failed:** A contractor, likely with less security context than an employee, became the single point of failure for network entry.

**Why it failed:** Third-party users often receive less training, less context, and identical access paths. The attacker chose the weakest human link, not the weakest system.

**Insight:** **Your human attack surface includes everyone with credentials, not everyone on payroll.**

### 3. Control Design

**What failed:** Hardcoded admin credentials in a script on an accessible network share, and a flat enough internal environment for a contractor account to reach it.

**Why it failed:** Secrets management existed as a tool, a PAM platform, but not as a practice. A useful way to interpret this is that the convenience of a hardcoded credential beat the policy against it, which is itself a human risk pattern.

**Insight:** **When a secure process is slower than an insecure shortcut, the shortcut wins over time.** Controls compete against convenience whether we acknowledge it or not.

### 4. Monitoring & Detection

**What failed:** Dozens of failed MFA pushes in a short window did not trigger an automated response or an out-of-band verification with the user.

**Why it failed:** The signal existed but was not treated as an indicator of attack. MFA fatigue was already a known technique by 2022, used against Cisco and others months earlier.

**Insight:** **Repeated MFA denials are not noise. They are one of the clearest real-time signals that credentials are already compromised.**

## Mapping to GRC Frameworks

**NIST CSF 2.0**

- **PR.AA, Identity Management, Authentication and Access Control:** weak authentication assurance (push MFA without phishing resistance), excessive standing privileges, insecure credential storage.
- **PR.AT, Awareness and Training:** contractor population undertrained for social engineering targeting MFA specifically.
- **DE.CM / DE.AE, Continuous Monitoring, Adverse Event Analysis:** failure to correlate mass MFA denials as an attack pattern.
- **GV.SC, Cybersecurity Supply Chain Risk Management:** third-party access treated with the same trust assumptions as internal access.

**ISO/IEC 27001:2022 (Annex A)**

- **A.5.17, Authentication information:** hardcoded admin credentials in scripts violate secure handling of authentication data.
- **A.8.5, Secure authentication:** MFA implementation lacked resistance to prompt bombing.
- **A.8.2, Privileged access rights:** PAM admin credentials reachable from a low-privilege account defeats the purpose of privileged access management.
- **A.6.3, Awareness, education and training:** coverage gap for third-party users.

## The Human Factor

This is where my background in Social Psychology shapes how I read the incident.

**Habituation:** users approve MFA prompts reflexively because they almost always follow their own login. The attacker triggered the reflex out of context. **Fatigue:** an hour of prompts made "approve" the stop button for the discomfort. The attacker weaponized relief. **Authority:** the fake IT message matched the contractor's mental model exactly, on a channel Uber could not see. **Design:** the final security decision landed on the person with the least context and no support path.

**"Human error" is usually the end of a bad explanation, not the start of a good one.** The cause here is a control designed for an alert, informed, calm user, deployed to humans who are frequently none of the three.

## The Shadow AI Parallel

The engineer who hardcoded credentials was removing friction, not attacking. The employee who pastes confidential data into an unapproved AI tool is doing the same.

**Both risks live in the gap between what policy assumes users do and what users actually do.** Organizations deploying AI governance today are repeating the MFA mistake if the secure path is the painful path. **Behavior follows incentives, not policy documents.**

## Prevention, Realistic Version

**Technical:** number matching or passkeys, lockout on repeated MFA denials, secret scanning, network segmentation.

**Human-centered:** scenario-specific training ("unexpected prompt equals report it"), a reporting path easier than approving, contractors included.

**GRC:** third-party access brought into risk scope, behavioral metrics (denial rates, time-to-report) tracked instead of only training completion rates.

## Key Takeaways

- **MFA fatigue outsources a security decision to a human at their worst moment.**
- Third parties are part of your human attack surface, not an exception to it.
- **Controls compete against convenience,** and time favors the shortcut.
- Repeated MFA denials are signal, not noise.
- **Shadow AI is the same bypass pattern on a new surface,** moving faster than governance.

## What I'm Learning From This

Every "human error" hides a design decision.

Mapping the breach to NIST CSF and ISO 27001 was the easy part. The frameworks tell you which controls were missing or weak. They do not tell you why an engineer hardcoded credentials or why a tired person taps "approve." That second layer is where my Social Psychology training turns out to be directly useful, and where I want to keep going deeper.

Shadow AI is not a new problem, it is a familiar problem wearing new clothes. I started studying Shadow AI as a separate topic. This case convinced me it belongs to the same family as MFA fatigue and hardcoded credentials: friction-driven bypass. That reframing matters, because it means the mitigation playbook can borrow from decades of human risk lessons instead of starting from zero.

Good GRC measures behavior, not just documents. Training completion rates and published policies did not stop this breach. Denial rates, reporting rates, and time-to-report might have. I am learning to ask, for any control: what does this assume about the human using it, and how would we know if that assumption breaks?

Human Risk Management, informed by how people actually respond to friction, fatigue, and authority, is not a soft complement to technical GRC. It is the discipline that predicts which controls will hold and which will be routed around. That prediction is where I want to work.

---

*Sources: public reporting on the September 2022 Uber security incident, including Uber's security update, coverage of the attacker's methods, and subsequent analysis of MFA fatigue techniques. This is an independent learning analysis, not an internal investigation.*
