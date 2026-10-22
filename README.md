# When a Trusted CA Fails: A Third-Party Risk Post-Mortem of the eMudhra Weak-Key Incident

**A certificate authority's key ceremony went wrong in April 2025. A warning sign in June 2025 was handled as a routine fix. The problem stayed hidden through two audit cycles until an outside researcher found it in October 2026. This post-mortem traces the process gaps behind it, asks which line of defense could have caught it, and sets out what companies that depend on a CA should do now.**

> **Status:** snapshot as of **October 21, 2026, 11:59 PM ET**. The incident is still open. eMudhra has not published its final root cause, and Google and Mozilla are still reviewing eMudhra's remaining CAs. Times are US Eastern (ET), with UTC where the source uses it.
>
> **How to read this:** facts carry a source tag such as **[S1]**. Anything marked **Analysis** is my own inference or opinion, not a confirmed fact.

---

## Executive summary

1. On April 30, 2025, eMudhra's key ceremony produced 8 intermediate CAs whose private keys were trivially guessable. eMudhra traces this to a flawed script step; the HSMs themselves were certified hardware. [S1][S8]
2. They were publicly trusted for about 18 months. In June 2025 a related key mismatch surfaced during a CRL fix and was not escalated. [S1]
3. A researcher reported the flaw on Oct 17, 2026. Browsers blocked the intermediates within hours, eMudhra revoked them, and no misuse has been reported. [S1][S4][S6]
4. Google and Mozilla then removed eMudhra's ECC root, which breaks about 30,800 live certificates, 99.9% of them issued through Internet2/InCommon. [S1][S4]
5. **Analysis:** the main failure was in process, not cryptography. Nobody independently checked the keys, an anomaly went unescalated, and assurance work checked procedure instead of outcomes.

## What this shows

- **Root-cause analysis:** going past "a script bug" to the decisions and controls that let it reach production and stay there.
- **Third-party risk assessment:** treating a certificate authority as a critical supplier with its own failure modes.
- **Control-gap analysis:** mapping each gap to NIST SP 800-53, NIST SP 800-161, ISO 27001:2022, and the CA/Browser Forum and WebTrust requirements.
- **Impact assessment:** separating direct harm from knock-on harm, and confirmed facts from inference.
- **Plain writing:** light on cryptography, with no math.

---

## Timeline

