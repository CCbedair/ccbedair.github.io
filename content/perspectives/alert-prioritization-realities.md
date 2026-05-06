---
title: "The Realities of Alert Prioritization"
date: 2026-04-04
draft: false
section: "Detection & Response"
type: "perspective"
tags: ["Detection engineering", "Alert tuning"]
description: "Why textbook alert prioritization frameworks fail in production, and what actually works when you're a team of three staring at 200 daily alerts"
---

Every security conference talk on alert prioritization follows the same script: build a severity matrix, weight it by asset criticality, feed in threat intelligence, automate triage. Clean slides, clean math, clean framework.

Then you get back to your desk, open your SIEM, and stare at 200 alerts that don't fit the framework. Your asset inventory is incomplete. Your threat intel is yesterday's news. Half the alerts are from staging environments nobody tagged properly. The matrix gives you a number. The number doesn't help.

I've built alert prioritization systems in production twice. Both times, the textbook approach broke down in predictable ways. Here's where — and what I do instead.

## Asset Criticality Is Aspirational

The prioritization matrix assumes every resource in your environment has a criticality label: crown jewel, standard, or low. In practice, most organizations do not maintain asset inventories at that level of fidelity, and even when they do, the data is often outdated by the time an alert fires.

The usual fallback is a simple heuristic: “Does this alert involve customer data? High. Authentication system? High. Production? High. Everything else? Low.” But that also breaks down quickly. A service may be tagged as staging and still communicate with production systems. A background job may not store customer data, but it may handle tokens or credentials that can reach it. Clean categories assume clean infrastructure, and clean infrastructure is rare.

A more practical approach is to start with two working questions:

- **What is the likely impact if this alert is real, based on what I know right now?**  
  This is a provisional blast-radius estimate, not a perfect answer—what systems, data, or permissions could be affected given the limited context available.

- **How much do I trust this detection rule, given its history and the evidence in front of me?**  
  This is a confidence estimate that can be refined over time by looking at prior outcomes, rule quality, and surrounding telemetry. A noisy rule should generally carry less weight than a stable one, but that judgment should be revisited as investigations add context.

A practical example: imagine you see two alerts arrive at the same time. One is a new SSH key added to AWS infrastructure; the other is a Google Drive file-sharing permission changed from internal-only to external access. For the SSH key, the real question is not “which environment label does this belong to?” but “what systems can this key reach, and how privileged are they?” For the Drive file, the key question is “what is actually in that file, and how sensitive is it?” The SSH alert might be noise if it links to a non-privileged host; the Drive alert might already be a direct exposure if it points to contracts, source code, or whistleblower material.

In that scenario, you don’t try to fully evaluate both alerts—you pick the one with the highest *expected impact*, even under uncertainty.

A simple way to do that in practice:

- **Prioritize alerts that imply confirmed data exposure over those that imply potential access.**
- **Prioritize irreversible outcomes over reversible ones.**
- **Prioritize signals where the worst-case impact could already be happening, not just enabled.**

Applied to the example:

- The Google Drive alert may represent **actual data exposure already in progress** (a file is now accessible externally).
- The SSH key alert represents **potential access** that may or may not be usable, depending on scope and privilege.

Given that, you would investigate the Drive alert first, because:
- The harm (data exposure) may already have occurred
- The blast radius could be immediately material
- The window to contain it (e.g. revoke access) is time-sensitive

Then you would return to the SSH alert to determine whether it represents meaningful access or benign activity.

This is not about certainty—it’s about acting on the alert where delay is most likely to make things worse.

Build prioritization around blast radius and detection confidence, but treat both as inputs that improve over time, not as static truths. Asset criticality labels are useful enrichment when they exist—they are not a prerequisite.

## Your Log Pipeline Is Lying to You

Before you trust any prioritization output, you need to answer a more basic question: is the data actually there?

Log ingestion pipelines break silently. Somebody spins up a new cloud account without enabling audit logging or A SIEM ingestion quota gets hit and starts dropping events. To be fair, most modern SIEMs have some capability to alert on ingestion failures — Datadog has log pipeline monitors, Google SecOps tracks ingestion health metrics, and CrowdStrike surfaces data connector status. But these built-in checks cover the happy path. They tell you "the pipeline is running." They don't tell you "a new cloud account was created last Tuesday and nobody onboarded its logs."

The gap is between what your SIEM monitors automatically and what your *environment* should be sending. That gap is where blind spots live.

A log source health practice should be your first investment before building any detection logic:

- **Ingestion volume per source against a rolling 24-hour baseline.** A sudden drop means a pipeline broke or a source stopped sending.
- **Ingestion latency per source.** If there's a 30-minute delay between event and SIEM availability, your "real-time" detection has a 30-minute blind spot.
- **Coverage map: expected sources versus active sources.** Every cloud account, every SaaS tool, every log type you expect — compared against what's actually arriving.
- **Last-seen timestamp per source.** The simplest check. If a source that normally sends every minute hasn't sent anything in an hour, something is wrong.

