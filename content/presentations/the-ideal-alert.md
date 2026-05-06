---
title: "The Ideal Alert"
date: 2026-04-12
draft: false
type: "presentations"
event: "Internal D&R talk"
tags: ["Detection engineering", "Alert tuning"]
description: "What would the perfect Friday security alert look like — and why doesn't it exist? A breakdown of the systems that have to work before any alert can be truly actionable."
---

<!-- {{< label "The hook" >}} -->

## It's Friday at 3:47 PM.

Your phone buzzes. Slack. It's an alert in the security channel

{{< slack-alert >}}

Everything you need. You know what happened, where, who, and what to do next. You act immediately.

<!-- slide -->

{{< label "The reality" >}}

## This is what your Friday alert looks like

<!-- Ideal alert has:
1. Blast radius: customer-exports, prod-analytics
2. Attribution: Lazarus Group — HIGH confidence
3. Signals: Okta + CloudTrail + outbound correlated -->

<!-- {{< slack-alert >}} -->

{{< guardduty-s3-alert >}} 

{{< alert-queue >}}

The gap between these two alerts is not a SIEM configuration problem. It's five separate problems, each one real.

<!-- slide -->

{{< label "Problem 1 of 5" >}}

## Asset criticality is aspirational.

The ideal alert knew the blast radius instantly: which S3 buckets, which RDS instance, what data. That requires every resource to have an owner, an environment tag, and a criticality label — all current.

{{< cmdb-table >}}

The SIEM fires on the service account. You can't tell if it matters, because nobody ever tagged it.

<!-- slide -->

The usual fallback is a simple heuristic: “Does this alert involve customer data? High. Authentication system? High. Production? High. Everything else? Low.” But that also breaks down quickly. A service may be tagged as staging and still communicate with production systems. A background job may not store customer data, but it may handle tokens or credentials that can reach it. Clean categories assume clean infrastructure, and clean infrastructure is rare.

Instead: assess blast radius from first principles. *If this alert is real, what can the attacker reach from here?* That question works even when the CMDB doesn't.

<!-- slide -->

{{< label "Problem 2 of 5" >}}

## Your Log Pipeline Is Lying to You

The ideal alert correlated three independent sources — Okta, CloudTrail, and outbound traffic — into a single story. That requires all three to be ingesting, current, and joinable.

{{< log-coverage >}}

<!-- Your SIEM tells you the pipeline is running. It doesn't tell you a new AWS account was spun up last Tuesday with no audit logging enabled. That gap — between "pipeline healthy" and "environment fully covered" — is where blind spots live. -->

A log source health practice should be your first investment before building any detection logic:

- **Ingestion volume per source against a rolling 24-hour baseline.** A sudden drop means a pipeline broke or a source stopped sending.
- **Ingestion latency per source.** If there's a 30-minute delay between event and SIEM availability, your "real-time" detection has a 30-minute blind spot.
- **Coverage map: expected sources versus active sources.** Every cloud account, every SaaS tool, every log type you expect — compared against what's actually arriving.
- **Last-seen timestamp per source.** The simplest check. If a source that normally sends every minute hasn't sent anything in an hour, something is wrong.

<!-- slide -->

{{< label "Problem 3 of 5" >}}

## Threat Intelligence Is Mostly Not Actionable

"Lazarus Group — confidence: HIGH." That line in the ideal alert implies weeks of TTP analysis, not a feed match. In practice, threat intel has a shelf-life problem.

{{< ioc-timeline >}}

The IP that fires the match today is not the one being used today. Your SIEM vendor already ingests the same IOC feeds you'd subscribe to — re-ingesting them adds no value.

Where threat intelligence actually helps is at the TTP level: not "is this IP known?" but "are we detecting the techniques that target organizations like ours?" That's a detection coverage question — not a feed question.

This reframes threat intel from a SIEM input to a detection validation tool:

| Intelligence Type | Value for Prioritization | How to Use It |
|---|---|---|
| **IOC-level** (IPs, hashes, domains) | Low — your SIEM vendor already ingests these | Useful during active investigations to check if a specific indicator is known. Not useful for proactive prioritization. |
| **TTP-level** (techniques, procedures, tooling patterns) | High — this is where you validate detection coverage | Map techniques used against your sector to your detection rules. Identify gaps. Build detections for the gaps. |
| **Strategic** (who targets your sector, why, what they want) | Medium — shapes your roadmap, not your daily triage | Understanding who targets your sector and why informs your detection roadmap and tabletop scenarios. It doesn't feed into automated alert prioritization. |

<!-- slide -->

{{< label "Problem 4 of 5" >}}

## Your SIEM Already Handles Deduplication