| When (ET) | What happened | Source |
|---|---|---|
| Apr 30, 2025 | Key ceremony creates 24 intermediates (series A, B, C; RSA and ECC; DV, OV, EV, code signing). The 8 ECC keys in series B and C are weak. eMudhra had approved a deviation from its usual ceremony to test a key wrap/unwrap step. | [S1] c.5, c.11 |
| May 14, 2025 | Sectigo's cross-certificates for the affected intermediates take effect (notBefore date). They were made from requests built on the weak keys. | [S1] c.11, c.30 |
| May 2025 | Series A is revoked and replaced by series A-1. eMudhra has not yet said why. | [S1] c.5, c.30 Q11 |
| May 23 – Jun 6, 2025 | CRLs for the 6 affected TLS intermediates fail signature checks (Bug 1970728, filed Jun 5). eMudhra fixes them by re-signing with the second key set, which turns out to be the weak one. Its report blames "incorrect key configuration." | [S1] c.5, c.11, c.30; [S2] |
| Mid-2025 onward | The hierarchy is in browser trust stores. eMudhra says it passed two independent audit cycles. | [S8][S9] |
| Oct 14, 2026 | A CRL for a different root, emSign Root TLS CA - G1, is published with a signature that does not verify against any disclosed key. | [S1] c.7, c.30 Q12 |
| **Oct 17, 2026, 12:27 PM** (16:27 UTC) | A researcher ("Wayne") finds the break while running new tooling across public CA data in the Common CA Database (CCADB). | [S1] c.2 |
| **Oct 17, 2:05 PM** (18:05 UTC) | The report, with proof of key control, goes to Apple, Chrome, Microsoft, Mozilla and eMudhra. | [S1] c.2; [S4] |
| Oct 17, 3:45 PM (19:45 UTC) | Chrome blocks the 6 TLS intermediates by key (CRLSet). No browser update is needed. | [S4] |
| Oct 17, 5:28–5:29 PM (21:28–21:29 UTC) | eMudhra acknowledges and notifies Sectigo. | [S1] |
| Oct 17, 5:48 PM (21:48 UTC) | Mozilla recovers all 8 private keys on its own, about 1 hour 40 minutes after it received the report. | [S1] |
| Oct 17, 6:59 PM (22:59 UTC) | Firefox's OneCRL block is complete. | [S1][S3] |
| **Oct 18, 5:24–5:38 AM** (09:24–09:38 UTC) | eMudhra revokes all 8 intermediates (reason: CA compromise). | [S1] c.1, c.10 |
| Oct 18–19 | Sectigo revokes its 8 cross-certificates. | [S1] c.13; [S11] |
| Oct 19–20 | Apple and Microsoft block the intermediates (as reported by NDTV Profit). Android gets the block through Play Services. | [S4][S13] |
| By Oct 20 | Mozilla and Chrome announce emergency removal of **emSign ECC Root CA - G3**. eMudhra stops issuing from every intermediate under it. | [S1] c.18–21; [S3][S4][S5] |
| Oct 20 | Mozilla and Google security blogs; Firefox 158.0.1 / ESR and NSS 3.131.1 ship; Cloudflare publishes its response; eMudhra posts its incident page. | [S3][S4][S7][S8] |
| Oct 21 | eMudhra press release and NSE/BSE filing (letter EL/SEC/2026-27/62). CERT-In advisory; MeitY forms an investigation team. Cisco Talos publishes. Mozilla posts 12 questions to eMudhra (Bugzilla comment 30). | [S9][S10][S6][S12][S14][S1] |
| **Oct 27, 2026** (morning PDT) | Chrome Root Store removal of the G3 root reaches all users. | [S4] |

**Date discrepancy (flagged, not resolved):** eMudhra's own page says it learned of the issue on **Oct 18** (late evening, India time) and revoked on **Oct 19** [S8]. The public Bugzilla record shows the report going to eMudhra at 18:05 UTC on **Oct 17**, eMudhra acknowledging at 21:28 UTC on Oct 17 (about 3 AM Oct 18, India time), and revocation at 09:24–09:38 UTC on **Oct 18** [S1]. This analysis uses the Bugzilla timestamps.

---

## What happened

### In plain terms

A certificate authority (CA) is a company that browsers trust to vouch for websites. It does this through a chain of trust: a **root** certificate, built into browsers and operating systems, signs **intermediate** certificates, and those sign the certificates websites use. Each link has a secret **private key**. Whoever holds an intermediate's key can create a certificate for *any* website, and browsers will accept it.

A private key is meant to be a random number so large that no one could guess it in the lifetime of the universe. Because of a ceremony error, eight of eMudhra's intermediate keys were tiny: small enough that a laptop, or an AI assistant asked to "solve a CTF," could find them almost at once [S1]. With such a key, an attacker in the right network position could impersonate any website to people whose software still trusted these intermediates.

### Technical detail (light)