How well do the major managed SIEMs cover these natively?

| Capability | Datadog SIEM | Google SecOps | CrowdStrike Next-Gen SIEM |
|---|---|---|---|
| Ingestion volume monitoring | ✅ Log pipeline monitors, anomaly detection on volume | ✅ Ingestion metrics dashboard, alerts on volume drops | ✅ Data connector health status |
| Ingestion latency tracking | ✅ Pipeline latency metrics available | ⚠️ Partial — ingestion delay visible but no native alerting | ⚠️ Limited — connector status but not per-source latency |
| Expected vs. active source coverage | ❌ You must define expected sources externally | ❌ No native "expected source" concept | ❌ No native coverage gap detection |
| Last-seen per source | ✅ Can be built via log monitors | ⚠️ Can be queried but no built-in dashboard | ⚠️ Connector-level only, not per-log-type |

The consistent gap across all three: **none of them know what you *should* be receiving.** They can tell you what's flowing. They can't tell you what's missing. That gap — between "pipeline healthy" and "environment fully covered" — is your responsibility.

## Your SIEM Already Handles Deduplication

A common recommendation is to build alert deduplication and entity grouping on top of your SIEM. Modern managed SIEMs — Datadog, Google SecOps, CrowdStrike — already do this natively. They group alerts by entity, deduplicate within time windows, and correlate related signals. Rebuilding this in a custom script is reinventing what you've already paid for.

Where you actually add value is in the gaps your SIEM doesn't bridge:

- **Same attacker, multiple identities.** SIEMs group by entity. If an attacker compromises one account, pivots through a service account, and exfiltrates via a third identity, the SIEM shows three separate alert clusters. The correlation that links them is your job.
- **Cross-product joins.** SSO logs, cloud control plane logs, and developer platform logs often don't share a common field that the SIEM can auto-correlate. Linking an Okta session to a CloudTrail identity to a GitHub audit event requires custom logic — or schema normalization. This is where standards like [OCSF (Open Cybersecurity Schema Framework)](https://schema.ocsf.io/) help. OCSF provides a vendor-neutral schema that maps fields like `actor.user.uid` consistently across sources. Datadog's [OCSF Common Data Model](https://www.datadoghq.com/blog/ocsf-common-data-model/) already normalizes CloudTrail, Okta, and GitHub audit logs into OCSF event classes, enabling cross-source detection rules. For example, a single rule can detect failed authentication across both Okta and CloudTrail because both map to the same OCSF Authentication event class. If your SIEM supports OCSF normalization, cross-product correlation becomes a query rather than custom code. If it doesn't, building that mapping is one of the highest-value investments a small D&R team can make.
- **Organizational context the SIEM can't see.** The SIEM knows that a service account accessed a data store. It doesn't know that your company just went through a reorganization and three engineers who lost access to that project still have active service account keys because the offboarding process didn't cover machine identities. It doesn't know that last week was an internal hackathon and half the unusual API activity is engineers experimenting in production-adjacent environments. It doesn't know that a new hire started on Monday and their access patterns will look anomalous for two weeks until they establish a baseline. This organizational context — reorgs, hackathons, onboarding waves, travel schedules, acquisition integrations — changes the priority of alerts in ways no automated system can capture. The analysts who carry this context in their heads are the most effective triagers, and no amount of automation replaces that judgment.

The honest answer to "how do you handle alert deduplication?" is: "The SIEM handles the basics. I focus on correlating across the gaps it can't bridge — linking identities across platforms into a single investigation, and bringing the organizational context that no log source contains."

## Threat Intelligence Is Mostly Not Actionable

The textbook says: subscribe to threat intel feeds, ingest IOCs into your SIEM, auto-escalate alerts that match known indicators. In practice, this adds far less value than the framework implies.

Most "threat intel" is reading news articles and vendor blog posts. Useful for awareness, but not for automated prioritization. When concrete IOCs do exist — IP addresses, file hashes, domain names — managed SIEM vendors are already ingesting them from the same feeds and creating detections on your behalf. You're not adding value by re-ingesting what your vendor already handles. And IOCs have a short shelf life. The IP address from last week's campaign isn't the one being used today.

Where threat intelligence genuinely helps is at the TTP level — understanding how attackers operate, not what specific indicators they've used in the past. The valuable question isn't "is this IP in a blocklist?" It's "are we detecting the techniques that are being used against organizations like ours?"

Consider a concrete example. When a supply chain compromise hits — say a widely-used npm package or a CI dependency like axios or trivy gets compromised — the IOC-level response is: "do we use the affected version?" That's necessary but insufficient. The TTP-level response asks deeper questions: do we have third-party dependencies that have access to secrets in our CI environment? If a dependency is compromised and harvests those secrets, do we have detection for anomalous credential usage? Do our endpoints have credential-harvesting malware detection? Can we detect a compromised build artifact calling out to an unexpected external host?

This is the shift from "do we have the specific vulnerability?" to "do we detect the class of attack technique?" — and it's where MITRE ATT&CK becomes genuinely useful. When communicating supply chain risk to engineering teams and leadership, framing it in ATT&CK techniques (T1195 Supply Chain Compromise, T1552 Unsecured Credentials, T1071 Application Layer Protocol for C2) provides a common language that maps directly to detection capabilities you can verify.

<!-- TODO: Add reference to SEC-T 2025 talk on using MITRE techniques for communicating supply chain threats. Video: https://www.youtube.com/watch?v=NppF78j83Yc — verify speaker name and exact title before publishing. -->

This reframes threat intel from a SIEM input to a detection validation tool:

| Intelligence Type | Value for Prioritization | How to Use It |
|---|---|---|
| **IOC-level** (IPs, hashes, domains) | Low — your SIEM vendor already ingests these | Useful during active investigations to check if a specific indicator is known. Not useful for proactive prioritization. |
| **TTP-level** (techniques, procedures, tooling patterns) | High — this is where you validate detection coverage | Map techniques used against your sector to your detection rules. Identify gaps. Build detections for the gaps. |
| **Strategic** (who targets your sector, why, what they want) | Medium — shapes your roadmap, not your daily triage | Understanding who targets your sector and why informs your detection roadmap and tabletop scenarios. It doesn't feed into automated alert prioritization. |

## What Actually Works

After building these systems twice, my operating model is simpler than any framework:

**Every investigated alert produces an action that improves the system.** If it was a true positive, ask: can the response be automated next time? If it was a false positive, ask: why did this rule fire, and what's the minimum change to prevent it from firing again? If it was inconclusive, ask: what data was missing, and can we add it?

![Alert investigation feedback loop](/images/alert-feedback-loop.svg)

**Prioritize by blast radius, not by environment label.** The common advice — suppress staging alerts, prioritize production — makes a dangerous assumption: that non-production environments are isolated. They often aren't. Credentials are reused. Network paths are shared. A staging database connects to the same data store as production. A developer's laptop has access to both environments.

Threat actors don't check your environment tags before deciding to attack. A compromised credential in staging that grants access to production secrets is a production incident, regardless of where the initial alert fires. A malware detection on a sandbox host that shares a VPN split-tunnel with your corporate network has production blast radius.

Instead of filtering by environment, assess each alert by asking: **if this alert is real, what can the attacker reach from here?** A credential compromise in a fully isolated sandbox with no shared secrets and no network path to production is genuinely low priority. The same credential compromise in a staging environment that shares IAM roles with production is critical. The environment label doesn't tell you which case you're in — the blast radius does.

**Metrics tell you if you're getting better.** Track the ratio of actionable alerts to total alerts over time. If your tuning is working, this ratio increases month over month. If it's flat, your tuning is chasing individual rules instead of addressing systemic noise sources.

![Actionable alert ratio trend](/images/alert-ratio-trend.svg)

**Know your throughput.** A useful exercise: measure how long an average alert investigation takes your team. If a thorough triage takes 30 minutes and your team of three has 6 effective investigation hours per day (accounting for meetings, context-switching, and other responsibilities), your maximum throughput is:

```
3 analysts × 6 hours × (60 min / 30 min per alert) = 36 alerts/day
```

That's your ceiling. Everything above 36 either gets auto-triaged, batched for weekly review, or missed. This math should drive your tuning investment: every rule you tune to auto-resolve frees a slot in that 36. Every rule you retire removes noise that competes for attention. The goal isn't to investigate every alert — it's to ensure the 36 you investigate are the right 36.

This formula improves as you reduce investigation time through better tooling (pre-enriched alerts, playbook automation, faster SIEM queries) and as you increase automation coverage. Track your actual mean-investigation-time monthly — if it's dropping, your tooling investment is working.

**Accept that you'll miss things.** No prioritization framework catches everything. The goal isn't perfect coverage — it's maximizing the signal quality of what your team can actually investigate. A team of three investigating 30 high-quality alerts per day is more effective than the same team drowning in 200 alerts and investigating none of them thoroughly.

The frameworks aren't wrong. They're just incomplete. They describe the destination without acknowledging that most teams are starting from a dirt road, not a highway. Start with what you can do today — blast radius assessment, feedback loops, honest coverage verification — and build toward the matrix, not from it.