<!-- The ideal alert linked Okta → CloudTrail → S3 into one story. SIEMs group by entity. If the attacker pivots through three identities, the SIEM generates three separate alert clusters — and the connection is invisible. -->

A common recommendation is to build alert deduplication and entity grouping on top of your SIEM. Modern managed SIEMs — Datadog, Google SecOps, CrowdStrike — already do this natively. They group alerts by entity, deduplicate within time windows, and correlate related signals. Rebuilding this in a custom script is reinventing what you've already paid for.

{{< siem-cards >}}

<!-- Three separate alert clusters. No shared entity field. Your SIEM can't auto-join an Okta session to a CloudTrail role assumption to a DNS query — they come from different schemas, different sources, different entity models. The analyst who knows to look across all three is doing the correlation that no automation provides.

This is where OCSF normalization pays off: when sources share a common field model, cross-source detection becomes a query rather than custom code. When they don't, someone has to build the bridge manually. -->

Where you actually add value is in the gaps your SIEM doesn't bridge:

Same attacker, multiple identities
Cross-product joins
Organizational context the SIEM can't see.

<!-- slide -->

<!-- {{< label "Problem 5 of 5" >}}

## It's Friday at 4:47 PM.

Even if all four previous problems were solved, there's a capacity ceiling. Your team has a maximum throughput — and Friday afternoon is not the moment to exceed it.

{{< alert-queue >}}

Everything above 36 either gets auto-triaged, batched for Monday, or missed. This alert arrived as #73 in the queue. It's Saturday before anyone looks at it closely. By then, the exfil window closed six hours ago. -->

<!-- slide -->

<!-- {{< label "The gap" >}}

## What the ideal alert actually requires.

{{< gap-table >}} -->

<!-- slide -->

## What Actually Works

**Every investigated alert produces an action that improves the system.** If it was a true positive, ask: can the response be automated next time? If it was a false positive, ask: why did this rule fire, and what's the minimum change to prevent it from firing again? If it was inconclusive, ask: what data was missing, and can we add it?

![Alert investigation feedback loop](/images/alert-feedback-loop.svg)

<!-- slide -->

**Prioritize by blast radius, not by environment label.** The common advice — suppress staging alerts, prioritize production — makes a dangerous assumption: that non-production environments are isolated. They often aren't. Credentials are reused. Network paths are shared. A staging database connects to the same data store as production. A developer's laptop has access to both environments.

Threat actors don't check your environment tags before deciding to attack. A compromised credential in staging that grants access to production secrets is a production incident, regardless of where the initial alert fires. A malware detection on a sandbox host that shares a VPN split-tunnel with your corporate network has production blast radius.

Instead of filtering by environment, assess each alert by asking: **if this alert is real, what can the attacker reach from here?** A credential compromise in a fully isolated sandbox with no shared secrets and no network path to production is genuinely low priority. The same credential compromise in a staging environment that shares IAM roles with production is critical. The environment label doesn't tell you which case you're in — the blast radius does.

<!-- slide -->

**Metrics tell you if you're getting better.** Track the ratio of actionable alerts to total alerts over time. If your tuning is working, this ratio increases month over month. If it's flat, your tuning is chasing individual rules instead of addressing systemic noise sources.

![Actionable alert ratio trend](/images/alert-ratio-trend.svg)

<!-- slide -->

**Know your throughput.** A useful exercise: measure how long an average alert investigation takes your team. If a thorough triage takes 30 minutes and your team of three has 6 effective investigation hours per day (accounting for meetings, context-switching, and other responsibilities), your maximum throughput is:

```
3 analysts × 6 hours × (60 min / 30 min per alert) = 36 alerts/day
```

That's your ceiling. Everything above 36 either gets auto-triaged, batched for weekly review, or missed. This math should drive your tuning investment: every rule you tune to auto-resolve frees a slot in that 36. Every rule you retire removes noise that competes for attention. The goal isn't to investigate every alert — it's to ensure the 36 you investigate are the right 36.

This formula improves as you reduce investigation time through better tooling (pre-enriched alerts, playbook automation, faster SIEM queries) and as you increase automation coverage. Track your actual mean-investigation-time monthly — if it's dropping, your tooling investment is working.

<!-- slide -->

**Accept that you'll miss things.** No prioritization framework catches everything. The goal isn't perfect coverage — it's maximizing the signal quality of what your team can actually investigate. A team of three investigating 30 high-quality alerts per day is more effective than the same team drowning in 200 alerts and investigating none of them thoroughly.

The frameworks aren't wrong. They're just incomplete. They describe the destination without acknowledging that most teams are starting from a dirt road, not a highway. Start with what you can do today — blast radius assessment, feedback loops, honest coverage verification — and build toward the matrix, not from it.