- **What was affected:** 8 intermediates. Six were TLS CAs (EM DV, OV and EV TLS CA, G3B and G3C), trusted by Chrome, Mozilla, Apple and Microsoft under *emSign Root TLS CA - G3*. Two were code-signing CAs (EM OV CS CA, G3B and G3C), trusted by Microsoft. Sectigo had also cross-signed all 8 [S1]. *Correction to an early framing: only the 6 TLS intermediates were trusted by all four root programs; the 2 code-signing ones were Microsoft-only [S1][S6].*
- **How weak:** these ECC (P-384) keys should be random 384-bit numbers. All 8 were numbers below 256. No break has been reported in the series A and A-1 ECC keys (Mozilla tried and failed) or in the RSA keys from the same ceremony (eMudhra says they followed a different code path) [S1].
- **eMudhra's account so far:** the HSMs were FIPS 140-2 Level 3 Utimaco units. A ceremony step that tested wrapping and unwrapping keys for export created a second, weak key pair next to each strong one. The labels looked alike, so the weak keys were used for the certificate requests, which the root then signed and Sectigo cross-signed [S1] c.11; [S8]. eMudhra says the final root cause is still being confirmed [S8].
- **Use in production:** eMudhra says 7 of the 8 were never used in production. The one that was (EM DV TLS CA - G3C) had issued no end-entity certificates [S1] c.10; [S8]. Google reports that no TLS certificates chaining to the 6 TLS intermediates were logged to Certificate Transparency before its block [S4].
- **Misuse:** none has been reported. Talos had seen no malicious use as of Oct 21, 10:00 AM ET [S6], and Cloudflare found no active exploitation on its services [S7]. **Analysis:** "not seen" is not proof of absence, because clients that skip revocation checks and CT enforcement would not leave the same trail.

---

## Root cause: process, not just a script bug

**Analysis** throughout this section, built on the facts cited.

1. **A ceremony scripting error, introduced through an approved deviation.** The flaw entered through a one-off change to a high-assurance procedure, the export wrap/unwrap test [S1] c.11. Approved changes to a key ceremony need the same rigor as the ceremony itself: a dry run, peer review of the script, and a check that the outputs match. eMudhra's own words are "human errors in multiple steps of the process" [S1] c.11.
2. **No check of key quality at the moment of generation.** Nothing in the ceremony confirmed that the key behind each certificate request was the intended, strong key. A basic weak-key test on the public keys would have flagged all 8. This is exactly how the researcher found them later, using only public data [S1] c.2. The CA/Browser Forum Baseline Requirements already expect key-parameter quality checking (BR §6.1.6) [S18].
3. **June 2025: an anomaly was fixed quietly instead of escalated.** When the CRLs failed to verify, eMudhra found that the key it expected did not match the published certificate. It then found a second key that did match, re-signed with it, and closed the issue as a configuration error [S1] c.11, c.30 Q8–Q9. Two key pairs for one CA is a key-management integrity event. It was a chance to stop, investigate, and tell Sectigo and the root programs. Mozilla asks in its Q8–Q10 why that did not happen [S1] c.30. **Analysis:** the incident report answered "what broke the CRL?" and stopped there, instead of asking "why does this CA have two keys?"
4. **Ceremony witnesses and auditors did not catch it.** Baseline Requirements §6.1.1.1 calls for CA key generation to be witnessed by a qualified auditor or video-recorded, with a key generation report [S18]. eMudhra says the hierarchy passed two independent audit cycles [S8][S9]. MeitY names BDO International as the WebTrust auditor for eMudhra's global business [S14]. **Analysis:** witnesses and auditors usually confirm that steps were *followed*, not that the *outputs* are sound. Nobody in the room was assigned to check that the key in the certificate was the intended key, or that it was strong.
5. **The fixes promised in 2025 did not hold.** In June 2025 eMudhra committed to automated post-signing validation of CRLs and to replacing manual CRL signing [S1] c.30 Q12(b). On Oct 14, 2026 a root CRL (G1) was again published with an invalid signature [S1] c.7. **Analysis:** remediation was recorded but not verified as effective.

---

## Blast radius: direct vs. indirect

