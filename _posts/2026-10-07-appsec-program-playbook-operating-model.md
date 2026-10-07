---
layout: post
title: "3.1 The AppSec Program Playbook: Operating Model - Strategy, Ownership & the First 90 Days"
date: 2026-10-07 11:00:00 -0000
categories: appsec security devsecops
tags: ["The AppSec Program Playbook", "Leadership", "Risk Management", "Operations"]
series: "The AppSec Program Playbook"
series_part: "3.1"
---

It's the new AppSec lead's first Monday at Pinecart.

By 10 a.m. everything is urgent. The VP of Engineering wants "the security roadmap" for end-of-month planning. Last year's scanner shows 40,000 open findings; nobody knows which are real. `#sec-exceptions` has 60 unanswered threads, most of them "can we ship anyway?" The payments team says the PCI DSS (Payment Card Industry Data Security Standard) assessor is back next quarter. The SOC (security operations center) asks whether last night's npm advisory is AppSec's or theirs.

There's no inventory, no owner list and no charter. Nothing is defined.

The instinct is to start fixing. That's how programs stay busy for a year and end it no safer.

<div style="text-align: center; margin: 30px 0;">
  <img src="/assets/images/playbook-3-1.png" alt="A hooded strategist leans over a large wooden table in a vaulted stone chamber, hands on an unfinished map with glowing cyan lines, an hourglass and candles beside him, while other cloaked figures wait in the shadows of the arches" style="max-width: 100%; height: auto; border-radius: 8px; box-shadow: 0 2px 12px rgba(0,0,0,0.15);">
  <p style="font-size: 0.9em; color: #666; margin-top: 8px;"><em>Plan the road before you start building it.</em></p>
</div>

*Pinecart is the series' fictional e-commerce company: about 400 engineers and a few thousand repos. The company is made up; the problems aren't. Every Pinecart number is illustrative.*

## Decide What the Program Is For, Before You Pick Tools

In [3.0 The AppSec Program Playbook: A Program, Not a Scanner](/2026/10/07/appsec-program-playbook-introduction/) I argued that security gets done when it blocks the work, when it's effortless, or when someone owns it with a real deadline. Everything else gets skipped. The program's job is to manage all four, including shrinking the pile that gets skipped.

Phase 0 decides how. Before choosing a tool, settle what the program owns, who owns risk, how the work is organized, where you stand against a standard, and **what you will deliberately not do first.**

The standards agree that this comes first:

- NIST's [Secure Software Development Framework (SP 800-218 v1.1)](https://csrc.nist.gov/pubs/sp/800/218/final) starts with *Prepare the Organization* (PO): security requirements (PO.1) and roles and responsibilities (PO.2) come before everything else. (SSDF v1.2 is out as an [initial public draft](https://csrc.nist.gov/Projects/ssdf/publications); as I write, v1.1 is still final.)
- [OWASP SAMM (Software Assurance Maturity Model) v2](https://owaspsamm.org/model/) lists Governance (Strategy & Metrics, Policy & Compliance, Education & Guidance) first of its five business functions.
- [BSIMM16](https://www.blackduck.com/resources/analyst-reports/bsimm.html), which studied 111 firms, starts its framework with Governance too.

None of them tell you what to do on Monday. This post does.

## The Problem: Everything Is Urgent, Nothing Is Defined

Every item in that scene is real and has a sponsor. You can't do them all first. Pick wrong and you spend the year serving whoever shouts loudest.

| Tension | One side | The other side |
|---|---|---|
| **Speed vs. foundation** | Show a win in the first month | Build the inventory and ownership every win depends on |
| **Mandate vs. adoption** | A policy gets formal compliance, not real change | A paved road gets real use, but slowly |
| **Fix the backlog vs. stop new findings piling up** | Leadership sees the 40,000 number | New findings arrive faster than old ones close |

### Field Notes: What Comes Last

[3.0](/2026/10/07/appsec-program-playbook-introduction/) covers the order I'm building my program in: make the tooling real, automate delivery, build visibility, go to the teams, make fixing easy, then set standards. People get that last step backwards. **Enforced mandates come last, once there's a road to follow. Baseline non-negotiables and the compliance obligations you already have get written down from day one.** A mandate written before the paved road is a list of things teams can't do yet. Written after, it describes how they already work.

## How to Do It: Nine Steps

### Step 1: Write the Charter and Draw the Boundaries

A charter is one page that stops "is this ours?" arguments before they start. The part people skip is the boundary: what AppSec **owns**, what it **influences**, and what belongs to someone else.

Most friction is with the teams next door:

| Area | AppSec owns | AppSec influences | Someone else owns |
|---|---|---|---|
| **Secure SDLC** (software development lifecycle) | Standards, paved-road templates, scanner config, triage rules | Framework and library choices | Engineering owns shipping and fixing |
| **Platform / CI** (continuous integration) | The shared pipeline security stages | Runner hardening, build isolation | Platform owns CI uptime and the pipeline product |
| **SOC / IR** (incident response) | Exposure analysis ("are we affected?"), fix coordination, root-cause input | Detection content for app-layer attacks | SOC/IR owns detection, incident command, forensics |
| **GRC** (governance, risk and compliance) | Control evidence from the pipeline | How requirements are worded for engineers | GRC owns the control framework, audits and the risk register |

Own what only you can do well. Influence what others run better: you don't want the CI platform, just your stage in it. Write the "someone else" column down. The charter template at the end also covers cloud, privacy and legal.

### Step 2: Split Who Finds, Who Fixes and Who Accepts

The most important sentence in the charter:

> **Security owns visibility and guidance. Engineering owns the fix. The business owns risk acceptance.**

If security owns the fix, it becomes the bottleneck, and the org learns vulns are someone else's problem. If security owns risk acceptance, it becomes the team that always says no, and the business works around it. If engineering owns visibility, you get a green dashboard and a breach. Each party owns what it can actually decide.

"The business" means whoever is accountable for the service's results: a business exec for the highest-risk items, the engineering leader who owns the service for the rest. Never the security team.

[NIST CSF 2.0](https://www.nist.gov/cyberframework) (Cybersecurity Framework) puts this under Govern: roles, responsibilities and authorities (GV.RR) and risk-management strategy (GV.RM).

The priority levels (P0 to P4) come from <span data-cs="/2026/10/07/appsec-program-playbook-prioritize-ruthlessly/">3.4</span> *(coming soon)*. If you think in scanner labels, they map roughly like this. The difference: in this series the priority comes from real risk, not the scanner's label.

| Priority | Roughly like | Example | Fix within |
|---|---|---|---|
| **P0** | Emergency | A leaked live secret | Revoke now; never accepted as a risk |
| **P1** | Critical | A bug attackers are already exploiting, on an internet-facing system | 72 hours |
| **P2** | High | A serious bug on an exposed system that is reachable, or not yet known to be unreachable | 7 days |
| **P3** | Medium | A serious bug that isn't exposed, or a moderate one that is | 30 days |
| **P4** | Low | Moderate risk, not exposed | 90 days |

**RACI for a Pinecart-sized org** (R = responsible, A = accountable, C = consulted, I = informed).

| Activity | AppSec | Eng team | Eng leadership | Platform | SOC/IR | GRC | Business owner / exec |
|---|---|---|---|---|---|---|---|
| Secure SDLC standards | **A/R** | C | C | C | I | C | I |
| Asset inventory and ownership data (3.2) | A | R (keeps its own entries current) | C | R | I | I | - |
| Triage and prioritization (3.4) | **A/R** | C | I | - | C | I | - |
| Fixing findings | C (guidance, automated PRs) | **R** | **A** | C | - | I | - |
| SLA (service-level agreement, i.e. a deadline) escalation | R | I | **A** | - | - | I | I |
| Risk acceptance / exceptions | R (security review) | R (requests) | **A** for P2-P4 (the leader who owns the service) | - | C | C | **A** for P1 (the most urgent) |
| Advisory intake and exposure ("are we affected?") | **A/R** | C | I | C | C | - | - |
| Security incident command | C | C | I | C | **A/R** | I | I |
| Regulatory notification (SEC, EU CRA, breach laws) | C | - | I | - | C | R (with legal) | **A** (legal / exec) |

Note two things. Engineering leadership, not the team, is **accountable** for fixes, because SLA escalation needs someone who can change priorities. And AppSec is never accountable for risk acceptance at any priority; it does the security review (Step 7).

### Step 3: Pick an Operating Model, Then Staff It with Champions

There are three common shapes:

<div class="mermaid">
graph LR
    H["AppSec hub<br/>paved road, standards, triage"]
    H -->|tools, training, guidance| A["Champion<br/>Team A"]
    H -->|tools, training, guidance| B["Champion<br/>Team B"]
    H -->|tools, training, guidance| C["Champion<br/>Team C"]
    A -.->|what hurts, what's new| H
    B -.->|what hurts, what's new| H
    C -.->|what hurts, what's new| H
</div>

| | Centralized | Federated | Hub-and-spoke with champions |
|---|---|---|---|
| **How it works** | One AppSec team does reviews, triage and guidance for everyone | Security engineers sit in, and often report to, business units (BUs) | A hub builds the paved road and standards; a champion in each team carries the practice |
| **Best fit** | Small org, one product, early program | Large, different BUs with their own stacks and regulators | Most mid-to-large engineering orgs |
| **Strength** | Consistent quality, one set of rules | Deep context, fast answers, trusted locally | Grows without hiring one-for-one; context stays in the team |
| **How it fails** | Bottleneck: the queue becomes the program | Drift: five BUs, five programs, no shared data | Champions without time or support quietly drop off |

For Pinecart's 400 engineers, hub-and-spoke is the answer. A central team can't read every PR, and there aren't enough business units to justify federation.

**Hub size.** Industry benchmarks like [BSIMM](https://www.blackduck.com/resources/analyst-reports/bsimm.html) show bigger security teams than many companies actually have. In real life, a handful of AppSec people often covers hundreds of developers. That's exactly why the paved road and champions matter: they're how a small hub scales. The better your paved road, the further a small team stretches.

#### The Security Champions Program

A champion is a product-team engineer who cares about security, has time for it, and is the team's first stop for questions. BSIMM calls this group "the satellite." In the BSIMM16 data, **96% of the top-scoring firms have security champions, and only 30% of the bottom-scoring firms do**. That's a correlation, not proof of cause, but it's a strong one.

The [OWASP Security Champions Guide](https://securitychampions.owasp.org/principles/) lists ten principles. Four decide whether the program lasts: *secure management support*, *nominate a dedicated captain*, *invest in your champions* and *anticipate personnel changes*. How I design each part:

| Element | Design choice | Why |
|---|---|---|
| **Selection** | Volunteers first, then fill gaps so every Tier 1 service's team has one. One per team, not per 50 engineers. | Coverage follows ownership (3.2). |
| **Time** | A visible share of time, agreed with the manager and in the team's planning. Pinecart starts at 10% (illustrative). | Time nobody agreed to disappears. |
| **Enablement** | Monthly session, private channel with the hub, early access to paved-road changes, a short threat-modeling and triage course | Enthusiasm isn't expertise. |
| **Responsibilities** | First-pass triage of team findings, flagging designs for review, false-positive feedback, rolling out paved-road updates | Real jobs, not a title. |
| **Recognition** | Counts in performance reviews, named role in the catalog, conference or cert budget, credit in program reports | Unrewarded work gets dropped. |
| **Succession** | Every champion has a backup; champion handover is on the team reorg checklist | People move. |

**Field notes.** The champions group I was part of started as a monthly security training I ran for peers, not a program. It grew into the organization's official security champions group. The community existed before the org chart did. If you start from nothing, start with something people want to show up to.

The resistant-team story in [3.0](/2026/10/07/appsec-program-playbook-introduction/) shows the same thing from the other side: **people adopt what makes their week easier, and they follow the person who did the work with them.** A champions program is that lesson at scale.

> **Dev lens:** If my manager asks me to be the team's security champion and nothing comes off my plate, I say yes and then do very little. Not because I don't care, but because sprint work always comes first. Put the hours in the team's planning and make the role count in my review, and I'll actually show up.

### Step 4: Get a Baseline with a Quick SAMM Self-Assessment

A baseline shows where the gaps are. A year later, it proves you closed some.

[OWASP SAMM v2](https://owaspsamm.org/model/) has five business functions and fifteen practices, each scored 0 to 3. The [SAMM Toolbox](https://github.com/owaspsamm/toolbox-spreadsheet) gives interview questions per activity and rolls answers up into practice, function and overall scores.

**Keep it light.** Skip the six-week assessment. Interview three or four groups (AppSec, two typical product teams, platform) for a couple of hours each. You want a baseline you can defend, not a perfect one.

**Pinecart's baseline (illustrative): the practices that move most, plus the two deferred:**

| Function | Practice | Now | 12-month target | Roadmap item |
|---|---|---|---|---|
| Governance | Strategy & Metrics | 0.5 | 1.5 | Charter, roadmap one-pager, baseline metrics (3.6) |
| Governance | Education & Guidance | 0.5 | 1.5 | Champions program, secure-coding guidance for the top bug classes |
| Design | [Secure Architecture](https://owaspsamm.org/model/design/secure-architecture/) | 0.5 | 1.5 | Paved-road libraries for auth, secrets, HTTP clients (3.3) |
| Implementation | Secure Build | 1.0 | 2.0 | Pipeline security stages, dependency pinning, SBOMs, or software bills of materials (3.2, 3.3) |
| Implementation | Defect Management | 0.5 | 2.0 | One findings model, priorities and SLAs (3.4) |
| Operations | Environment Management | 0.5 | 1.5 | Automated patching, base-image update cadence (3.5) |
| Verification | Architecture Assessment | 0.25 | 0.5 | Deferred (see below) |
| Verification | Requirements-driven Testing | 0.25 | 0.5 | Deferred |

The other seven practices move a point or less, mostly as side effects.

**Nobody targets 3.** Level 3 everywhere takes years, most organizations don't need it, and SAMM leaves targets to you. And **two practices are deferred on purpose.** Saying out loud what you won't do this year keeps the roadmap believable.

### Step 5: The First 90 Days

This is the plan I follow. My first answer to "what would you do first?" is always the same: **ask what's broken. Then, almost always, visibility, because everything else depends on it.**

<div class="mermaid">
graph TB
    subgraph D1["Days 0-30: listen and see"]
        direction LR
        a1["Listening tour"] --> a2["Inventory + owners"] --> a3["SAMM baseline"]
    end
    subgraph D2["Days 30-60: one win, one pilot"]
        direction LR
        b1["Quick win:<br/>secrets to owners"] --> b2["Paved-road pilot<br/>with one team"] --> b3["First champions"]
    end
    subgraph D3["Days 60-90: measure and commit"]
        direction LR
        c1["Three baseline<br/>numbers"] --> c2["Exception policy<br/>+ IR runbook"] --> c3["Roadmap to<br/>leadership"]
    end
    D1 --> D2 --> D3
</div>

| Window | Goal | Outcome | Not doing, on purpose |
|---|---|---|---|
| **Days 0-30: listen and see** | Understand the org before changing it | You know what exists, who owns it, which SLAs are real, and your SAMM baseline | Mandating, buying, or attacking the backlog |
| **Days 30-60: one win, one pilot** | Earn trust with something teams feel | **One quick win** teams notice (e.g. leaked secrets routed straight to the owner) and **one paved-road pilot** with a willing team | Rolling the pilot out to everyone, or writing the standard |
| **Days 60-90: measure and commit** | Turn what you learned into a plan leadership can fund | A baseline with **three numbers**, not thirty; exception policy and SOC/IR runbook live; roadmap presented upward | Promising the backlog will hit zero |

The quick win must **take work away** from engineers; a new blocking gate never counts. And "auto-close" means *fixed and confirmed by a rescan*, never "unreachable, so we closed it." Findings shown to be unreachable, with evidence, go to the hygiene lane in <span data-cs="/2026/10/07/appsec-program-playbook-prioritize-ruthlessly/">3.4</span> *(coming soon)*, where they still get fixed.

**Field notes.** I'm now pushing harder on fixing than the table does. By day 60: one automated fix path running (dependency and container updates as tested PRs, PR checks at the source, secrets routed to owners). By day 90: SLAs by asset class, a monthly MTTR (mean time to remediate) trend, and an AI-assisted triage pilot where a human approves every downgrade. That pilot is the L4 rung in 3.4: target state, not something I've run in production. Make fixing easy early. That's what earns you the right to ask for anything else.

### Step 6: Make Decisions Without All the Facts

In Phase 0 you decide without an inventory, a real false-positive rate, or knowing which teams will engage. Waiting for certainty is also a decision, usually the wrong one. Three habits help.

**1. Sort decisions into one-way and two-way doors.** The idea comes from Jeff Bezos's 2015 shareholder letter. Two-way doors can be undone: decide fast, close to the problem. One-way doors are expensive to undo: slow down, write it up, get the right people in the room.

| Decision | Door | Who decides | How fast |
|---|---|---|---|
| Tune a noisy SAST (static application security testing) rule | Two-way | AppSec team | Same day |
| Make a control *block* for Tier 1 | Mostly one-way (lost trust is hard to win back) | AppSec lead + eng leadership | After a pilot and a false-positive budget |
| Accept a P1 risk | One-way while it's open | Business exec (Step 7) | Hours, with an interim control |

**2. Write short decision records, with a revisit date.** The architecture decision record (ADR), from Michael Nygard's [2011 post](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions), works for program decisions too. I call these PDRs (program decision records). Add a field most templates lack: **a revisit date**. It makes changing your mind part of the plan, not a retreat. The example is an L1-rung decision in 3.4's terms: it still gates on a CVSS (Common Vulnerability Scoring System) label, which 3.4 later retires.

```markdown
# PDR-007: Gate on new Critical findings only, for Tier 1, starting next sprint

- Status: accepted
- Date: <decision date>
- Revisit by: <date + 90 days>, or earlier if the false-positive rate on blocked PRs exceeds 10%
- Deciders: AppSec lead, Director of Engineering (Checkout)

## Context
Gating on Critical+High blocked Checkout teams several times a week; most blocks
were in dependencies the code never calls. Teams have started asking for blanket exceptions.

## Decision
Block only on new Critical findings in Tier 1 repos. Everything else warns, and goes to
the hygiene lane with a backstop deadline (see 3.4). KEV and verified live secrets always block, in every repo.

## Consequences
+ Less friction; exception requests should drop.
- Residual risk moves out of the gate. It has to be tracked in the hygiene lane, or it disappears.

## Evidence we'll check at the revisit
Blocked-PR count, override count, hygiene-lane burn-down, exception requests per week.
```

**3. Pilot before you mandate.** Every control earns its mandate on one willing team first. The pilot shows the false-positive rate, the time cost, and what the docs are missing.

**Field notes.** The first gate in the program I'm building blocked on whatever the scanner called Critical or High. A fair place to start, the wrong place to stay. Over several meetings with engineering it narrowed: first to Critical only, then to Critical and reachable where we had reachability data. That's the ladder in <span data-cs="/2026/10/07/appsec-program-playbook-prioritize-ruthlessly/">3.4</span> *(coming soon)*. Each of those meetings was a revisit of a two-way-door decision. A decision record with a revisit date and named evidence turns that from a reaction to pressure into a scheduled check.

### Step 7: Risk Acceptance and Exceptions, with an Expiry Date

Every program needs a "not now" that doesn't mean "never." Without a policy, exceptions still happen in DMs and `#sec-exceptions`, with no owner, end date or record. That's the **shadow backlog**: risk the organization has accepted without anyone deciding to accept it.

Every exception needs **an owner, a justification, an expiry date and an approver.** P1 to P3 also need **a verified compensating control**, and the higher the priority, the more senior the approver. Here's the full policy, keyed to the priorities in <span data-cs="/2026/10/07/appsec-program-playbook-prioritize-ruthlessly/">3.4</span> *(coming soon)*. **Security review** confirms the control is verified and the approver understands what they're accepting. It can block an exception with an unverified control, but never accepts the risk itself.

| Priority (3.4) | Fix SLA | Can it be excepted? | Who accepts the risk | Security review | Max expiry per grant | Renewals |
|---|---|---|---|---|---|---|
| **P0**: verified live secret | Revoke and rotate now; compromise check and log review within 72 hours | **No.** Rotation and the compromise check are never deferred; only the *code cleanup* (removing it from history) can be scheduled. | - | - | - | - |
| **P1**: KEV (CISA's Known Exploited Vulnerabilities catalog) or active exploitation on an exposed asset (full definition in 3.4) | ≤ 72 hours | **Never for KEV.** [CISA's required action](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) or isolating the asset closes or re-tiers the finding; neither is an exception. Other P1s: only with a verified control that removes the exploit's precondition. | Accountable business exec (VP or above) | CISO | 14 days | One, then the CEO or CTO decides, after CISO review |
| **P2**: worst-impact, reachable or unknown, exposed; other KEV on a non-exposed asset | ≤ 7 days | **Not for KEV** (required action or isolation only). Otherwise yes, with a verified control that removes the precondition. | VP of Engineering for the area | Head of Security | 30 days | One, then escalate to the P1 approvers |
| **P3**: high risk (top impact tier not exposed or strongly mitigated; or middle tier, exposed) | ≤ 30 days | Yes, with a verified control | Engineering director who owns the service | AppSec lead | 90 days | Two, then escalate one level |
| **P4**: moderate risk (middle tier not exposed, or low tier) | ≤ 90 days (60 if exposed) | Yes | Engineering manager who owns the service | AppSec engineer | 180 days | One, then escalate one level |
| **Hygiene lane** | Backstop: next scheduled upgrade or ≤ ~180 days, whichever comes first | Yes, as a planned upgrade date | Engineering manager | AppSec informed | To the planned upgrade, max ~180 days; longer needs P4-level approval | Re-promotion triggers in 3.4 still apply |

These approver levels are my recommendation, not an industry standard. Tune them to your org chart, but keep the shape: **higher priority means a more senior approver, a shorter expiry and fewer renewals. KEV is never excepted, at any priority.**

Rules that make it work:

- **The approver must own the consequence.** An AppSec engineer can't accept a P1 for the business; a VP shouldn't get paged for a P4.
- **Only some controls can back an exception.** A filter (a WAF, or web application firewall, rule, or a virtual patch) buys time while the fix ships; the SLA clock keeps running. A test-verified control that removes the exploit's precondition (an egress allowlist against SSRF, or server-side request forgery; the vulnerable feature turned off) can back a time-boxed exception. "It's internal" is neither. 3.4's mitigator table has the detail; there, the same verified control can re-tier the finding instead. Either way it carries evidence and an expiry.
- **Expiry reopens the finding automatically.** Nobody should have to remember.
- **Renewals escalate.** Once the renewal limit in the table is used up, the next renewal goes one level up. So "renew it again" always means a more senior person has to approve it.
- **Cap the total.** After about a year of back-to-back exceptions, it's no longer an exception. It's a funding decision for the roadmap.
- **Regulated scope is different.** Your exceptions don't bind an assessor. [PCI DSS v4.0.1](https://blog.pcisecuritystandards.org/just-published-pci-dss-v4-0-1) has its own compensating-control documentation (Appendices B and C), and Requirement 6.3.3 still expects critical patches within one month of release on in-scope systems ("critical" per your 6.3.1 risk ranking, not CVSS alone). If you're bound by CISA's BOD 26-04 (Binding Operational Directive), an internal exception can't extend its deadlines either.
- **Exceptions are data.** Report count, age, percentage past expiry and renewal rate by team, next to the findings. [NIST SP 800-40 Rev. 4](https://csrc.nist.gov/pubs/sp/800/40/r4/final) treats acceptance as a risk response you track, not a way to close a ticket.

> **Dev lens:** As a dev, if the exception form is easier than the fix, I file the exception. If it expires on its own and the renewal needs a VP's signature, I'm much more likely to put the fix in the next sprint. The process decides which path I take, not my good intentions.

**Field notes.** In the program I'm building, exceptions carry an owner, a compensating control and an expiry, with exec sign-off above a threshold.

If I had to defend one rule, it's reporting exceptions next to the findings. An accepted risk beside open findings, with an owner's name and an expiry date, stays a decision someone made. One in a separate spreadsheet gets forgotten.

The record should be structured data, not a Slack thread. YAML template at the end.

### Step 8: Draw the AppSec ↔ SOC/IR Line Before You Need It

The worst time to decide who owns a compromised npm package is the morning it's compromised. The SOC's "is this yours or ours?" should already be answered in the charter.

The rule: **AppSec owns exposure; SOC/IR owns compromise.** AppSec answers "are we affected, where, and how do we fix it?" SOC/IR answers "were we hit, what did they do, how do we contain it?" The handoff is a defined trigger, not a 2 a.m. judgment call.

**Handoff triggers** (any one sends it to SOC/IR):

- evidence of exploitation against our assets, from logs, alerts or a third party;
- a malicious package version was installed, built or run in CI or on a developer machine during the exposure window;
- a verified live secret was exposed publicly or shows use from an unknown source;
- the answer to "were we hit?" is still **unknown** after a set time box (assume compromise);
- a possible notification obligation (customer data, regulated scope, a product covered by the CRA).

**Shared artifacts**: one runbook, one ticket carrying both halves, an agreed channel. After the incident, the root cause comes back to AppSec as a paved-road change. The full advisory-to-IR loop, including a supply-chain compromise playbook, is in <span data-cs="/2026/10/07/appsec-program-playbook-remediate-at-scale/">3.5</span> *(coming soon)*.

**Field notes.** I've worked both sides of this line, in incident response and in AppSec. The goal is no gap where each team assumes the other has the advisory. The fix is boring: one named owner for intake, written triggers, and regular rehearsals with a real recent advisory.

### Step 9: Fund It, and Tie the Roadmap to Obligations and Velocity

Programs funded by fear get cut when nothing bad happens. Programs funded by obligations and velocity survive, because both are still true next year.

**Obligations first.** Map each to what it asks of AppSec, then to a roadmap item. For Pinecart (e-commerce SaaS with checkout and payments):

| Obligation | What it asks of AppSec (verified 2026-09) | Roadmap item |
|---|---|---|
| **SOC 2** ([AICPA Trust Services Criteria](https://www.aicpa-cima.com/resources/download/2017-trust-services-criteria-with-revised-points-of-focus-2022)) | Detect new vulnerabilities (CC7.1); authorize, test and approve changes (CC8.1) | Pipeline evidence of scanning and review |
| **PCI DSS v4.0.1**, Requirement 6 ([PCI SSC](https://blog.pcisecuritystandards.org/just-published-pci-dss-v4-0-1)) | Secure development and code review for bespoke software (6.2.x); risk-ranked vulns (6.3.1); an **inventory of bespoke software and its third-party components** (6.3.2); critical patches within one month (6.3.3); protect public-facing web apps (6.4.2); manage payment-page scripts (6.4.3) and detect tampering on those pages (11.6.1) | Inventory and SBOMs (3.2); paved road with review and WAF evidence (3.3); risk ranking and SLAs (3.4); checkout script inventory and tamper detection |
| **EU Cyber Resilience Act** ([Regulation (EU) 2024/2847](https://eur-lex.europa.eu/eli/reg/2024/2847/oj)) | For *products with digital elements* sold in the EU, including ones already on the market: since **11 September 2026**, [report](https://digital-strategy.ec.europa.eu/en/policies/cra-reporting) actively exploited vulns and severe incidents via the Single Reporting Platform run by ENISA (the EU Agency for Cybersecurity). Early warning in **24 hours**, notification in **72 hours**, final report 14 days after a fix (vulns) or one month after notification (incidents). SBOM and vuln-handling duties from **11 December 2027**. | Decide scope with legal; a 24-hour exploited-vulnerability path in the SOC/IR runbook; SBOMs per shipped artifact |
| **SEC cybersecurity disclosure** ([Release 33-11216](https://www.sec.gov/files/rules/final/2023/33-11216.pdf)), if publicly listed in the US | Yearly description of cyber risk management, strategy and governance (Reg S-K Item 106); material incidents disclosed within **four business days of determining materiality** (Form 8-K Item 1.05), and that determination made without unreasonable delay | Program described in terms the 10-K can use; an incident runbook that reaches a materiality decision fast |

Caveats. Pure SaaS is generally outside the CRA ([recital 12](https://eur-lex.europa.eu/eli/reg/2024/2847/oj); [analysis](https://www.technologyslegaledge.com/2026/02/cyber-resilience-act/)). Pinecart's hosted storefront probably isn't in scope; a mobile app, merchant plugin or SDK it ships to the EU likely is. Scope is a legal call. The SEC rules are in force despite a [petition to rescind Item 1.05](https://www.sec.gov/comments/4-856/4-856.htm) and a January 2026 review of Regulation S-K ([Debevoise](https://www.debevoisedatablog.com/2026/05/21/cybersecurity-incident-disclosure-form-8-k-tracker-two-year-update/)). PCI DSS v4.0.1 is the only active version as of this writing, its future-dated requirements mandatory since 31 March 2025.

**Velocity second.** Every roadmap item also says what it gives engineering back: fewer blocked PRs, faster audits, patches in minutes instead of sprints (3.6 pairs this with DORA, or DevOps Research and Assessment, metrics).

**Build vs. buy.** Decide after the inventory exists, never before. The questions I ask:

| Question | Leans *buy* | Leans *build* |
|---|---|---|
| Is it a commodity, like SCA (software composition analysis) matching or SAST engines? | Yes | - |
| Is it glue between your systems (routing, ownership, inventory joins)? | - | Yes; nobody sells your org chart |
| Does it need your context (tiers, owners, exposure) to be useful? | Only if it has open APIs to pull that context in | Yes |
| Does it lock findings into one vendor's format? | Avoid | Prefer open formats (SARIF, the Static Analysis Results Interchange Format; CycloneDX; OpenVEX) |

**Field notes.** I've owned vendor selection and budget decisions and briefed senior security and engineering leadership. Offer a choice, not a scare: two options, each with its cost, what it reduces and what it leaves open. Leadership picks; AppSec makes sure they pick with real information. The parts of my program that became standards, the pipeline stages and the patching automation, both made engineering's week easier, not just safer.

## Demo: An Exception Register You Can Copy

Step 7 said every exception needs an owner, a justification, an expiry date and an approver. In practice, exceptions end up in Slack threads and spreadsheets, and nobody notices when they expire. Here's a simple fix you can copy: **keep each exception as a small file in one Git repo, and let a check catch the problems for you.**

It's all in the [appsec-playbook-templates](https://github.com/hijacksecurity/appsec-playbook-templates/tree/main/operating-model/exception-register) repo, tested and ready to use.

### Where It Lives and Who Does What

There's **one exceptions repo for the whole company**, owned by AppSec. At Pinecart it's called `security-exceptions`. It's not in each team's code repo, for two reasons: you get one place to see every accepted risk, and a team can't quietly approve its own exception.

| Who | What they do |
|---|---|
| **The dev team** that needs the exception | Opens a PR that adds one file for it |
| **The approver** for that priority (Step 7) | Reviews the PR and accepts the risk, or says no |
| **AppSec** | Checks that the compensating control really works, then approves |
| **The checks** (GitHub Actions) | On every PR: validate the file and confirm both approvals are in. Every weekday: flag expired exceptions |
| **The owner** (one named person) | Fixes the issue before the expiry date, or asks for a renewal with a new PR |

### One Exception, Start to Finish

Here's how it plays out at Pinecart:

1. **The problem.** The checkout team has a P2 server-side request forgery (SSRF) bug. The real fix needs a new shared HTTP client, which is three weeks away. The P2 deadline is 7 days.
2. **The request.** Jane, on the checkout team, opens a PR in `security-exceptions` that adds `exceptions/p2/EXC-2026-014.yaml`. In it: what's wrong, who owns it (her), the temporary control (an egress allowlist that blocks the attack), and an expiry 30 days out, the maximum for P2.
3. **The check runs.** It confirms the file is in the right folder, isn't for a KEV finding, and doesn't go past 30 days. It passes.
4. **The approvals.** CODEOWNERS automatically asks the VP of Engineering and the Head of Security to review. AppSec tests the egress allowlist, confirms it blocks the exploit, and approves. The VP approves. A second check confirms there's one approval from each side (the VP accepts the risk, security reviews it), because CODEOWNERS on its own would accept either one. The PR merges, and the exception is now active.
5. **The happy ending.** The fix ships in three weeks. Jane opens a small PR deleting the file. Done.
6. **The other ending.** The fix slips. On day 30 the weekday check fails and opens a GitHub issue naming Jane. She either ships the fix or opens a renewal PR. A second renewal has to go one level higher.

<div class="mermaid">
graph TB
    A["Dev team opens a PR<br/>adding the exception file"] --> B["Check runs +<br/>approvers review"]
    B -->|rejected or check fails| X["Fix it now instead"]
    B -->|approved| C["Exception is active"]
    C -->|fix ships| D["PR deletes the file"]
    C -->|expiry date passes| E["Weekday check opens<br/>an issue for the owner"]
    E --> F["Ship the fix, or<br/>renew with a new PR"]
</div>

In the team's own code repo, the scanner suppression points back to the exception, for example `# EXC-2026-014` next to the ignored finding. Anyone who sees the suppression can find who approved it and when it ends.

### What an Exception Looks Like

```yaml
# exceptions/p2/EXC-2026-014.yaml
id: EXC-2026-014
asset: checkout-api
priority: P2
kev: false
summary: "SSRF in image-fetch endpoint; fix needs the shared HTTP client migration"
owner: jane.doe
compensating_control:
  type: precondition-removed        # filters like a WAF rule can't back an exception
  description: "Egress allowlist blocks internal and metadata ranges"
  verified_by: appsec-oncall
  evidence: "link to the test run showing the attack is blocked"
justification: "Fix needs the shared HTTP client, shipping in sprint 42"
accepted_by: vp-eng-commerce
security_review: head-of-security
approved_on: 2026-10-20
expires: 2026-11-19
```

The full template, with every field explained, is [in the repo](https://github.com/hijacksecurity/appsec-playbook-templates/blob/main/operating-model/exception-register/exception-template.yaml).

### What the Check Catches

Here's a real run against three Pinecart exceptions. One is fine; two aren't:

```text
::error file=exceptions/p1/EXC-2026-009.yaml::P1 granted for 30 days (max 14)
::error file=exceptions/p1/EXC-2026-009.yaml::expired on 2026-10-01; owner sam.lee must fix or renew
::error file=exceptions/p4/EXC-2026-017.yaml::KEV findings can't be excepted: apply CISA's required action or isolate the asset
3 problem(s) found.
```

The check fails when an exception:

- is past its expiry date;
- was granted for longer than its priority allows (P1: 14 days, P2: 30, P3: 90, P4: 180);
- is for a KEV finding or a leaked live secret, which can never be excepted;
- sits in the wrong priority folder (so a P1 can't get P4's easier rules);
- has no verified compensating control (P1 to P3), or uses a filter like a WAF rule as its control;
- is missing a required field, like the owner or the justification.

### Set It Up in Your Repo

Copy the two check scripts, the approvers list, the workflow, the CODEOWNERS example (with your team names) and the empty `exceptions/` folders. Then make both checks required in your branch rules. One setup step needs care: the approvals check reads team membership, so it needs a GitHub token that can read your org's teams. The [README](https://github.com/hijacksecurity/appsec-playbook-templates/tree/main/operating-model/exception-register) walks through every step.

> **Dev lens:** As a dev, this is the version I'd actually respect. Filing an exception is a normal PR, I can see who approved it, and the expiry date isn't something someone has to remember. When it runs out, the bot tells me, not my manager in a meeting.

**The same idea works for the other two Phase 0 records.** Keep your SAMM baseline (the [SAMM Toolbox](https://github.com/owaspsamm/toolbox-spreadsheet) spreadsheet from Step 4) and your decision records (the template is [in the repo too](https://github.com/hijacksecurity/appsec-playbook-templates/blob/main/operating-model/decision-record-template.md)) in Git, so next year you can see what changed and why.

## What Good Looks Like

| | Crawl | Walk | Run |
|---|---|---|---|
| **Charter** | One page; SOC and platform boundaries written down | Reviewed yearly; used to settle disputes | Boundaries tested in exercises; changes go through decision records |
| **Ownership** | Security, engineering and business roles named | RACI in use; SLA escalation goes to eng leadership | Risk acceptance is routine, by the service's risk owner, with data |
| **Operating model** | Central team, a few friendly teams | Hub-and-spoke; champions on every Tier 1 team with agreed time | Champions ship paved-road improvements back to the hub |
| **Exceptions** | A form, an approver and an expiry date | Owner, compensating control, expiry, approver by priority; reported with findings | Auto-reopen on expiry; renewals escalate; exception rate falls |
| **Roadmap** | A list of projects | Tied to obligations and a SAMM gap | Tied to obligations, velocity and risk; funded as a multi-quarter plan |

## Anti-Patterns

- **Trying to do everything at once.** A 40-item first-year plan. Pick the few that unblock the rest; defer the others out loud.
- **Policy before paved road.** "All services must scan dependencies," before there's a one-line way to do it, gets you exception requests, not security.
- **Buying a platform before knowing the inventory.** You can't judge coverage of assets you haven't found. Inventory first (3.2).
- **Measuring findings found.** It rewards noisy scanners. Measure risk reduced and time to fix (3.6).
- **The permanent exception.** No expiry, no renewal limit, no visibility: the shadow backlog.
- **No line drawn with the SOC.** Every advisory turns into a negotiation while the clock runs.

## Metrics for This Phase

Phase 0 metrics show whether the operating model exists and works, not whether the org is secure yet. <span data-cs="/2026/10/07/appsec-program-playbook-measure-what-matters/">3.6</span> *(coming soon)* defines the full metric set.

| Metric | Formula | Why it matters |
|---|---|---|
| **Ownership coverage** | Tier 1 assets with a named owning team ÷ all Tier 1 assets | Findings without owners don't get fixed (3.2) |
| **Champion coverage** | Tier 1 teams with an active champion ÷ Tier 1 teams | Shows whether the spokes exist |
| **Exceptions past expiry** | Open exceptions past expiry ÷ open exceptions | Catches the shadow backlog; target is zero |
| **Exception renewal rate** | Of exceptions closed this period, the share renewed at least once | High means exceptions are replacing fixes |
| **Time to "are we affected?"** | Advisory published → exposure answer, median | First test of the SOC line and the inventory |

## Takeaways and Templates

### The Charter (One Page)

```markdown
# Pinecart Application Security Charter

**Mission.** Make the secure way the easy way for Pinecart engineering, so every
team gets security without extra work.

**Scope.** Software Pinecart builds and ships: application code, dependencies, IaC,
container images, CI/CD pipelines, the dev platform, and AI tools in the SDLC.

**AppSec owns.** Secure SDLC standards; the shared pipeline security stages and
scanner configuration; triage and prioritization; exposure analysis for advisories;
the security champions program; pipeline control evidence.

**AppSec influences.** Framework and library choices; CI runner hardening; cloud
guardrails; detection content for application-layer attacks.

**Not AppSec.** Incident command and forensics (SOC/IR); cloud runtime posture
(cloud security); the control framework and audits (GRC); regulatory interpretation
and notifications (legal/privacy).

**Ownership principle.** Security owns visibility and guidance. Engineering owns the
fix. The business owns risk acceptance.

**Operating model.** Central AppSec hub + a security champion in every Tier 1 team,
with <X>% of the champion's time agreed with their manager.

**Decision rights.** Two-way-door decisions: AppSec team. One-way-door decisions:
AppSec lead + engineering leadership, recorded as a decision record with a revisit
date. Risk acceptance follows the exception policy, not these rights.

**Exceptions.** Per the risk-acceptance policy: the service's risk owner accepts,
security reviews, maximum expiry by priority. KEV is never excepted.

**Success measures.** Ownership coverage, SLA adherence by priority, exceptions past
expiry, SAMM delta. Reported quarterly.

**Sponsor.** <exec sponsor>.

**Review.** Yearly, or when the org changes shape.
```

### The Exception Record and Decision Record

Both templates are in the [appsec-playbook-templates](https://github.com/hijacksecurity/appsec-playbook-templates/tree/main/operating-model) repo, with the working exception check.

### The Roadmap One-Pager

| Section | Content |
|---|---|
| **Where we are** | SAMM baseline (one chart), three baseline numbers, top three risks in plain language |
| **Where we're going (12 months)** | SAMM targets per practice, and the two or three practices deferred on purpose |
| **Why** | Each theme mapped to an obligation (SOC 2, PCI DSS v4.0.1, CRA, SEC) *and* to what it gives engineering back |
| **Quarter by quarter** | Three to five themes per quarter, each with an owner and a definition of done |
| **Asks** | Headcount, budget, and decisions leadership must make, as options with cost and risk |
| **What we're not doing** | The deferred list, with the reason and the revisit date |

## Next

With a charter, owners and a 90-day plan, the program has a shape. But it can't see yet. Every step here assumed you could answer "what do we have, and who owns it?" Most organizations can't, not at the start. The next part covers building the inventory everything else depends on, including the ownership mess nobody warns you about.

**Next:** [3.2 The AppSec Program Playbook: See Everything - Inventory, Ownership & Attack Surface](/2026/10/07/appsec-program-playbook-see-everything/)
