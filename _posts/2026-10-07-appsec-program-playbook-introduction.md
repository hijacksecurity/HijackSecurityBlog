---
layout: post
title: "3.0 The AppSec Program Playbook: A Program, Not a Scanner"
date: 2026-10-07 10:00:00 -0000
categories: appsec security devsecops
tags: ["The AppSec Program Playbook", "DevSecOps", "Leadership", "Supply Chain"]
series: "The AppSec Program Playbook"
series_part: "3.0"
---

Most AppSec programs don't fail with a breach headline. They fail slowly, one skipped ticket at a time.

I've seen apps go years without a single patch. Nobody touched them because they worked. Then one Critical lands, and the "one-line fix" needs a new framework, which needs a new runtime, which needs a new base image. Now it's a rewrite, and it's a much bigger problem than security.

<div style="text-align: center; margin: 30px 0;">
  <img src="/assets/images/playbook-3-0.png" alt="A lone figure in a long coat stands in a dark stone courtyard at dusk; on the left a crowd of hooded travelers waits before a shut iron gate, while ahead a lantern-lit road runs freely through a great arch into a glowing city" style="max-width: 100%; height: auto; border-radius: 8px; box-shadow: 0 2px 12px rgba(0,0,0,0.15);">
  <p style="font-size: 0.9em; color: #666; margin-top: 8px;"><em>A gate stops people. A road carries them.</em></p>
</div>

This series is about doing it differently. It's called **The AppSec Program Playbook**, and it's how I think an application security program should be built and run: the phases, the tradeoffs, the decisions, and what I'm learning as I run it. This post sets the frame: the thesis, the model, how the model maps to the standards you'll get asked about, and how to read the other seven parts.

## The Thesis

**An AppSec program is not a scanner. It's everything it takes to make your apps secure and keep them that way:** knowing what you have, making the secure way the easy way, fixing what matters first, and fixing it fast. Done right, security stops being the team that slows everyone down. It becomes something every team gets without asking.

Everything in this series comes back to one thing I've seen again and again:

> **Security gets done when it blocks the work, when it's effortless, or when someone owns it with a real deadline. Everything else gets skipped.**

Security matters. But to a developer trying to ship, it's extra work. So every security task ends up in one of four places:

1. **It blocks the work.** The build fails, the PR can't merge, the deploy stops. It gets done because it has to. But blocking only works when it's **rare and right**. Block too often, or block on noise, and developers find ways around it: exceptions pile up and nobody trusts the next finding. I learned this first-hand; 3.4 tells the story.
2. **It's effortless.** The fix shows up as a green PR. The secure option is already the default. The library already does the right thing. It gets done because it costs almost nothing. "Almost" matters here: someone still has to trust the tests before they merge, and an app that hasn't been updated in years can't be patched with one click. That's a planned upgrade.
3. **Someone owns it, with a deadline.** Some things can't be automated or blocked: a design flaw, an access-control bug from a pentest, a big legacy upgrade. These get done when they have a named owner, an SLA (service-level agreement, i.e. a deadline), and a number leadership actually looks at. That's why [OWASP SAMM (Software Assurance Maturity Model)](https://owaspsamm.org/model/implementation/defect-management/) treats tracking and metrics as a core practice.
4. **Everything else.** A ticket with no owner. A finding on a dashboard. A wiki page. A "please upgrade when you can" email. It gets skipped.

> **Dev lens:** I've been a developer for 15+ years. I do what the ticket asks, at minimum. A finding with no owner and no deadline? I'd skip it too. Not because I don't care, but because the sprint doesn't.

**The job of an AppSec program is to manage all four:** block only what's worth blocking, make most fixes effortless, give the judgment calls owners and deadlines, and keep shrinking the pile that gets skipped.

One more rule keeps this honest: **the goal is less risk, not more work.** If you make everything effortless but don't decide what matters, you just flood teams faster. Prioritization decides what gets blocked, what gets a deadline, and what gets fixed in bulk. That's 3.4.

The other thing people get wrong: security gets treated as someone else's job. It's everyone's job, but then developers are never given the time or the tools to do it. You have to fix both. It has to be easy, it has to get done, and developers have to care. Miss any of these and you end up in one of the two failure modes below.

## Two Ways Programs Fail

Most organizations start the same way. They buy a scanner, point it at every repo, and get thousands of findings. From there, most programs get stuck in one of two traps.

### Failure Mode 1: Security as a Gate

The first reaction is to enforce. Fail the build on any Critical or High. It feels responsible, and for a few weeks it looks like progress.

Then developers start hitting it. A team gets blocked three times in one week by Criticals in libraries their code never calls. A release slips because of a finding in a test dependency. The scanner flags a "secret" that is a documented example value. Each one is small. Together they teach everyone one lesson: **security is an obstacle you work around.**

So people work around it. Exceptions get filed in bulk and approved in bulk. Pipelines quietly get a `continue-on-error`. New services launch outside the pipeline that has the gate. The gate is still there and the metrics say it's enforcing, but the risk has moved somewhere the gate can't see.

A gate built on scanner severity blocks the wrong things often enough that people stop believing it when it blocks the right ones. <span data-cs="/2026/10/07/appsec-program-playbook-prioritize-ruthlessly/">3.4</span> *(coming soon)* is all about fixing that.

### Failure Mode 2: Security as a Report

The opposite reaction is to avoid friction completely. Turn the scanner on, send findings to a dashboard, publish a monthly report.

Nobody gets blocked. Nobody complains. Nothing gets fixed either, because every finding lands in the fourth bucket. Leadership watches the number grow. Security says it *told* everyone. Engineering says it has no time. Everyone is right, and nothing moves.

This failure is quieter than the gate. I think it's more dangerous, because it looks like governance: there's a policy, a tool and a monthly meeting. What's missing is anything that turns a finding into a fix.

### The Same Root Cause

Both make the same mistake: **they treat the scanner as the program.** The gate hands judgment to the scanner's severity field. The report hands action to whoever happens to read it. Neither has a feedback loop, an owner for each asset, a way to decide what matters, or a way to make fixing cheap.

A scanner is a sensor. The program is everything around it: who owns what, what a finding means for *this* asset, what happens next, how the fix gets cheaper next time, and how you know it's working.

## What Skipping Really Costs

This is the part people underestimate, and it's what the opening is about.

Skipped security work doesn't stay the same size. **Security debt turns into technical debt.** A dependency one minor version behind is a five-minute bump. Two majors behind, it's an afternoon of API changes. Leave it for years and the dependency drags in its framework, the framework drags in the runtime, the runtime drags in the base image, and the "patch" becomes a rewrite. By then it's much bigger than a security problem. The service can't take a new feature cleanly, can't move to new infrastructure, and nobody wants to work on it.

That's why patching has to be continuous and boring, with a cooldown. New releases wait a minimum release age before you take them (the [FBI's July 2026 FLASH](https://www.ic3.gov/CSA/2026/260702.pdf) recommends seven days), and security fixes skip the wait. That way "stay current" never means "install the worm first." Small, frequent updates keep every future fix small, and emergency patching gets easy because you're always close to the fix. <span data-cs="/2026/10/07/appsec-program-playbook-remediate-at-scale/">3.5</span> *(coming soon)* comes back to this as continuous dependency hygiene. It's one of the best returns a program can get.

## The Alternative: Security by Default

So if the scanner isn't the program, what is?

My model comes from platform engineering. **Security is a product, engineering teams are its customers, and the best security control is one teams get without doing anything.** A team that turns on the paved-road pipeline stages gets secret scanning, dependency scanning, IaC checks, container checks, and whatever gets added next quarter. A team that uses the approved HTTP client gets [SSRF protection](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html), backed up by network egress controls. A team on the standard base image gets someone else's patch schedule. They did no security work. They got it for free.

This is how work moves out of the fourth bucket and into "effortless." It's also where the industry has been pushing for years. CISA's [Secure by Design](https://www.cisa.gov/securebydesign) initiative asks software makers to ship products that are secure out of the box and to treat security as a core business requirement, not a feature. The same idea works inside a company. The platform and security teams are the "manufacturer," and product teams are customers who deserve secure defaults.

For teams to get a control for free, it needs three things:

| Property | What it means | Example |
|---|---|---|
| **Secure by default** | The easy path is the secure path. Opting out takes effort; opting in takes none. | A pipeline template with scanning on; a framework config with CSRF protection on |
| **Self-serve** | Teams adopt it without a ticket or waiting on security | Three lines to include a reusable workflow; docs that answer the common questions |
| **Upgraded centrally** | Security improves it once and every team benefits | Security updates the shared pipeline stages once; every team gets the new check on its next run |

Gates still exist in this model. They're just narrow, earned and easy to explain:

- a verified live secret, blocked at push time (once it's pushed, the fix is rotation, not a failed merge);
- anything actively exploited in the wild, whatever its CVSS (Common Vulnerability Scoring System) score;
- an exposed or high-impact finding that is reachable, or can't be shown to be unreachable;
- a policy violation on a crown-jewel service.

Everything else warns, goes to an owner, or gets fixed by automation. <span data-cs="/2026/10/07/appsec-program-playbook-paved-road/">3.3</span> *(coming soon)* covers building the road; 3.4 covers what's allowed to block it.

### How I'm Building It

In the program I'm running, the order matters more than any one tool. Roughly:

1. **Make the tooling real.** Scans that run reliably and find things people can trust.
2. **Automate delivery.** Findings go where developers already work, not to a separate portal.
3. **Get visibility.** Inventory and ownership, so every finding has a home.
4. **Go to the teams.** Sit with them, fix something together, learn what hurts.
5. **Make fixing easy.** Automated patching (an agentic service that turns findings into tested, ready-to-merge PRs, plus grouped dependency updates), fix guidance, and paved-road libraries.
6. **Then set standards.** Enforced mandates come last, once there's a road to follow. Baseline non-negotiables and the compliance rules you already have to meet get written down from day one.

If I could keep one lesson, it's this: **people adopt what makes their week easier.** The teams that pushed back hardest on security changed their minds when security showed up, fixed something with them, and then took recurring work off their plate with automated patching. Across thousands of repos, that did more than any mandate. [3.1](/2026/10/07/appsec-program-playbook-operating-model/) goes deep on the operating model behind it.

## AppSec's Scope Got Much Bigger

One more change is big enough to affect how you staff and plan a program.

Not long ago, "application security" meant the application: the code a team wrote and the libraries it pulled in. Infrastructure was a separate world of servers someone racked or requested. That world is mostly gone. Infrastructure is now code, owned by developers and DevSecOps engineers: Terraform modules, Kubernetes manifests, Helm charts, Dockerfiles, and the CI/CD pipelines that build and ship all of it. Nobody sets up a server by hand any more. They merge a PR. And the tools developers use every day, from the SCM platform to the IDE to the AI assistant, are now part of the attack surface too.

So in my view, AppSec now owns, or at least has to secure, all of this:

- **Infrastructure as code:** Terraform, Kubernetes manifests, Helm charts, policy as code.
- **Containers:** Dockerfiles, base images, image provenance and admission.
- **CI/CD pipelines:** workflow permissions, third-party actions, runner hygiene, publish credentials.
- **The software supply chain:** every open-source package, action, extension and AI tool that goes into the build, plus the packages you publish yourself.
- **The dev platform itself:** GitHub, GitLab or Azure DevOps. Org settings, SSO, branch protection and rulesets, who can create tokens, which third-party apps have access, secret scanning and push protection, audit logs. If someone takes over your SCM org, they don't need a code bug. The [OpenSSF SCM Platform Configuration Best Practices](https://best.openssf.org/SCM-BestPractices/) guide and the [CIS Software Supply Chain Security Guide](https://www.cisecurity.org/insights/white-papers/cis-software-supply-chain-security-guide) give you a checklist, and open-source scanners like [Legitify](https://github.com/Legit-Labs/legitify) check it for you.
- **Developer machines and IDEs:** laptops hold tokens, SSH keys and cloud credentials, and IDE extensions run with the developer's access. [GlassWorm](https://blogs.eclipse.org/post/mika%C3%ABl-barbero/open-vsx-security-update-october-2025) spread through extensions published with leaked tokens.
- **The AI toolset:** coding assistants, agents, AI CLIs and MCP (Model Context Protocol) servers. They read your code, hold credentials and can run commands. In the [Nx "s1ngularity" attack](https://nx.dev/blog/s1ngularity-postmortem), the malware used the developer's own AI CLIs to hunt for secrets. The [OWASP Top 10 for Agentic Applications](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) covers this new surface. 3.7 goes deep on it.
- **Secrets, across all of the above:** they leak from code, CI logs, laptops and AI prompts alike.

That list is my own view, not one standard's headline, but the standards point the same way:

- The [OWASP Top 10:2025](https://top10.owasp.org/2025/A03_2025-Software_Supply_Chain_Failures) added **A03 Software Supply Chain Failures**. It widens the old "known vulnerable components" category to "all supply chain failures, not just ones involving known vulnerabilities." It came top in the community survey, with exactly 50% of respondents ranking it first.
- The [OWASP Top 10 CI/CD Security Risks](https://owasp.org/www-project-top-10-ci-cd-security-risks/) treats the pipeline as attack surface in its own right.
- [NIST SP 800-204D](https://csrc.nist.gov/pubs/sp/800/204/d/final) puts supply-chain controls inside the DevSecOps pipeline and maps them to the SSDF (Secure Software Development Framework).
- The joint NSA/CISA guidance on [Defending CI/CD Environments](https://media.defense.gov/2023/Jun/28/2003249466/-1/-1/0/CSI_DEFENDING_CI_CD_ENVIRONMENTS.PDF) says plainly that CI/CD environments are targets.

### Supply Chain Is a Top-Tier Threat

The incidents make the case better than any framework. Two big ones from 2025, and a much busier 2026, all from primary sources:

| Date | Incident | What it showed |
|---|---|---|
| 2025-03-14 | **tj-actions/changed-files** ([CISA](https://www.cisa.gov/news-events/alerts/2025/03/18/supply-chain-compromise-third-party-tj-actionschanged-files-cve-2025-30066-and-reviewdogaction)) | Attackers moved the action's version tags to a malicious commit. Every workflow using a mutable tag ran code that dumped CI secrets into build logs. **Your pipeline runs other people's code.** |
| 2025-09 | **Shai-Hulud**, a self-replicating npm worm ([CISA](https://www.cisa.gov/news-events/alerts/2025/09/23/widespread-supply-chain-compromise-impacting-npm-ecosystem)) | A postinstall payload stole npm tokens and secrets, then republished the victim's *other* packages with itself inside. Over 500 packages. **One stolen credential became a worm.** |
| 2026-03-19 onward | **TeamPCP: Trivy, then KICS, then LiteLLM** ([Aqua advisory](https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23), [FBI FLASH](https://www.ic3.gov/CSA/2026/260702.pdf), [Unit 42](https://unit42.paloaltonetworks.com/teampcp-supply-chain-attacks/)) | A bot token left over from incomplete credential rotation let attackers force-push the `trivy-action` tags. According to Unit 42's analysis, stolen CI secrets were then chained into the KICS action, then LiteLLM on PyPI. **Security scanners were the way in.** |
| 2026-03-31 | **axios** ([CISA](https://www.cisa.gov/news-events/alerts/2026/04/20/supply-chain-compromise-impacts-axios-node-package-manager), [postmortem](https://github.com/axios/axios/issues/10636)) | The lead maintainer's own PC was compromised through targeted social engineering. The attacker published versions with a malicious dependency that dropped a remote-access trojan. About 3 hours live on a package with 100M+ weekly downloads. **A maintainer's laptop is part of your supply chain.** |
| 2026-05-11 | **TanStack / "Mini Shai-Hulud"** ([TanStack postmortem](https://tanstack.com/blog/npm-supply-chain-compromise-postmortem)) | A `pull_request_target` pwn request, Actions cache poisoning, and an OIDC (OpenID Connect) token read from runner memory. 84 malicious versions across 42 packages in six minutes, **published from TanStack's own release pipeline**, so they looked like real releases. Proof of where a package was built isn't proof the build was safe. |
| 2026-08-04 | **"ChainDrop" (Shai-Hulud returns)**, starting with keyv ([CSA Singapore](https://www.csa.gov.sg/alerts-and-advisories/advisories/ad-2026-009/)) | One compromised maintainer account for `keyv`. The worm hunted for tokens allowed to publish **without 2FA**, then spread to 400+ packages and 1,300+ versions with about 2 billion monthly downloads between them. It also planted persistence in AI agent config (`.claude/settings.json`). **The worms keep coming back, bigger.** |

Three things stand out:

- **These attacks are wormable.** Stolen publish tokens let a payload spread to everyone downstream, across ecosystems, in minutes. And 2026 shows the worms coming back in new versions, not going away.
- **The pipeline is the way in.** Script injection, mutable tags, over-privileged tokens and cache poisoning show up again and again.
- **Developer machines and AI tools are now targets.** A maintainer's laptop (axios), and persistence planted in AI agent config (TanStack, ChainDrop).

None of it is a "code vulnerability" in the classic sense, and all of it lands on AppSec's desk.

Two more lessons from the same incidents:

- **Normal 2FA doesn't stop real-time phishing.** In September 2025 a maintainer of `debug` and `chalk` typed a live one-time code into a fake npm site. CISA and the FBI now push for phishing-resistant MFA (security keys, passkeys).
- **Victims usually find out from someone else.** TanStack's postmortem says it plainly: "No internal alerting." Outside researchers spotted it about 20 minutes after publish. That's why the advisory-to-response loop in 3.5 starts with outside threat intel.

The ecosystem has moved fast. npm [revoked classic tokens](https://github.blog/changelog/2025-12-09-npm-classic-tokens-revoked-session-based-auth-and-cli-token-management-now-available/) in December 2025, added staged publishing in May 2026, and [npm v12](https://github.blog/changelog/2026-07-08-npm-install-time-security-and-gat-bypass2fa-deprecation/) turned dependency lifecycle scripts off by default. Dependabot now waits 3 days before proposing a new version by default. Trusted publishing, minimum release ages, SHA-pinned actions and install-script allowlists went from niche to mainstream. The FBI's July 2026 FLASH even recommends a seven-day minimum package age. These controls are now a big part of the paved road.

Regulation caught up too: EU Cyber Resilience Act reporting duties have applied since September 11, 2026, and CISA's 2026 SBOM (software bill of materials) minimum elements replaced the 2021 NTIA baseline.

I chose not to give supply chain its own part. It runs through the phases instead, because that's how you actually defend against it:

- **3.2** covers SBOMs and inventory of IaC, images, workflows and developer tooling.
- **3.3** covers the controls: trusted publishing, pinning, minimum release age, install-script blocking, and provenance and signing.
- **3.5** covers what to do when a package you depend on is compromised, as a rehearsed campaign instead of a scramble.

## The Phase Model: A Loop, not a Waterfall

The series is built around seven phases. They're numbered because you have to start somewhere, but they form a loop. Each phase feeds the next, and later phases feed back into earlier ones. If you run them once, in order, and declare victory, you've run a project, not a program.

<div class="mermaid">
graph TB
    P0["Phase 0 · Operating Model"] --> P1["Phase 1 · See Everything"]
    P1 --> P2["Phase 2 · Paved Road"]
    P2 --> P3["Phase 3 · Prioritize"]
    P3 --> P4["Phase 4 · Remediate"]
    P4 --> P5a["Phase 5a · Measure"]
    P5a --> P5b["Phase 5b · Evolve"]
    P4 -.->|fix at the source| P2
    TI(["Threat intel:<br/>are we affected?"]) -.->|skip the line| P3
    P5b -.-> R(["Repeat: yearly reset, back to Phase 0"])
</div>

| Phase | Name | The question it answers | Part |
|---|---|---|---|
| 0 | Operating Model | Who owns what, what do we do first, and what are we choosing *not* to do? | 3.1 |
| 1 | See Everything | What exists, who owns it, how critical is it, and what is exposed? | 3.2 |
| 2 | Paved Road | How do we make the secure path the easiest one, and test that it holds (dynamic testing or DAST, pentests, bug bounty)? | 3.3 |
| 3 | Prioritize | Of everything we found, what really matters, and what is allowed to block? | 3.4 |
| 4 | Remediate | How do we turn findings into fixed risk at a pace engineering can keep up? | 3.5 |
| 5a | Measure | Is it working, is it getting cheaper, and how do we tell leadership? | 3.6 |
| 5b | Evolve | How does each phase stretch to cover LLM apps, agents and AI-written code? | 3.7 |

The loops are where the payoff grows over time:

- **Remediate → Paved Road.** Fix something once in a shared library, base image or template, and the finding closes for every team that uses it. That's how you fix classes, not instances.
- **Measure → Paved Road.** The most honest program metric is recurrence. When the same bug class keeps coming back, turn it into a guardrail: a lint rule, a template change, a policy. The next team never writes that bug.
- **Measure → Operating Model.** Once a year, reassess maturity and refresh the roadmap.
- **Threat intel → See and Prioritize.** When a package or action is compromised, the inventory has to answer "are we affected?" in minutes. Active exploitation overrides every other prioritization rule.
- **Evolve → everything.** AI doesn't get its own silo. It gets inventoried, gets a paved road, gets prioritized and gets remediated like everything else. That's why 3.7 walks through each phase again.

## Mapping to the Standards

Auditors, customer security questionnaires and leadership will all ask how your program maps to a framework. Two standards do most of the work. A third is useful for comparison.

- **[OWASP SAMM](https://owaspsamm.org/model/)** (v2) is a prescriptive maturity model. It has **five business functions** (Governance, Design, Implementation, Verification and Operations), each with three security practices, so fifteen in total. It's built for self-assessment and roadmapping, which is why 3.1 uses it for the baseline.
- **[NIST SSDF](https://csrc.nist.gov/pubs/sp/800/218/final)** (SP 800-218, v1.1, February 2022) is a set of outcome-based practices in **four groups**: **Prepare the Organization (PO)**, **Protect the Software (PS)**, **Produce Well-Secured Software (PW)** and **Respond to Vulnerabilities (RV)**. It's the language US government buyers use. NIST published [an initial public draft of SSDF v1.2](https://www.nist.gov/news-events/news/2025/12/secure-software-development-framework-ssdf-version-12-available-public) ([SP 800-218r1 ipd](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-218r1.ipd.pdf)) on December 17, 2025. As I write, v1.1 is still the final version. There's also [SP 800-218A](https://csrc.nist.gov/pubs/sp/800/218/a/final), the SSDF community profile for generative AI, which 3.7 uses.
- **[BSIMM](https://www.blackduck.com/resources/analyst-reports/bsimm.html)** describes instead of prescribes: it records what real companies' programs actually do. BSIMM16 came out in early 2026. Use it to compare yourself with peers, not as a to-do list.

Here's how the phases map. The mapping is mine. Neither framework is organized by phase, and most practices touch more than one. Use it to find your way around the standard, not as a compliance crosswalk.

| Phase | OWASP SAMM v2 (business function → practices) | NIST SSDF v1.1 (practice group → practices) |
|---|---|---|
| **0 · Operating Model** | **Governance** → Strategy & Metrics, Policy & Compliance, Education & Guidance | **PO** → PO.1 Define security requirements, PO.2 Implement roles and responsibilities |
| **1 · See Everything** | **Design** → Threat Assessment (application risk profile), Secure Architecture (technology management); **Implementation** → Secure Build (software dependencies) | **PW** → PW.4 Reuse well-secured software (component tracking); **PS** → PS.3 Archive and protect each release (including component data such as SBOMs) |
| **2 · Paved Road** | **Implementation** → Secure Build, Secure Deployment; **Design** → Security Requirements, Secure Architecture; **Verification** → Security Testing | **PW** → PW.1, PW.2, PW.4, PW.5, PW.6, PW.7, PW.8, PW.9 (secure design, design review, reuse, coding, build, review, test, secure defaults); **PS** → PS.1 Protect code, PS.2 Release integrity; **PO** → PO.5 Secure environments |
| **3 · Prioritize** | **Implementation** → Defect Management; **Design** → Threat Assessment | **RV** → RV.2 Assess, prioritize and remediate; **PO** → PO.4 Criteria for security checks |
| **4 · Remediate** | **Implementation** → Defect Management; **Operations** → Incident Management, Environment Management | **RV** → RV.1 Identify and confirm vulnerabilities, RV.2 Assess, prioritize and remediate |
| **5a · Measure** | **Governance** → Strategy & Metrics; **Implementation** → Defect Management (metrics and feedback) | **RV** → RV.3 Root-cause analysis; **PO** → PO.4 Criteria for security checks |
| **5b · Evolve** | All five functions, applied again to AI components; especially **Governance** → Policy & Compliance and **Design** → Threat Assessment | All four groups, via **SP 800-218A** (the SSDF profile for generative AI) |

Two practical notes:

- **Don't let the mapping become the program.** It's tempting to run a SAMM assessment, find the lowest scores, and call that the roadmap. Low scores show where you're immature, not where the risk is. The operating model in 3.1 uses SAMM to find the *gaps* and business risk to set the *order*.
- **Most of the paved road lives in SSDF's PW group. Most of the pain lives in RV.** If your program is mostly RV (finding and chasing), you're in the report failure mode. Moving effort from RV to PW means moving work from the fourth bucket to the effortless one.

## Crawl, Walk, Run

Every part ends with a crawl / walk / run view of its phase. Here they all are in one table, so you can see where you stand today. Most programs I've seen sit at different levels in different phases, and that's normal. The goal isn't "run" everywhere. It's no phase stuck at "crawl" holding the others back.

| Phase | Crawl | Walk | Run |
|---|---|---|---|
| **0 · Operating Model** | One security person, a scanner, no charter. Exceptions by email. | Written charter and RACI (responsible, accountable, consulted, informed): security owns visibility, engineering owns the fix, the business owns risk acceptance. Exceptions have an approver and an expiry date. | Hub-and-spoke with active security champions; roadmap tied to business risk and obligations; SAMM reassessed every year. |
| **1 · See Everything** | A spreadsheet of "important apps." Findings with no owner. | Automated repo inventory with owners from a service catalog or CODEOWNERS; tiers by criticality and exposure; SBOMs for Tier 1. | Code → artifact → deployment linked; IaC, images, workflows and developer tooling in the inventory; "are we affected by X?" answered in minutes. |
| **2 · Paved Road** | Scanners run centrally on a schedule; results go to a dashboard. | Security runs the pipeline stages as a service (main = prod, a test branch for new features), default for new services; warn first, block narrowly; new-code baseline. | Required for Tier 1; approved libraries remove whole bug classes; supply-chain controls (pinning, minimum release age, trusted publishing) on by default. |
| **3 · Prioritize** | Scanner severity is the priority. Gate on Critical/High. | Narrower gate plus exploitation signals: KEV (CISA's Known Exploited Vulnerabilities catalog) and EPSS (Exploit Prediction Scoring System), plus reachability where available; a hygiene lane so nothing gets silently dropped. | Full risk-based prioritization using reachability, exploitation, exposure and impact; AI gathers the context and a human checks it. |
| **4 · Remediate** | Security files tickets one finding at a time. | Findings go to owning teams with SLAs; grouped automated dependency updates; variant analysis for recurring bugs. | Fix at the source (base images, shared libraries, templates); safe auto-merge by risk, after a minimum release age; a rehearsed supply-chain incident playbook; agentic fixes behind a human gate. |
| **5a · Measure** | Count of open findings. | Coverage, MTTR by class and SLA adherence, reported per team. | Recurrence rate drives the paved-road roadmap; security and delivery metrics reported together; a QBR leadership actually uses. |
| **5b · Evolve** | AI use is banned or ignored. | An inventory of models, agents and MCP servers; a written agentic-AI standard. | AI components on the paved road: approved gateways, scoped agent identity, tool allowlists, evals and red-teaming before release. |

One honest caveat: the "run" column for Prioritize is the target state. In 3.4 I say clearly which rungs of that ladder I've run in production, and which are the model I've developed and practiced but not yet rolled out.

## Demo: The Pinecart Lab

Frameworks are easy to nod along to. So every part of the series is shown at the same company.

**Pinecart is a fictional e-commerce SaaS company.** It runs storefronts, checkout and payments, customer accounts and public APIs. It has about 400 engineers and a few thousand repos, a realistic mix of old and new services, and all the problems that come with that: orphaned repos, a legacy service nobody wants to touch, a payments path that has to meet PCI DSS, and more open findings than anyone can read. **The company is made up. The problems aren't.** Every scenario is built from things I've dealt with and seen in real AppSec work, mixed with industry best practice. A fictional company lets me show real situations in full detail, without pointing at any real team, system or person.

Pinecart isn't only on paper. It runs as a containerized lab: a slice of about a dozen repos in several languages, with owners, tiers and planted problems, plus an isolated network for anything vulnerable. Every screenshot in the series comes from the lab, never from a real system. Each post explains its ideas fully on its own, so you don't need the lab to follow along.

The lab uses open-source and free tools. The series has one rule for tools: **name the category first, give at least two options, and never treat a tool as the answer.** Here's the toolbox by category:

| Category | Examples in the lab (open source unless noted) | Where it shows up |
|---|---|---|
| Inventory and ownership | Backstage, CODEOWNERS files; OpenSSF Scorecard for repo posture | 3.2 |
| SBOM generation and analysis | Syft, OWASP Dependency-Track | 3.2, 3.5 |
| Secret detection | Gitleaks, TruffleHog | 3.3 |
| Static application security testing (SAST) | Semgrep CE, gosec, CodeQL CLI (free for open source and research; private code needs a GitHub license) | 3.3, 3.5 |
| Software composition analysis (SCA) and reachability | OSV-Scanner, govulncheck, Grype | 3.3, 3.4 |
| Container and IaC scanning | Trivy, Checkov | 3.3 |
| Dynamic testing (DAST) | ZAP, Nuclei | 3.3 |
| Policy as code | OPA/Conftest, Kyverno | 3.3 |
| Dependency updates | Renovate, Dependabot | 3.3, 3.5 |
| Findings management and deduplication | OWASP DefectDojo, Faraday Community | 3.4 |
| Exploitation signals | CISA KEV, FIRST EPSS | 3.4 |
| Signing and provenance | Sigstore cosign, SLSA provenance generators | 3.3 |
| Vulnerable targets | OWASP Juice Shop, crAPI, small services with planted bugs | throughout |

Remember the Trivy and KICS incidents above. **Pin and verify your security tools like any other dependency.** A scanner running in CI with access to your secrets is part of your supply chain.

## Why Listen to Me

Briefly, because the rest of the series should earn your trust on its own:

- **I've been a developer for 15+ years.** I've written code continuously since college, so I know what it's like to get a security ticket in the middle of a sprint. That's the "Dev lens" you'll see in every part.
- **I'm building and running an AppSec program.** Thousands of repos and thousands of fixes: inventory and ownership, a self-serve pipeline, prioritization, and remediation at scale, including agentic patching behind a human merge gate. The field notes in every part come from that work: what we tried, what broke, and what I'm changing.
- **I hold the GIAC Cloud Security Automation (GCSA) certification**, currently active, earned through SANS SEC540. I also **won the course CTF**, which came with the SEC540 challenge coin. Where a practice in this series really came from that course, I'll say so.
- **I built [Intercept](/2026/05/02/building-intercept-introduction/)**, an application security platform, under the Hijack Security name, and I keep adding features to it. SANS instructors have tested it. The 2.x series, starting with [2.0 Building Intercept: One Founder, Three AI Teams](/2026/05/02/building-intercept-introduction/), covers how it was built. This series doesn't discuss its internals.

You'll get the guide and the field notes side by side. The guide is what the standards and the industry say good looks like, with links. The field notes are the decisions I made and why.

## Anti-Patterns

Each part has its own list. These are the ones that sink a program before it starts.

- **Buying a platform before you know your inventory.** You can't judge coverage on an estate you haven't mapped.
- **Policy before paved road.** A standard that says "all services must do X," with no easy way to do X, just fills the fourth bucket.
- **Gating on scanner severity.** Severity isn't risk, as the [CVSS v4.0 User Guide](https://www.first.org/cvss/v4.0/user-guide) itself says. It blocks the wrong things often enough that everyone learns to work around the gate.
- **Measuring findings found.** It rewards noise and punishes the teams that fixed things.
- **Treating supply chain as someone else's problem.** Your CI pipeline, its actions, its tokens and your developers' tools are all attack surface.
- **Security owning the fix.** Security can't patch thousands of repos by hand, and when it tries, engineering stops caring.
- **Trying to do everything at once.** All seven phases, at "run," in the first quarter. Pick the phase that unblocks the others.

## Metrics for the Whole Program

3.6 goes deep on measurement. At this level, I'd want one headline signal per phase, just to know the loop is turning:

| Phase | Headline signal | What it tells you |
|---|---|---|
| 0 · Operating Model | Share of open exceptions with an owner, an approver and an expiry date not yet passed | Whether risk acceptance is real or a hidden backlog |
| 1 · See Everything | Share of assets with a known owner and a tier | Whether findings can be routed at all |
| 2 · Paved Road | Share of Tier 1 services running the pipeline security stages | Whether teams actually use the road |
| 3 · Prioritize | Share of blocking findings confirmed as true positives after triage | Whether the gate has earned trust |
| 4 · Remediate | Median time to remediate, by class and tier (not blended) | Whether fixes are getting cheaper |
| 5a · Measure | Recurrence rate of bug classes you already fixed | Whether prevention works |
| 5b · Evolve | Share of AI components inventoried and covered by the standard | Whether AI is inside the program or next to it |

## Takeaways

**The thesis in one line:** a program manages all four buckets. Most fixes become effortless, only real risk gets blocked, judgment calls get owners and deadlines, and the skipped pile keeps shrinking.

**Your program on one page.** Copy this and fill it in. Keep it short; 3.1 expands each part. The values show Pinecart as an example.

```yaml
# appsec-program.yaml: the program on one page
mission: Make the secure way the easy way for every engineering team.

scope:
  we_own:       [app code, dependencies, IaC, containers, CI/CD, supply chain, dev platform]
  we_influence: [cloud security, SOC / incident response, compliance]

who_does_what:
  finds_and_guides: AppSec
  fixes:            the team that owns the service
  accepts_risk:     business owner, by priority, always with an expiry date
  never_accepted:   actively exploited bugs (CISA KEV), leaked live secrets

standards:
  maturity_model: OWASP SAMM
  practices:      NIST SSDF (SP 800-218)
  benchmark:      BSIMM

review:
  progress: quarterly
  maturity: once a year
```

**Checklist: before you buy anything.**

- [ ] Can you list your repos, and who owns each one?
- [ ] Do you know which ten services would hurt most if compromised?
- [ ] Could a new service turn on your pipeline security stages in an afternoon?
- [ ] Is it written down who can accept risk, and for how long?
- [ ] Would you know within an hour whether a newly compromised package is in your builds?
- [ ] Do your CI workflows pin third-party actions to a commit SHA? ([OpenSSF Scorecard's Pinned-Dependencies check](https://github.com/ossf/scorecard/blob/main/docs/checks.md#pinned-dependencies) flags the ones that don't.)

If most answers are "no," a new scanner will give you a bigger number, not a better program. Start with 3.1 and 3.2.

---

A scanner finds problems. An application security program is everything around it: knowing what you have and who owns it, making the secure way the default, deciding what actually matters, getting it fixed fast, testing that it holds, and learning so the same bug doesn't come back.

**People, process and tools, working together, so the software you ship is secure without slowing down the teams that ship it.**

The rest of the series shows how to build one. If you're starting or inheriting a program, begin with 3.1. If you're drowning in findings, jump to <span data-cs="/2026/10/07/appsec-program-playbook-prioritize-ruthlessly/">3.4</span> *(coming soon)*. Every part is listed below.

**Next:** [3.1 The AppSec Program Playbook: Operating Model - Strategy, Ownership & the First 90 Days](/2026/10/07/appsec-program-playbook-operating-model/)