| | Direct | Indirect |
|---|---|---|
| **Certificates** | 8 intermediates and 8 Sectigo cross-certificates revoked [S1] | Removing the G3 root breaks about **30,800** live certificates; 30,791 (99.9%) came from two **Internet2/InCommon** intermediates used across US higher education, and 18 from eMudhra's own CAs [S1] c.30 |
| **Users and relying parties** | Window of exposure for any client trusting the 8 intermediates; browsers closed it within hours [S1][S4] | Software that ignores revocation, such as some VPNs, language runtimes, containers and appliances, needs patches. Go shipped fixes (1.27.3 / 1.26.10) and Cloudflare patched WARP, cloudflared and its CDN [S6][S7] |
| **eMudhra** | Global TLS business, which eMudhra puts at about 1% of revenue [S9][S10] | Brand-wide trust event: Google is "closely investigating" its remaining CAs [S4]; Mozilla's 12 questions [S1]; CERT-In advisory and a MeitY team [S12][S14] |
| **Not affected** | India's CCA-regulated Digital Signature Certificates and eSign, per MeitY and eMudhra [S9][S14] | |

**Market impact (fact line):** eMudhra shares fell 20% to the ₹432 lower circuit on NSE on Oct 21, 2026 [S11][S12]. The board meets on Nov 4 to approve Q2 FY27 results [S15].

---

## Who should have caught it: lines of defense

| Line | What it could have caught | Gap (process, not people) |
|---|---|---|
| **Ceremony operators** (1st line) | That the certificate request was built from the wrong, look-alike key | Similar key labels, and no step to compare the request's public key against the intended key ([S1] c.11) |
| **Internal QA** (1st line) | Weak keys before signing; invalid CRL signatures before publishing | No automated key-quality test or post-signing validation at the time (the latter was promised in June 2025) [S1] |
| **Second line: risk and compliance** | The risk in the approved wrap/unwrap deviation; the meaning of "two keys for one CA" in June 2025 | Change approval without a test plan for its outputs; an incident review that stopped at the symptom (**Analysis**) |
| **Third line: internal audit** | Whether the 2025 root-cause analysis was complete and its fixes worked | No sign of an independent check on remediation, and the Oct 2026 CRL error recurred (**Analysis**, based on [S1] c.7, c.30) |
| **WebTrust external auditors** | Key generation that did not produce the intended keys | Ceremony witnessing and audits check conformance to procedure, not key values (**Analysis**) |
| **Cross-signer (Sectigo)** | Weak public keys in the requests it cross-signed in May 2025 | **Analysis:** the weakness was visible from the public keys alone, so a weak-key check on incoming requests could have flagged it. Sectigo revoked promptly once told [S1] c.6, c.13 |
| **Browser root programs** | Weak keys among their trusted intermediates in public CCADB data; the June 2025 CRL bug as a warning sign | After the report, Chrome confirmed no other trusted keys share this failure mode [S4]. **Analysis:** that kind of sweep can be run continuously |

**Credit where due:** once notified, eMudhra acknowledged within about 3½ hours, revoked within about 16, stopped issuing under G3, and is offering customers certificates from an alternate CA [S1][S8]. The root programs' response took hours, not days.

---

## Controls and framework mapping (one table)

Preventive (**P**) and detective (**D**) controls, each with one sentence on why it applies here.

| Control | Type | Framework anchor | Why it applies here |
|---|---|---|---|
| Automated key-quality checks at generation (key size, valid parameters, known-weak tests such as [badkeys](https://badkeys.info/) [S21]) | P | **NIST SC-12** (key management); **CA/B BR §6.1.6** | A one-line check on the 8 public keys would have failed them before the root ever signed. |
| Approved cryptography actually delivered at full strength | P | **NIST SC-13** (cryptographic protection) | P-384 was the right algorithm, but a key with almost no randomness gives none of its protection. |
| Ceremony scripts tested before use (dry run, peer review, expected-output tests) | P | **NIST SA-11** (developer testing) | The flaw sat in a one-off script that was never tested against its outputs. |
| Change control for ceremony deviations | P | **NIST CM-3** (configuration change control) | The wrap/unwrap test was approved, but there was no plan to verify what it produced. |
| Dual control with independent verification (a second person or tool confirms that the request's key matches the intended key) | P | **NIST AC-5** (separation of duties); **CA/B BR §6.1.1.1** | Look-alike labels slipped through because no one independently matched the key to the request. |
| Post-ceremony checklist and key-generation report covering outputs, not just steps | P/D | **CA/B BR §6.1.1.1**; **WebTrust for CAs** key generation criteria [S19] | The ceremony was witnessed and audited, but its results were never checked. |
| Linting of issued certificates *and* the CA's own keys, continuously | D | **NIST CA-7** (continuous monitoring) | The researcher found the flaw from public data, so the CA could have too. |
| CRL signature validation monitoring | D | **NIST SI-4** (system monitoring); **CA/B BR §4.9.7** | Invalid CRLs in May–June 2025 and again on Oct 14, 2026 were early warnings. |
| Mandatory escalation of key anomalies (any key mismatch counts as a security incident) | D | **NIST IR-4** (incident handling) | The June 2025 mismatch was fixed quietly instead of escalated. |
| Timely, complete incident disclosure | D | **NIST IR-6** (incident reporting); **CCADB incident reporting** [S22]; **CA/B BR §4.9.1.2** (revoke a compromised intermediate within 7 days) | Revocation was fast in 2026, but the 2025 report left out the two-key fact (Mozilla Q9). |
| Supply chain risk plan that names CAs as critical suppliers | P | **NIST SR-2 / SR-3** (supply chain risk management) | For relying parties, a CA is a single upstream dependency for every encrypted connection. |
| Supplier assessment of CAs (audit reports, incident history, root-program standing) | D | **NIST SR-6** (supplier assessments) | Bug 1970728 was public 16 months before the distrust. |
| Enterprise C-SCRM program covering trust infrastructure | P | **NIST SP 800-161r1** (C-SCRM) [S20] | It puts PKI providers into the same tiered supplier risk process as any critical vendor. |
| Supplier relationship and ICT supply chain controls | P/D | **ISO 27001:2022 Annex A 5.19–5.22** | 5.19–5.20 set CA expectations in contracts, 5.21 covers the ICT supply chain, and 5.22 requires watching for changes such as a distrust. |
| Certificate inventory | D | **NIST CM-8** (component inventory) | You can't replace before Oct 27 what you can't find. |

*Section numbers refer to the current CA/B Forum Baseline Requirements [S18]; check against the version in force for a given audit period.*

---

## Game-theory lens

**Analysis.** This is a plain-language way to look at the incentives, not a formal model.

**The ceremony team.** A key ceremony is expensive. It means booked rooms, HSMs, witnesses and auditors, so the pressure is to get it *done*. Finding a problem means doing it again, and nobody is rewarded for that. In June 2025 the team faced a choice. It could quietly re-sign the CRLs and close the ticket, which was cheap now. Or it could escalate "we have two keys for one CA," which was costly now: revocations, cross-signer conversations, perhaps a new ceremony. The quiet option won. That was a rational move on a short horizon and a very expensive one on a long one.

**Attackers.** Recovering a key below 256 costs almost nothing, and the payoff, a key that can impersonate any website, is large. On paper that is an irresistible offer. Yet no misuse has been reported, likely because using a forged certificate also needs network position and would show up in Certificate Transparency against CT-enforcing clients [S4]. Cheap to find does not mean cheap to profit from, but it does mean the window cannot stay open.

**Google and Mozilla as rule-setters.** Root programs play a *repeated game* with every CA. Their threat to distrust a CA only works if they sometimes carry it out. Distrusting the G3 root, even though it hurts innocent InCommon users, keeps that threat credible for every other CA. *Comparison point:* Entrust in 2024 was distrusted for a long pattern of compliance failures, not a single compromise [S17]. Different facts, same lesson. Root programs act on patterns and on how a CA handles problems.

**Defenders.** Prevention here was cheap: one automated weak-key check, and one rule that any key mismatch gets escalated. Cleanup is expensive: about 30,800 certificates to replace, patches across runtimes and VPNs, regulatory attention, and a 20% one-day share drop [S1][S6][S11].

**Takeaway:** controls work when they change incentives. Independent verification removes the option of "nobody will check," and rewarding escalation makes "stop and report" the move people actually choose.

---

## Third-party and supply-chain risk lessons

**Analysis** throughout this section.

- **Your CA is a supplier, and usually a single point of failure.** InCommon's members did nothing wrong. They inherited a root distrust from a supplier two levels up and now have days to re-issue.
- **Keep a backup CA ready before you need one.** That means an account opened, validation done, and automation tested.
- **Keep a certificate inventory.** Track each certificate's issuer, chain, expiry, and owner.
- **Practise crypto agility.** Automated issuance (ACME, ideally with renewal information support) turns a CA change into a configuration change. eMudhra's own migration from the ECC G3 root to the RSA G1 root may need client-side changes [S1] c.30 Q5, so rehearse the switch.
- **Watch root-program decisions as a vendor-risk signal.** Comparison point: in the 2024 Entrust case, Chrome announced distrust months before it took effect [S17]. Here the root removal came within days, so the notice period cannot be relied on.

---

## What a downstream company should do Monday morning

1. **Find your exposure today.** Search Certificate Transparency for your domains, and scan your endpoints, for any chain ending in **emSign ECC Root CA - G3**, including InCommon *OVG3C* / *DVG3C* intermediates [S1] c.30.
2. **Replace before Oct 27.** Get certificates from a different hierarchy that is in the Chrome Root Store [S4]. Don't wait for an ACME renewal window to arrive on its own.
3. **Patch non-browser trust.** Update OS CA bundles, language runtimes (for example Go 1.27.3 / 1.26.10 [S6]), container base images, VPN clients and appliances. **Never "fix" errors by turning off certificate verification** [S7].
4. **Set up a backup CA** and test a failover on one low-risk service.
5. **Put CAs in your vendor-risk register** as critical suppliers, with an owner and a review cycle.
6. **Ask your CA these vendor-risk questions:**
   - How do you verify key quality and the key-to-certificate binding at every key ceremony?
   - Who independently verifies ceremony *outputs*, not just steps?
   - Show us your last two WebTrust or ETSI audit reports and any qualified findings.
   - What incidents have you filed on Bugzilla or CCADB in the past 24 months, and were their fixes verified?
   - What is your escalation rule when keys, CRLs or OCSP responses don't verify?
   - If one of your roots is distrusted, how fast can you re-issue our certificates from another hierarchy, and from which?
   - Do you support ACME renewal information (ARI) so that mass replacement is automatic?
7. **Monitor root-program decisions** (CCADB announcements, the Mozilla dev-security-policy list [S5], and Chrome's blog) as part of continuous monitoring.

---

## Lessons learned

### For certificate authorities

- **Validate key quality at generation.** Check every new key, and the key in every request, against known-weak tests before the root signs. The flaw here was detectable from public data alone.
- **Escalate mismatches; don't patch around them.** The June 2025 discovery of two keys for one CA was the moment to stop, revoke and disclose. A quiet re-sign turned a contained error into an 18-month exposure.
- **Verify ceremonies independently.** A witness who confirms the steps is not enough. Someone, or some tool, independent of the operators must confirm that the outputs are the intended, strong keys.

### For GRC practitioners

- **"No misuse found" is not "no risk."** The exposure lasted 18 months, and non-browser clients that skip revocation checks leave little trace.
- **A business line can be small in revenue and large in trust risk.** Global TLS is about 1% of eMudhra's revenue [S9], but it triggered a brand-wide trust event: regulator attention, a review of all remaining CAs, and a 20% share drop [S4][S11][S14].
- **Repeated small compliance slips are a leading indicator.** Invalid CRLs in 2025, and again in October 2026, are the kind of pattern root programs weigh, as the Entrust comparison shows [S17]. Track them as key risk indicators, not one-off tickets.

### For organizations relying on third-party CAs

- Keep a **certificate inventory** and know which root each certificate depends on.
- Hold a ready **backup CA** and practise **crypto agility** through automation.
- Ask the **vendor-risk questions** above at onboarding and every year.
- **Watch root-program decisions.** Google and Mozilla are still investigating eMudhra's remaining CAs [S1][S4].

---

## What to watch next

- eMudhra's answers to Mozilla's 12 questions, especially why the 2025 report left out the two-key fact (Q8–Q10), why series A was replaced (Q11), and the Oct 14 G1 CRL (Q12) [S1] c.30.
- eMudhra's final root-cause report and its promised re-verification of every key ceremony [S1] c.11; [S8].
- Chrome's decision on eMudhra's remaining CAs [S4], and whether Mozilla keeps trusting *emSign Root CA - G1* [S1] c.18.
- The open HSM question in Bugzilla comment 31: were the weak keys generated inside the HSM, or created outside and loaded in? The answer affects other CAs using the same hardware [S1] c.31.
- Migration progress for the InCommon certificates before **Oct 27** [S1][S4].
- Findings from CERT-In and MeitY [S12][S14], and the Q2 FY27 results on Nov 4 [S15].

---

## Confirmed vs. unverified

- **Confirmed (primary sources):** the timestamps in Bugzilla 2083449; the 8 intermediates and their split into 6 TLS and 2 code-signing; keys below 256; the G3 root removal and Chrome's Oct 27 date; the ~30,800 / 99.9% InCommon figures; the 12 questions; and the filing letter number and the ~1% revenue figure.
- **eMudhra's account, not yet independently confirmed:** the wrap/unwrap mechanism, "7 of 8 never in production," the claim that the RSA keys are unaffected, and that no misuse occurred.
- **Secondary sources only:** Apple and Microsoft block dates (NDTV Profit [S13]); the ₹432 price (Business Standard, CNBC-TV18 [S11][S12]).
- **Discrepancy:** eMudhra dates its notice and revocation one day later than the Bugzilla record (see Timeline).

---

## Sources

| # | Source |
|---|---|
| S1 | [Mozilla Bugzilla 2083449: eMudhra: Private key compromise of Intermediate Certificates](https://bugzilla.mozilla.org/show_bug.cgi?id=2083449) (comments cited as c.N) |
| S2 | [Mozilla Bugzilla 1970728: eMudhra invalid CRL signatures (2025)](https://bugzilla.mozilla.org/show_bug.cgi?id=1970728), as summarized in S1 c.5 and c.30 |
| S3 | [Mozilla Security Blog, Oct 20, 2026: Protecting Firefox users following the compromise of eMudhra intermediate certificates](https://blog.mozilla.org/security/2026/10/20/protecting-firefox-users-following-the-compromise-of-emudhra-intermediate-certificates/) |
| S4 | [Google, Oct 20, 2026: Protecting Chrome and Android Users from Compromised eMudhra Intermediate CA Certificates](https://blog.google/security/protecting-chrome-and-android-users-from-compromised-emudhra-intermediate-ca-certificates/) |
| S5 | [Mozilla dev-security-policy (MDSP) discussion thread](https://groups.google.com/a/mozilla.org/g/dev-security-policy/c/0R3XLJL6FuE/m/VAgSvefvAAAJ) |
| S6 | [Cisco Talos, Oct 21, 2026: Compromised eMudhra intermediate certificates](https://blog.talosintelligence.com/compromised-intermediate-certificates/) |
| S7 | [Cloudflare Blog, Oct 20, 2026: 8 eMudhra certificates are compromised. Here's what you need to know](https://blog.cloudflare.com/emudhra-certificate-compromise/) |
| S8 | [eMudhra: Statement on Recent Certificate Security Incident](https://emudhra.com/en/post/certificate-security-incident) (first published Oct 20, updated Oct 21) |
| S9 | [eMudhra press release, Oct 21, 2026: Statement on distrust of select Global TLS Intermediate and one Root CA](https://emudhra.com/en/press-release/emudhra-statement-distrust-global-tls-intermediate-root-ca) |
| S10 | [NSE/BSE filing, letter EL/SEC/2026-27/62, Oct 21, 2026](https://www.bseindia.com/xml-data/corpfiling/AttachLive/6be0c423-b335-494b-86f3-54c561c3a5f9.pdf) |
| S11 | [Business Standard, Oct 21, 2026: eMudhra tanks 20%](https://www.business-standard.com/markets/news/emudhra-tanks-20-why-it-enabled-services-stock-sank-in-wednesday-s-trade-126102100353_1.html) |
| S12 | CNBC-TV18, Oct 21, 2026: [stock fall](https://www.cnbctv18.com/market/emudhra-share-price-here-is-why-the-stock-fell-20-percent-cybersecurity-incident-20015881.htm) · [CERT-In advisory and MeitY team](https://www.cnbctv18.com/technology/cert-in-urgent-advisory-emudhra-private-key-compromise-20016169.htm) |
| S13 | [NDTV Profit, Oct 21, 2026: eMudhra shares crash 20%](https://www.ndtvprofit.com/markets/emudhra-shares-crash-20-after-browsers-move-to-distrust-some-of-its-global-tls-certificates-12213738) |
| S14 | [News On AIR (Government of India): MeitY says impact very limited](https://newsonair.gov.in/meity-says-impact-of-emudhra-security-incident-on-indian-websites-apps-very-limited/) |
| S15 | [eMudhra Q2 FY27 earnings call intimation (EL/SEC/2026-27/61)](https://emudhra.com/investor-assets/investors/esg-profile/fy2027/q2-earning-call-intimation.pdf); [board meeting Nov 4 (Economic Times announcements)](https://economictimes.indiatimes.com/emudhra-ltd/stocksupdate/companyid-2043301.cms) |
| S16 | [oss-security, Oct 20, 2026: eMudhra Intermediates Compromise](https://openwall.com/lists/oss-security/2026/10/20/1) |
| S17 | [Google Security Blog, Jun 2024: Sustaining Digital Certificate Security (Entrust distrust)](https://security.googleblog.com/2024/06/sustaining-digital-certificate-security.html) |
| S18 | [CA/Browser Forum Baseline Requirements](https://cabforum.org/working-groups/server/baseline-requirements/documents/) |
| S19 | [WebTrust Principles and Criteria for CAs (CPA Canada)](https://www.cpacanada.ca/en/business-and-accounting-resources/audit-and-assurance/overview-of-webtrust-services/principles-and-criteria) |
| S20 | NIST [SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final) · [SP 800-161 Rev. 1 (C-SCRM)](https://csrc.nist.gov/pubs/sp/800/161/r1/upd1/final) · ISO/IEC 27001:2022 Annex A |
| S21 | [badkeys: a tool and service to find known-weak public keys](https://badkeys.info/) |
| S22 | [CCADB incident reporting guidelines](https://www.ccadb.org/cas/incident-report) |

---

## About this analysis

**Analysis by Rashon Alsberry.** I work on supply chain and third-party risk, using AI automation to do it at scale. To me this incident is a supplier-risk story: a dependency most companies never list in their vendor register failed two levels upstream, and the organizations hurt most were customers of a customer. The questions I would ask any critical supplier apply directly to CAs. Who checks the outputs? What happens when something doesn't match? How fast can I switch?

*AI-assisted research and analysis:* I used AI tools to gather and cross-check sources and to draft. I reviewed the facts against the primary sources listed, and the judgments marked **Analysis** are mine. This is not legal, investment, or audit advice. Corrections are welcome through an issue.

Related work: [grc-control-mapper](https://github.com/Starboy1968/grc-control-mapper) (policy-to-NIST 800-53 mapping) · [two-suppliers-one-factory](https://github.com/Starboy1968/two-suppliers-one-factory) (hidden tier-2 single points of failure) · [supply-web-audit](https://github.com/Starboy1968/supply-web-audit)

Writing licensed [CC BY 4.0](LICENSE).

*Rashon Alsberry · Supply chain risk + AI automation*
