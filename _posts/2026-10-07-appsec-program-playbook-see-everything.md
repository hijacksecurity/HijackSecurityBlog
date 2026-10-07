---
layout: post
title: "3.2 The AppSec Program Playbook: See Everything - Inventory, Ownership & Attack Surface"
date: 2026-10-07 12:00:00 -0000
categories: appsec security devsecops
tags: ["The AppSec Program Playbook", "Supply Chain", "Containers", "Infrastructure"]
series: "The AppSec Program Playbook"
series_part: "3.2"
---

It's 9:40 on a Monday. An advisory lands: someone repointed the tags of a popular GitHub Action to a malicious commit. Every workflow that uses it by tag has been dumping runner memory, secrets included, into its build logs.

Pinecart's AppSec lead asks one question in the engineering channel: *do we use it?*

The first answer is "probably." The second is a code search that times out halfway through the org. Someone starts a spreadsheet. By Wednesday they have about 60 repos, a third with no obvious owner. Two belong to a team that was removed in a reorg last spring. Nobody knows which of those workflows ran during the exposure window, or which secrets they held.

Now replay the same morning at a Pinecart that did the work in this post. The AppSec lead runs one query. Four minutes later she has every workflow that uses the action, at every ref, with its resolved commit, repo, tier and the owning team's on-call channel. The fan-out message to those teams goes out a few minutes later.

Same advisory both times. The difference was the inventory.

<div style="text-align: center; margin: 30px 0;">
  <img src="/assets/images/playbook-3-2.png" alt="A hooded figure holds up a lantern in a vast stone archive, facing a towering glowing city of countless lights and small banners, with shelves of ledgers lining the arches on both sides" style="max-width: 100%; height: auto; border-radius: 8px; box-shadow: 0 2px 12px rgba(0,0,0,0.15);">
  <p style="font-size: 0.9em; color: #666; margin-top: 8px;"><em>You can't secure what you can't see.</em></p>
</div>

*Pinecart is the series' fictional e-commerce company: about 400 engineers and a few thousand repos. The company is made up; the problems aren't. Every Pinecart number is illustrative.*

## The Principle: You Can't Secure What You Can't See or Route

Every later phase needs answers to four questions:

1. **What exists?**
2. **Who owns it?**
3. **How critical is it?**
4. **What is exposed, and where is it running?**

The paved road in <span data-cs="/2026/10/07/appsec-program-playbook-paved-road/">3.3</span> *(coming soon)* needs to know who has adopted it. Prioritization in <span data-cs="/2026/10/07/appsec-program-playbook-prioritize-ruthlessly/">3.4</span> *(coming soon)* needs exposure and impact. Remediation in <span data-cs="/2026/10/07/appsec-program-playbook-remediate-at-scale/">3.5</span> *(coming soon)* needs an owner for every finding and a fast "are we affected?" Metrics in <span data-cs="/2026/10/07/appsec-program-playbook-measure-what-matters/">3.6</span> *(coming soon)* need a denominator.

The principle I build around: **an inventory is a query engine, not a spreadsheet.** Its job is to answer real questions in minutes. The acceptance test: *"Are we affected by compromised package X, or action Y? Where, since when, and who owns it?"* If the inventory can't answer that within the first hour, it isn't done.

The standards agree. NIST's Secure Software Development Framework (SSDF, SP 800-218 v1.1) task [PS.3.2](https://csrc.nist.gov/pubs/sp/800/218/final) asks producers to "collect, safeguard, maintain, and share provenance data for all components of each software release (e.g., in a software bill of materials [SBOM])." One example: make that data available "to the organization's operations and response teams to aid them in mitigating software vulnerabilities." A [v1.2 draft](https://www.nist.gov/news-events/news/2025/12/secure-software-development-framework-ssdf-version-12-available-public) followed in December 2025 and is still a draft. The [CIS (Center for Internet Security) Controls v8.1](https://www.cisecurity.org/controls/cis-controls-list) puts enterprise and software asset inventory at Controls 1 and 2, because nothing downstream works without them. In OWASP SAMM (Software Assurance Maturity Model), this is [Secure Build → Software Dependencies](https://owaspsamm.org/model/implementation/secure-build/) (maturity 1: "create records with Bill of Materials of your applications") plus [Threat Assessment → Application Risk Profile](https://owaspsamm.org/model/design/threat-assessment/), which is where tiering lives.

## What "Everything" Means Now

After years of AppSec work, I'm sure of one thing: **AppSec's scope has exploded.** Infrastructure is now code (IaC, infrastructure as code), owned by developers and DevSecOps engineers. Nobody racks servers for a new service any more. They merge a Terraform module and a Helm chart. So Terraform, Kubernetes (K8s) manifests, Dockerfiles, base images, CI/CD (continuous integration and continuous delivery) pipelines and the software supply chain are all AppSec now.

> **An inventory that stops at "repos and their dependencies" is years out of date.**

The incidents since 2024 prove it. Attackers got in through:

- CI workflows ([tj-actions/changed-files](https://www.cisa.gov/news-events/alerts/2025/03/18/supply-chain-compromise-third-party-tj-actionschanged-files-cve-2025-30066-and-reviewdogaction), [Nx](https://nx.dev/blog/s1ngularity-postmortem), [TanStack](https://tanstack.com/blog/npm-supply-chain-compromise-postmortem));
- security scanners themselves ([Trivy](https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23), then KICS);
- IDE (integrated development environment) extensions ([GlassWorm](https://blogs.eclipse.org/post/mika%C3%ABl-barbero/open-vsx-security-update-october-2025));
- MCP (Model Context Protocol) servers ([postmark-mcp](https://postmarkapp.com/blog/information-regarding-malicious-postmark-mcp-package));
- AI libraries (LiteLLM).

The [FBI FLASH](https://www.ic3.gov/CSA/2026/260702.pdf) says the same actor modified Trivy, KICS and LiteLLM. [Unit 42](https://unit42.paloaltonetworks.com/teampcp-supply-chain-attacks/) reports that LiteLLM was reached with CI secrets stolen in the Trivy compromise. In 2026, payloads even planted persistence in `.claude/settings.json` and `.vscode/tasks.json` ([TanStack postmortem](https://tanstack.com/blog/npm-supply-chain-compromise-postmortem)). A classic application inventory sees none of this. These are the layers to cover:

<div class="mermaid">
graph TB
    subgraph Build["Source and build"]
        CODE[Code<br/>SCM orgs, repos, archived and dead repos]
        DEPS[Dependencies<br/>lockfiles, SBOMs]
        CICD[CI/CD and the dev platform<br/>workflows, actions, runners,<br/>installed apps and extensions]
        IAC[IaC<br/>Terraform, Helm, Kustomize,<br/>K8s YAML manifests]
    end
    subgraph Ship["Artifacts"]
        PKG[Packages we publish]
        IMG[Container images<br/>and their base images]
    end
    subgraph Run["Runtime"]
        DEP[Deployed services, endpoints,<br/>cloud accounts, clusters]
        EXT[External attack surface<br/>domains, IPs, exposed APIs]
    end
    subgraph Around["Around the development lifecycle"]
        SAAS[Third party and SaaS<br/>tools with code or data access]
        DEV[Developer tooling<br/>IDE extensions, AI/MCP tools, agent configs]
        AI[AI components, AI-BOM<br/>models, agents, MCP servers, gateways]
    end
    CODE --> DEPS
    CODE --> CICD
    CODE --> IAC
    CICD --> PKG
    CICD --> IMG
    IMG --> DEP
    IAC --> DEP
    DEP --> EXT
    DEV -.->|writes| CODE
    SAAS -.->|holds tokens to| CODE
    AI -.->|runs inside| DEP
</div>

| Layer | What to capture | Source of truth | Why it gets a row (real examples) |
|---|---|---|---|
| **Code** | Every repo, including archived, forks and dead ones; language, ecosystem, default branch, last push | Source code management (SCM) APIs: GitHub, GitLab, Azure DevOps | Everything else hangs off it |
| **Dependencies** | Components by [purl](https://github.com/package-url/purl-spec) (package URL), version, direct or transitive, scope | SBOMs generated at build; lockfiles | [axios](https://www.cisa.gov/news-events/alerts/2026/04/20/supply-chain-compromise-impacts-axios-node-package-manager) 1.14.1 was live for about 3 hours ([postmortem](https://github.com/axios/axios/issues/10636)); you need to know who pulled it |
| **CI/CD** | Workflows, every `uses:` reference (ref as written *and* resolved SHA), runner type, token permissions, secrets used | Workflow files; SCM dependency graph | tj-actions: versions ≤ 45.0.7 affected, with several tags repointed to one malicious commit ([GitHub advisory](https://github.com/advisories/GHSA-mrrh-fwg8-r2c3)). Trivy: 76 of 77 `trivy-action` tags force-pushed ([GitHub advisory](https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23)) |
| **Dev platform integrations** | Third-party apps and extensions installed on the platform itself: GitHub Apps and OAuth apps, Azure DevOps marketplace extensions, Jenkins plugins, GitLab integrations, org webhooks. For each: who installed it, its permissions and scopes, and which repos it can reach | Platform admin APIs, e.g. GitHub's [org app installations](https://docs.github.com/en/rest/orgs/orgs#list-app-installations-for-an-organization) and [org webhooks](https://docs.github.com/en/rest/orgs/webhooks#list-organization-webhooks), and Azure DevOps [installed extensions](https://learn.microsoft.com/en-us/rest/api/azure/devops/extensionmanagement/installed-extensions/list). The GitHub installations API lists GitHub Apps only; OAuth app access shows in the org's third-party access settings and audit log | In 2022, attackers used [OAuth tokens stolen from Heroku and Travis CI](https://github.blog/news-insights/company-news/security-alert-stolen-oauth-user-tokens/) to download private repos from dozens of GitHub orgs. An app with org-wide "read all repos" is a key to everything |
| **IaC** | Module sources and versions (registry, git ref), providers, charts, the cloud accounts they target, plus the workloads defined in Kubernetes manifests: images, privileges, service accounts, exposed ports | Terraform, Helm charts, Kustomize overlays, and plain Kubernetes (K8s) YAML manifests | Modules are dependencies too. A git-sourced module at `ref=main` works like an unpinned action: the code behind the ref can change. And raw K8s YAML is where `privileged: true` and `image: …:latest` hide |
| **Artifacts** | Image digest, base image name and digest, packages published, provenance | Registry, build metadata, OCI (Open Container Initiative) annotations | One base-image fix closes many findings (<span data-cs="/2026/10/07/appsec-program-playbook-remediate-at-scale/">3.5</span> *(coming soon)*), but only if you know who inherits from what |
| **Deployed** | Services, endpoints, running image digests, cloud accounts, clusters, namespaces | Cloud asset inventories, cluster API, deploy events | A finding in code needs to know if it's running, and where |
| **External surface** | Domains, subdomains, IPs, exposed APIs, certificates | DNS, certificate transparency, external discovery | Exposure drives priority in 3.4 |
| **Third party and SaaS** | Other SaaS tools with access to your code or data (outside the dev platform), and the OAuth grants people gave them | IdP (identity provider) app list, SaaS admin consoles | They hold tokens into your systems |
| **Developer tooling** | IDE extensions, AI coding assistants and their config, MCP servers developers run, local CLIs | Config committed to repos; endpoint management | GlassWorm (extensions), postmark-mcp (MCP), agent-config persistence (TanStack) |
| **AI components (AI-BOM, AI bill of materials)** | Models, providers, datasets, agents, MCP servers in products, AI gateways and SDKs (software development kits) | Code, config, gateways, SBOMs with ML-BOM (machine learning bill of materials) data | LiteLLM was hit in the same campaign as Trivy (reportedly through stolen CI secrets); AI SDKs are dependencies with an unusual blast radius |

You won't build all eleven layers at once (see crawl/walk/run below). But give every layer a slot in the data model from day one, because "are we affected?" doesn't care which layer got hit.

## The Tension: Complete, Fresh or Owned

**Completeness.** It's never complete. There's always another SCM org from a hackathon, a cloud account from an acquisition, a Lambda deployed from a laptop. Chasing 100% before you ship anything is why inventory projects fail.

**Freshness.** It goes stale the moment the crawl ends. A three-month-old inventory will confidently give the wrong answer in an incident, which is worse than an honest "unknown."

**Ownership.** Fresh and complete still can't drive fixes if many assets have no clear owner. **A finding without an owner never gets fixed.** It sits in a queue that belongs to everyone, so it belongs to no one.

You can't get all three. My tradeoff: **go broad and shallow first, make freshness automatic, and run ownership as a campaign that never ends**, not a field you fill in once. A thin record for every repo beats a rich record for a quarter of them, because the thin one at least tells you where to look. Add depth (SBOMs, deployment links, AI-BOM) layer by layer, Tier 1 first.

### Field Notes: Crawling Thousands of Repos

In my AppSec work I crawled thousands of repos. In the program I'm building, visibility comes third: make the tooling real, automate delivery, build visibility, then go to the teams.

A lesson from my prioritization work: function-level reachability works for some ecosystems, and for the rest the scanner simply can't tell (more in 3.4). So record **language and ecosystem per repo** from the first crawl. It shows you up front where you'll have strong signals, and where you'll have to rely on exposure and impact alone.

## How to Do It

### Step 1: Start with the Questions, Then the Minimum Data Model

Start from the questions the program must answer, not the fields a tool happens to export:

- Which services ship library X at version Y, and which of them run in production?
- Which workflows use action Y, at which refs, with which secrets?
- Who owns asset Z, and how sure are we?
- Which Tier 1 services face the internet, and what data do they touch?

Then define the smallest record that answers them (template below). The essentials: a **stable ID**, **layer**, **owner and owner confidence**, **tier**, **links to parent and child assets** (repo → image → deployment), **first seen / last seen / source**, and **status** (active, dormant, archived, decommissioned). Everyone forgets `last_seen` and `source`, and they're what make freshness measurable.

### Step 2: Discover at Scale

**Crawl the SCM first.** It's the cheapest, most complete source, and every other layer links back to it. At a few thousand repos, three things matter.

**Rate limits.** GitHub's REST API allows 5,000 requests an hour per authenticated user; a GitHub App installation scales from 5,000 up to 12,500 (15,000 on Enterprise Cloud). Secondary limits cap you at 100 concurrent requests ([GitHub rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)). Use a GitHub App, not a personal token: higher limits, and the credential doesn't leave with a person. GitLab and Azure DevOps have their own limits; check their docs.

**Start simple, then go incremental.** In the program I'm building, the inventory clones every repo once a week. It's simple, it's reliable, and it's a fine way to start. The tradeoff is freshness: the data can be up to a week old, and a full clone of thousands of repos gets heavier as you grow. When that starts to hurt, move to incremental sync:

- listen to org webhooks for new, deleted, archived or pushed repos;
- run one catch-up crawl a night that skips repos that haven't changed;
- deep-scan only what changed.

GitHub's [API best practices](https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api) recommend webhooks over polling for exactly this reason.

**Detect languages, frameworks and dead repos.** Record languages, package ecosystems (based on which manifests and lockfiles exist), Dockerfiles, IaC files and workflow files. For languages you don't need anything fancy: on GitHub, the [languages API](https://docs.github.com/en/rest/repos/repos#list-repository-languages) gives a breakdown per repo for free. On a cloned repo, [scc](https://github.com/boyter/scc) or [go-enry](https://github.com/go-enry/go-enry) (a port of GitHub's own detector) do the same. Then classify activity:

| Signal | Meaning |
|---|---|
| `archived: true` | Declared dead. Check it isn't still deployed. |
| No push in 12+ months (illustrative threshold) | Dormant |
| No releases, image builds or deploys in 12+ months | Probably not running |
| Used by a live deployment or as a dependency | **Not dead, whatever the commit history says** |

The last row is the one that causes trouble: "no commits" doesn't mean "not running." Apps left unpatched for years end up needing a full rewrite, and by then it's much bigger than a security problem: security debt compounds into tech debt. Never let the dead-repo classifier retire anything the deployment layer can still see.

### Step 3: Ownership, the Hardest Part

Ownership is a resolution problem. Several sources disagree, and each is wrong in its own way. Rank them by reliability, take the first confident answer, and record its source:

| Rank | Source | Strengths | How it fails |
|---|---|---|---|
| 1 | **Service catalog** entry: a small file in each repo that names the owning team, like `catalog-info.yaml` for [Backstage](https://backstage.io/docs/features/software-catalog/descriptor-format), the open-source developer portal ([see the example](#catalog-example)) | Explicit, reviewed, team-level | Only as fresh as the last reorg; rarely covers everything |
| 2 | **CODEOWNERS** default rule (`*`), with team handles | Lives with the code; auto-requested on PRs, but only enforced with branch protection or a ruleset ([GitHub docs](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)) | Individuals instead of teams; stale handles; path-level owners with no default |
| 3 | **Deployment metadata** (cloud account, K8s namespace, owner tags) | Shows who actually runs it | Shared accounts and platform namespaces |
| 4 | **SCM team permissions** (the team with admin or maintain) | Always there | Often an org-wide group, which means nobody |
| 5 | **Commit history heuristic** (active committers over 90 days, mapped to their team through the IdP) | Works on orphans | Contractors, drive-by fixes, bots; goes stale fast |
| 6 | **Default owner** (the parent org's engineering leader) plus the orphan queue | Always gives an answer | Only a temporary answer |

Rules that make this work:

- **Owners are teams, not people.** People leave; teams get renamed, which is still better. Check every owner against the IdP or HR (human resources) system on each sync. If the team is gone, the asset drops a rank.
- **Record confidence.** "High" means ranks 1 and 2 agree. "Medium" means one explicit source. "Low" means heuristic only. Report the confidence split, not just "% owned."
- **Route to where the owner already works.** Resolve each owner to a backlog and an on-call channel, not just a name. That's what makes the 3.5 fan-out possible.

**Orphans need a process.** An asset with no confident owner goes to the default owner, with a deadline to claim it, reassign it, or schedule it for archive. Run regular **archive campaigns**: send dormant, undeployed repos to their likely owners with an archive date. Archiving is reversible, so default to archive unless someone objects.

People adopt what makes their week easier, so skip the form. Open a pre-filled PR that adds a `catalog-info.yaml` or a CODEOWNERS default rule with your best-guess owner, and let the team approve or fix it.

> **Dev lens:** If security sends me a form asking which of 40 repos my team owns, it sits in my inbox until someone reminds me. A PR with the owner already filled in takes me ten seconds, so I approve it. If the guess is wrong, I fix it right away, because I don't want another team's findings landing in my backlog.

### Step 4: Tier by Criticality

Not every asset needs the same controls or the same deadlines (SLAs, service-level agreements). Sort each asset into one of three tiers by answering three simple questions:

1. **Does it matter to the business?** Is it on the path that makes money, or a key customer journey?
2. **What's the most sensitive data it touches?** In this series, "sensitive" means payment data, credentials and secrets, or personal data about many customers. A little personal data makes it Tier 2.
3. **Can attackers reach it?** Record three facts, using the same fields as 3.4:
   - `internet_facing`: can the internet reach it, directly or through a gateway?
   - `auth`: who can use it: `anonymous`, `any-customer`, `staff` or `service`? `auth` doesn't change the tier; 3.4 uses it.
   - `indirect`: does data from the internet reach it anyway? A batch job that processes customer uploads has no public endpoint, but it still handles attacker input.

Then use this table. **If the answers point to different tiers, the highest tier wins.**

| Tier | If any of these is true | Pinecart examples (illustrative) |
|---|---|---|
| **Tier 1** | Revenue-critical, **or** holds publish or deploy rights, **or** handles sensitive data (payment data, credentials and secrets, bulk personal data) | Checkout, payments, auth, public storefront API, the order-history service, the CI system itself |
| **Tier 2** | Customer-facing, **or** exposed (internet-facing or indirect), **or** handles limited personal data | Search, recommendations, the public docs site |
| **Tier 3** | None of the above: internal, not exposed, no personal or sensitive data, not on a critical path | Internal dashboards, internal docs, experiments |

For example, Pinecart's order-history service isn't part of checkout, but it stores names, addresses and purchases for every customer. That's personal data about many customers, so it's Tier 1.

**Why the highest tier wins.** If you averaged the answers, a payment service that isn't on the internet would land in Tier 2. That's wrong: it still handles payment data. The standards agree: [FIPS 199](https://csrc.nist.gov/pubs/fips/199/final) (NIST's Federal Information Processing Standard 199) calls this the "high water mark," and the [OWASP Risk Rating Methodology](https://community.owasp.org/OWASP_Risk_Rating_Methodology) says to rate impact by "using the worst-case option." One more rule, from recent supply-chain attacks: **anything that can publish packages or deploy to production is Tier 1**, even if users never see it.

> **Dev lens:** If Tier 1 means stricter checks and shorter deadlines, I'll argue my service is Tier 2. Every dev would. So don't tier on opinions. Tier on facts nobody can argue with, like "this service handles payment data" or "this service is on the internet."

**How tiers feed prioritization.** In <span data-cs="/2026/10/07/appsec-program-playbook-prioritize-ruthlessly/">3.4</span> *(coming soon)*, a finding on a Tier 1 asset starts at the `top` impact tier, Tier 2 at `middle`, and Tier 3 at `low`. Three rules keep this safe:

- **The tier is a starting point, not a limit.** If a finding on a Tier 3 asset turns out to touch credentials, deploy rights or sensitive data, the asset was tiered wrong. Re-tier the asset with the table above (credentials, deploy rights or sensitive data make it Tier 1); don't cap the finding.
- **A tier can make a deadline stricter, never looser.**
- **Some findings ignore the tier completely:** actively exploited bugs (KEV, CISA's Known Exploited Vulnerabilities catalog), leaked live secrets, and exposed or high-impact findings where nobody knows yet if the bug is reachable. They get their priority no matter what tier the asset is.

Tiers also decide which paved-road controls are required (<span data-cs="/2026/10/07/appsec-program-playbook-paved-road/">3.3</span> *(coming soon)*) and the order of fix campaigns (<span data-cs="/2026/10/07/appsec-program-playbook-remediate-at-scale/">3.5</span> *(coming soon)*). Store the tier in the catalog next to the owner, so everyone uses the same value.

### Step 5: SBOMs, the Dependency Layer Done Right

#### SBOMs in Two Minutes

An SBOM (software bill of materials) is a list of everything your software is built from, like the ingredients label on food. For each component it records:

- **what it is:** name, version, and a package URL ([purl](https://github.com/package-url/purl-spec)) such as `pkg:npm/axios@1.14.1`;
- **how you got it:** which component pulled it in (the dependency relationships);
- **proof and paperwork:** component hashes, license, producer (who made it).

**Does it include sub-dependencies?** A good one does. Your code uses a library, that library uses three more, and those use others. Those transitive (indirect) dependencies are where much of the vulnerable code hides, and CISA's 2026 guidance says an SBOM should cover them with "no minimum depth." But whether you get them depends on how the SBOM was made:

| How it was made | Sub-dependencies? | What it misses |
|---|---|---|
| From manifests only (`package.json`, `pom.xml`) | No, or guessed | Exact versions of everything indirect |
| From lockfiles (`package-lock.json`, `poetry.lock`) | Yes, with exact versions | OS packages, vendored code, anything added at build time |
| From the built container image (e.g. [Syft](https://github.com/anchore/syft) on the image) | Yes, for packages the scanner recognizes | Build-only dependencies, such as dev dependencies and build tools that never reach the image. Files with no package metadata, such as copied-in binaries and vendored code. And it's only as fresh as the last build |

**What an SBOM doesn't tell you:** whether any of those components is vulnerable. For that you match the SBOM against vulnerability databases (that's what Dependency-Track does), and whether a vulnerable component actually matters is a <span data-cs="/2026/10/07/appsec-program-playbook-prioritize-ruthlessly/">3.4</span> *(coming soon)* question. The SBOM's job is simpler: when an advisory lands, it answers "do we have this, and where?" in minutes.

**The two formats.** CycloneDX and SPDX (System Package Data Exchange) are both open standards, and CISA's 2026 guidance names both:

| | CycloneDX | SPDX |
|---|---|---|
| Steward | OWASP; standardized as [ECMA-424](https://ecma-international.org/publications-and-standards/standards/ecma-424/) | Linux Foundation; SPDX 2.2.1 is ISO/IEC 5962:2021 |
| Current version | [v1.7](https://cyclonedx.org/news/cyclonedx-v1.7-released/) (October 2025; [1.7.2](https://github.com/CycloneDX/specification/releases) is the current patch); ECMA-424 2nd edition (December 2025) | [3.0.1](https://spdx.github.io/spdx-spec/v3.0.1/) (December 2024), with [3.1 in release candidate](https://github.com/spdx/spdx-spec/releases); much tooling still emits 2.3 |
| Main focus | Security: vulnerabilities, VEX (Vulnerability Exploitability eXchange), services, ML-BOM, cryptography | Started with license compliance; 3.0 adds security, build, AI and dataset profiles |
| AI coverage | ML-BOM since [v1.5](https://cyclonedx.org/capabilities/mlbom/) (models, datasets, model cards) | [AI and Dataset profiles](https://spdx.github.io/spdx-spec/v3.0.1/model/AI/AI/) in 3.0 |

My rule: **store one format internally, accept both.** Most security tools (Dependency-Track included) are built around CycloneDX, so use it as your internal default. Still accept SPDX, because vendors and platforms send it. GitHub, for example, can export an SBOM for any repo, but only in SPDX. It's built from the repo's manifests and lockfiles, so it's one SBOM per repo and won't include OS packages in your images. Great for coverage, not as your only source.

One tip if you pull SBOMs from GitHub: use the newer [two-step export](https://docs.github.com/en/rest/dependency-graph/sboms) (start a report, then download it when it's ready). The older one-step export shuts down on November 13, 2026.

Don't let the format turn into a months-long debate. CISA's 2026 guidance says to "accept any widely used, interoperable, and machine-processable SBOM format." What matters more is what's inside the SBOM, which is next.

**What a good SBOM must contain.** In July 2026, CISA published the [2026 Minimum Elements for an SBOM](https://www.cisa.gov/resources-tools/resources/2026-minimum-elements-software-bill-materials-sbom), which replaces the older 2021 list. You don't need to memorize it; your SBOM tool fills in most of it. Four points matter for an inventory:

- **Include sub-dependencies, all the way down.** The test: if a component isn't in your SBOM, you should be able to say "we're not affected" with confidence. That only holds for what your SBOM method can see. An image SBOM can't clear a build-time compromise, such as a malicious install script in a dev dependency that ran during the build and never reached the image. For that, check the lockfile SBOM and the build logs.
- **Record how it was made.** Before build, at build, or after build. Two SBOMs of the same service can differ, so label which kind you have. (Syft doesn't fill this in yet; add it yourself before upload.)
- **"Unknown" must be written down,** never left as a silent gap.
- **Sign it and hash the components,** so people can trust it. 3.3 covers signing.

**It also helps your budget case** ([3.1](/2026/10/07/appsec-program-playbook-operating-model/)). The [EU Cyber Resilience Act](https://eur-lex.europa.eu/eli/reg/2024/2847/oj) requires SBOMs for products sold in the EU, with vulnerability-handling rules from December 2027. Pure SaaS is mostly out of scope, so check which parts of your company are in. And PCI DSS (Payment Card Industry Data Security Standard) requirement 6.3.2 already asks for an inventory of your custom software and its third-party components.

**Where to make SBOMs: use all three.**

| Where | Why | Use it for |
|---|---|---|
| **In the pipeline**, in the build job (an image scan there is still post-build) | Matches exactly what shipped, and can be signed | Every Tier 1 service and every new pipeline |
| **From repos**, by scanning them | Covers everything, even repos nobody builds anymore | Older repos the pipeline doesn't build yet |
| **From the running image** | Shows what's actually running in production | Checking production, especially old services |

Label each SBOM with how it was made, so you know which one you're looking at.

**Store them where you can search them.** SBOMs sitting in a bucket are an archive, not an inventory. Load them into something you can search by package and version, and join to owners and tiers. [Dependency-Track](https://docs.dependencytrack.org/usage/impact-analysis/) and [GUAC](https://github.com/guacsec/guac) (Graph for Understanding Artifact Composition) are open-source options. A plain database table works too; the [starter kit](https://github.com/hijacksecurity/appsec-playbook-templates/tree/main/inventory/are-we-affected) has one you can try.

**Write down "present, but not affected."** Sometimes a vulnerable component is there but can't be exploited, for example because your code never calls the vulnerable part. Record that as a VEX (Vulnerability Exploitability eXchange) statement with the reason, keep it next to the SBOM, and recheck it on every build. Otherwise the same false alarm comes back every time. 3.4 covers what evidence you need.

### Step 6: Beyond SBOMs: Actions, IaC, Images, Dev Tools and AI

Every other layer needs its own collector.

**CI/CD workflows and third-party actions.** Parse every workflow file and record each `uses:` reference twice: **the ref as written** (`@v4`, `@main`, `@<sha>`) and **the commit it resolved to.** The tj-actions and trivy-action attacks *repointed tags*, so in an incident the question isn't "do we reference `v45`?" It's "did anything resolve to the malicious commit during the window?" A current-state `resolved_sha` can't answer that, because repointed tags are usually restored after the incident (tj-actions' were), so today's resolution looks clean. Keep a **timestamped history** of every resolution instead: one row per workflow run, or at least per change. The fallback is each workflow run's "Set up job" log, which records the SHA every action resolved to. But run logs expire (90 days by default, configurable), so pull them early as part of the compromise handoff. GitHub's dependency graph does parse `uses:` references as dependencies, but "for GitHub Actions, alerts are only generated for actions that use semantic versioning, not SHA versioning" ([GitHub docs](https://docs.github.com/en/code-security/reference/supply-chain-security/dependency-graph-supported-package-ecosystems)). You should pin by SHA (GitHub's [secure-use reference](https://docs.github.com/en/actions/reference/security/secure-use) says a full-length commit SHA, and 3.3 makes it the default), so your own inventory has to map SHAs back to upstream advisories. Also record the triggers (flag `pull_request_target`), the `permissions:` block, the secrets referenced, and hosted vs. self-hosted runners. Those fields turn "are we affected?" into "what could it have stolen?"

**Include your own security tools.** In March 2026 the scanners were the way in. Every scanner action or image in your pipeline templates is a dependency of every repo that uses them.

**IaC modules.** Record module sources (registry name and version, or git URL and ref), providers and their versions, Helm charts and chart versions, and the accounts and clusters each stack targets. A git-sourced module pinned to a branch is just as mutable as an action pinned to a tag.

**Container images and base images.** For every image, record the digest, the base image name and digest, the Dockerfile path and repo, and the SBOM. Inheritance is the most useful link: "which images derive from this base?" is the query behind 3.5's base-image strategy.

**Developer tooling.** Honestly, this is the hardest layer. Some of it lives in repos: `.vscode/extensions.json` recommendations, `.vscode/tasks.json`, `.mcp.json` and `.vscode/mcp.json` MCP server configs, and `.claude/settings.json` and similar agent configs. Crawl for those files: they're an inventory source and a place worms have planted persistence. The rest lives on laptops, so partner with whoever runs endpoint management to collect installed IDE extensions (for VS Code, `code --list-extensions --show-versions`), AI CLIs and local MCP servers. Add a developer-tooling row, with endpoint management as R, to the 3.1 RACI (responsible, accountable, consulted, informed) instead of building a shadow endpoint agent.

**AI components (AI-BOM).** For every product feature and internal agent, record:

- the model and provider (hosted API or self-hosted weights, plus the weights' hash if self-hosted);
- the SDKs and gateways (LiteLLM-style proxies are dependencies with keys to everything);
- the datasets used for fine-tuning or retrieval;
- the agents, and the **MCP servers and tools** each agent can call, with their scopes and credentials.

The standards are catching up: CycloneDX ML-BOM, SPDX 3.0's AI and Dataset profiles, and the CISA and G7 [Software Bill of Materials for AI - Minimum Elements](https://www.cisa.gov/resources-tools/resources/software-bill-materials-ai-minimum-elements) (May 2026), written by the G7 Cybersecurity Working Group and explicitly not mandatory. It groups supplemental elements into seven clusters: Metadata, System Level Properties, Models, Datasets Properties, Infrastructure, Security Properties, and Key Performance Indicators. <span data-cs="/2026/10/07/appsec-program-playbook-agentic-era/">3.7</span> *(coming soon)* extends every phase to AI. For 3.2 the point is simpler: **if you can't list your agents and the tools they can reach, you can't answer "are we affected?" for the next postmark-mcp.**

**Dev platform integrations and third-party SaaS.** Pull installed apps, extensions, plugins and org webhooks from the platform admin APIs, with scopes and repo reach. Separately, list SaaS outside the dev platform that holds a token to code, data or production, and the OAuth grants people gave it. They hold credentials, so they belong in the same query.

### Step 7: The "Are We Affected?" Query, Your Acceptance Test

Everything above exists so this query returns in minutes. It's the exposure step of 3.5's advisory-to-incident-response loop, which only works if **exposure** is a query, not a project.

The pattern for every layer: **match the indicator → walk to the deployed asset → join owner and tier → sort by tier and exposure.**

```sql
-- Same tables as the starter kit's SQLite database (are-we-affected/schema.sql).
-- Q1: Are we affected by a compromised action? Match on the resolved commit during the
-- exposure window, not on the tag. action_usages keeps one timestamped row per run
-- (or per change), so a tag restored after the incident doesn't hide the hit.
SELECT a.repo, u.workflow_path, u.ref_as_written,
       MIN(u.observed_at) AS first_run, COUNT(*) AS runs,
       a.tier, a.owner_team, a.oncall
FROM   action_usages u
JOIN   assets a ON a.id = u.asset_id
WHERE  u.action = 'tj-actions/changed-files'
  AND  u.resolved_sha = '<malicious sha>'
  AND  u.observed_at BETWEEN '<window start>' AND '<window end>'
GROUP  BY a.repo, u.workflow_path, u.ref_as_written
ORDER  BY a.tier, a.repo;

-- Q2: Which services ship a malicious package version, and where do they run?
-- No deployment row means the image was built but isn't running.
SELECT a.name, a.tier, a.owner_team, a.oncall, c.version, c.generation_context,
       d.cluster, d.namespace, a.internet_facing, a.indirect
FROM   components c
JOIN   assets a         ON a.id = c.asset_id
LEFT JOIN deployments d ON d.image_digest = c.image_digest
WHERE  c.purl = 'pkg:npm/axios'
  AND  c.version IN ('1.14.1', '0.30.4')
ORDER  BY a.tier, a.internet_facing DESC, a.indirect DESC;
```

Run the same shape against IaC module refs, base-image digests, extension IDs and MCP server packages. Feed it threat intel, not just CVEs (Common Vulnerabilities and Exposures entries). OpenSSF's [Malicious Packages](https://github.com/ossf/malicious-packages) dataset publishes known-malicious versions as `MAL-` entries in the OSV (Open Source Vulnerabilities) format; match those against the inventory automatically.

Two caveats:

- **"In an SBOM" isn't "pulled during the window."** Knowing whether CI or a laptop resolved the bad version in a three-hour window takes build, proxy or registry logs. That's compromise assessment (in [NIST SP 800-61r3](https://csrc.nist.gov/pubs/sp/800/61/r3/final) terms, incident analysis under Respond, not detection). Per the [3.1](/2026/10/07/appsec-program-playbook-operating-model/) handoff, it goes to SOC/IR (security operations and incident response); 3.5 covers the trigger. The inventory narrows the search from thousands of repos to dozens, and knows where those logs live.
- **Unknown is an answer.** If a Tier 1 service has no SBOM, the query must return it as *unknown*, not drop it. A silent gap reads as "not affected." The starter kit has a query that lists running images with no SBOM; report its results next to Q2. That's the Explicitly Identifying Unknown Information principle, applied to your own estate.

**Practice it.** Once a quarter, pick a real past advisory and time it from advisory to owner fan-out. That drill is this phase's most honest metric.

### Step 8: Link Code to Cloud

A finding in code without deployment context is incomplete. Is it running at all, where, behind what, for how many tenants? That's the Exposed and Impact data 3.4 needs, and it comes from linking the chain:

<div class="mermaid">
graph LR
    R[Repo + commit<br/>owner, tier] -->|CI builds| B[Build run<br/>workflow, actions, secrets]
    B -->|produces| I[Image digest<br/>OCI source + revision annotations<br/>base image digest]
    B -->|generates| S[SBOM + provenance<br/>generation context recorded]
    I -->|deployed by digest| D[Deployment<br/>cluster, namespace, account]
    D -->|serves| E[Endpoint / domain<br/>exposure: internet_facing, indirect, auth]
    IAC[IaC module + ref] -->|provisions| D
    S -.->|queried by| Q["Are we affected?"]
    E -.->|feeds| P[3.4 Exposed + Impact]
</div>

The glue is small:

1. **Stamp images at build** with the standard [OCI annotations](https://github.com/opencontainers/image-spec/blob/main/annotations.md): `org.opencontainers.image.source`, `org.opencontainers.image.revision`, `org.opencontainers.image.base.name` and `org.opencontainers.image.base.digest`. These are manifest annotations; the same keys work as image-config labels for tools that only read labels. Now any running container points back to a repo and commit.
2. **Deploy by digest**, not by tag. Tags move; digests don't. Same lesson as action pinning.
3. **Read what's actually running** from the cluster API and cloud asset inventories, and join on digest.
4. **Tag cloud resources** with the service ID, so accounts, buckets and databases join back to the catalog.

The same linkage keeps dead-repo detection honest and traces a 3.5 base-image fix to every service it touches.

### Step 9: Score Repo Hygiene with OpenSSF Scorecard

Once you have every repo, score how well each one is protected. [OpenSSF Scorecard](https://github.com/ossf/scorecard/blob/main/docs/checks.md) runs automated checks against a repo. The ones that matter most internally map straight onto the incident record:

| Check | Scorecard risk | What it catches |
|---|---|---|
| **Dangerous-Workflow** | Critical | `pull_request_target` and script-injection patterns (Nx, TanStack, Ultralytics-style) |
| **Token-Permissions** | High | Workflows with broad write tokens |
| **Pinned-Dependencies** | Medium | Actions and dependencies pinned to mutable refs (tj-actions, trivy-action) |
| **Branch-Protection** | High | Unreviewed pushes to the default branch |
| **Code-Review** | High | Changes merged without review |
| **Dependency-Update-Tool** | High | No Dependabot or Renovate, so patching will hurt later |

Scorecard is built for open-source projects, so some checks (Contributors, Fuzzing) mean less inside a company. Run your policy subset across the inventory and store the results as posture fields per repo. The point is telling 3.3 which paved-road controls to push first, not building a leaderboard.

### Step 10: Keep It Fresh

You need two mechanisms:

- **Event-driven updates** for speed: SCM webhooks, a build hook that posts the SBOM and image digest, deploy events, and cloud config change streams (e.g. AWS Config).
- **Scheduled reconciliation crawls** for truth. Events get dropped, webhooks get misconfigured, and someone always deploys from a laptop. The crawl catches drift and resets `last_seen`.

Then set **freshness SLOs (service-level objectives) for the inventory itself**. For reference, CISA's [Binding Operational Directive (BOD) 23-01](https://www.cisa.gov/news-events/directives/bod-23-01-improving-asset-visibility-and-vulnerability-detection-federal-networks) tells federal agencies to "perform automated asset discovery every 7 days" and to enumerate vulnerabilities every 14 days. The [implementation guidance for BOD 26-04](https://www.cisa.gov/news-events/directives/bod-26-04-implementation-guidance-prioritizing-security-updates-based-risk) points back to it, expecting agencies to keep a list of agency-managed and public-exposed assets reportable to CISA within seven days. Code and pipelines are event-driven, so you can do much better (SLO table below).

## Demo: The Pinecart Lab

**Want to start today?** The [inventory starter kit](https://github.com/hijacksecurity/appsec-playbook-templates/tree/main/inventory) has everything in this demo, tested and ready to copy:

| What | Use it to |
|---|---|
| [`catalog-info.template.yaml`](https://github.com/hijacksecurity/appsec-playbook-templates/blob/main/inventory/catalog-info.template.yaml) | Give every service an owner, a tier and its exposure facts |
| [`tiering-rubric.md`](https://github.com/hijacksecurity/appsec-playbook-templates/blob/main/inventory/tiering-rubric.md) | Tier services the same way across teams (one page) |
| [`asset-record.template.yaml`](https://github.com/hijacksecurity/appsec-playbook-templates/blob/main/inventory/asset-record.template.yaml) | Know which fields to track for every asset |
| [`scripts/list-actions.sh`](https://github.com/hijacksecurity/appsec-playbook-templates/blob/main/inventory/scripts/list-actions.sh) | List every GitHub Action your repos use, and flag the ones not pinned to a SHA |
| [`are-we-affected/`](https://github.com/hijacksecurity/appsec-playbook-templates/tree/main/inventory/are-we-affected) | Try the "are we affected?" queries on a small SQLite database with sample data |

The tools below are examples by category, each with at least two open-source or standards-based options. Not endorsements.

| Category | Options | What it does in the lab |
|---|---|---|
| SCM inventory | GitHub / GitLab / Azure DevOps APIs; GitHub dependency graph | Repo crawl, workflow and `uses:` extraction, SPDX export |
| SBOM generation | Syft, cdxgen, Trivy | CycloneDX at build and at rest |
| SBOM store and query | OWASP Dependency-Track, OpenSSF GUAC | Portfolio search by purl, impact analysis |
| Service catalog and ownership | Backstage; commercial internal developer portals; a catalog table fed from CODEOWNERS | Owner, tier, lifecycle, system |
| Repo posture | OpenSSF Scorecard; zizmor or actionlint for workflow checks | Hygiene checks per repo |
| Cloud and cluster inventory | AWS Config, Azure Resource Graph, Steampipe, CloudQuery | What's actually running, by digest |
| External discovery | ProjectDiscovery subfinder/httpx, OWASP Amass | Domains, hosts, exposed services |

**SBOM at build, stamped and uploaded.** Every Pinecart service inherits this paved-road step from the template in 3.3 (actions pinned by SHA):

```yaml
# .github/workflows/build.yml (excerpt)
permissions:
  contents: read
  packages: write
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - name: Log in to the registry
        uses: docker/login-action@dbcb813823bdd20940b903addbd779551569679f # v4.6.0
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Set up Buildx (annotations need the docker-container driver)
        uses: docker/setup-buildx-action@f87e5991a6d7451dcb8d9637bfbc97413f497069 # v4.4.1
      - name: Build and push, stamped with OCI annotations
        id: push
        env:
          BASE_IMAGE: ${{ vars.BASE_IMAGE }}   # pinned by digest in the template, e.g. <registry>/node@sha256:...
        run: |
          BASE_NAME="${BASE_IMAGE%@*}"         # <registry>/node
          BASE_DIGEST="${BASE_IMAGE#*@}"       # sha256:...
          # "index," annotations need an image index. buildx creates one here only because it
          # adds a provenance attestation by default; with --provenance=false, use "manifest:" only.
          docker buildx build --push \
            --annotation "index,manifest:org.opencontainers.image.source=${GITHUB_SERVER_URL}/${GITHUB_REPOSITORY}" \
            --annotation "index,manifest:org.opencontainers.image.revision=${GITHUB_SHA}" \
            --annotation "index,manifest:org.opencontainers.image.base.name=${BASE_NAME}" \
            --annotation "index,manifest:org.opencontainers.image.base.digest=${BASE_DIGEST}" \
            --build-arg BASE_IMAGE="${BASE_IMAGE}" \
            --metadata-file meta.json \
            -t "ghcr.io/pinecart/checkout-api:${GITHUB_SHA}" .
          echo "digest=$(jq -r '."containerimage.digest"' meta.json)" >> "$GITHUB_OUTPUT"
      - name: SBOM from the pushed image (generation context, after build / post-build)
        uses: anchore/sbom-action@3ad7283483fc7af8ff2b4ea19663c2d5ca935e26 # v0.24.2
        with:
          image: ghcr.io/pinecart/checkout-api@${{ steps.push.outputs.digest }}
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
          format: cyclonedx-json
          output-file: sbom.cdx.json
      - name: Upload to the SBOM store
        env:
          DT_URL: ${{ vars.DT_URL }}
          DT_API_KEY: ${{ secrets.DT_API_KEY }}
        run: |
          # Syft doesn't set the CycloneDX lifecycle, so record the generation context here
          jq '.metadata.lifecycles = [{"phase": "post-build"}]' sbom.cdx.json > sbom.tmp && mv sbom.tmp sbom.cdx.json
          curl -sf -X POST "${DT_URL}/api/v1/bom" \
            -H "X-Api-Key: ${DT_API_KEY}" \
            -F "autoCreate=true" \
            -F "projectName=checkout-api" \
            -F "projectVersion=${GITHUB_SHA}" \
            -F "bom=@sbom.cdx.json"
```

For services that aren't on the paved road yet, scan at rest from the deployed digest:

```bash
syft scan registry:ghcr.io/pinecart/legacy-catalog@sha256:<digest> \
  -o cyclonedx-json=legacy-catalog.cdx.json
grype sbom:./legacy-catalog.cdx.json     # vulnerabilities from the SBOM, no re-scan
```

<a id="catalog-example"></a>
**Ownership in the catalog.** Here's the `catalog-info.yaml` from Step 3 for one Pinecart service. It says what the service is, who owns it and what it belongs to. We add a few annotations of our own for the security data from this post: tier, data class and exposure. AppSec opens it as a pre-filled PR, and the owning team approves or corrects it:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: checkout-api
  annotations:
    github.com/project-slug: pinecart/checkout-api
    pinecart.io/tier: "1"
    pinecart.io/data-class: payment
    pinecart.io/internet-facing: "true"      # same exposure fields as 3.4's scorecard
    pinecart.io/indirect: "true"
    pinecart.io/auth: any-customer
spec:
  type: service
  lifecycle: production
  owner: group:checkout-team
  system: storefront
```

Reading it: `owner` is the team that gets findings for this service, `system` groups related services, and the `pinecart.io/...` annotations carry the tier and exposure that 3.4 uses to prioritize. No Backstage? The same fields work as a plain YAML file in each repo, read by your inventory script.

**"Which services ship X?"** In Dependency-Track, a component search by purl across the portfolio (or the [impact analysis](https://docs.dependencytrack.org/usage/impact-analysis/) view) answers the package half. The SQL pattern in Step 7 joins it to owners, tiers and running deployments.

```bash
curl -s -H "X-Api-Key: ${DT_API_KEY}" \
  "${DT_URL}/api/v1/component/identity?purl=pkg:npm/axios@1.14.1&excludeInactiveProjects=true" \
  | jq -r '.[] | [.project.name, .project.version, .purl] | @tsv'
```

**Workflow and action inventory, refs as written (first pass)**, from a mirror of the org's repos. This captures only the refs as written, which is the tag-only view the anti-patterns below warn about, and it skips composite `action.yml` files. Use it to get started, then record resolved SHAs as in Step 6:

```bash
# Every action reference across the org (refs as written), most used first
grep -rhoE '^[[:space:]]*-?[[:space:]]*uses:[[:space:]]*[^ #]+' --include='*.yml' --include='*.yaml' \
  */.github/workflows | sed -E 's/.*uses:[[:space:]]*//' | tr -d "\"'" | sort | uniq -c | sort -rn | head -40
```

**Repo posture across the inventory.** Run Scorecard with a policy subset:

```bash
export GITHUB_AUTH_TOKEN=...   # a GitHub App or fine-grained token; Branch-Protection needs admin for full results
scorecard --repo=github.com/pinecart/checkout-api \
  --checks=Dangerous-Workflow,Token-Permissions,Pinned-Dependencies,Branch-Protection,Code-Review \
  --format=json > scorecard/checkout-api.json
```

Teams can also run the [Scorecard action](https://github.com/ossf/scorecard-action) (`ossf/scorecard-action@2d1146689b8cda280b9bc96326124645441f03bc # v2.4.4`), but for inventory the CLI loop gives one consistent run across every repo.

## What Good Looks Like

| | Crawl | Walk | Run |
|---|---|---|---|
| **Coverage** | Every repo in every SCM org, with language and ecosystem | + CI/CD workflows and actions, images, IaC modules, deployed services | + external surface, SaaS/SCM apps, developer tooling, AI-BOM |
| **Ownership** | CODEOWNERS default rule + SCM teams; an orphan list exists | Catalog for Tier 1 and 2; ranked resolution with confidence; orphan process with deadlines | Owners checked against the IdP on every sync; archive campaigns are routine; orphans near zero |
| **Tiering** | Tier 1 list agreed by hand | Rubric applied to all services and stored in the catalog | Tier drives controls (3.3) and SLAs (3.4) automatically |
| **SBOMs** | Scan at rest from lockfiles | Generated in the pipeline for Tier 1, stored and queryable; 2026 minimum elements met | Pipeline SBOMs for all paved-road services, running-image checks, VEX stored alongside |
| **Linkage** | Repo ↔ image by naming convention | OCI annotations, deploy by digest, joined to cluster and cloud inventory | Full repo → build → image → deployment → endpoint graph |
| **"Are we affected?"** | Hours to days, by hand | Under an hour for packages and actions | Minutes across every layer, practiced quarterly |
| **Freshness** | Weekly crawl | Webhooks + nightly reconciliation; SLOs defined | SLOs measured and alerted on, like a production service |

## Anti-Patterns

- **The spreadsheet inventory.** Accurate for a week, trusted for a year. If it isn't generated, it's fiction.
- **"Security owns the orphans."** Then security owns the fixes, and the program becomes the bottleneck it was meant to remove.
- **SBOMs as compliance paperwork.** Generated, archived, never queried. The test: can you answer "which services ship X?" from them today?
- **Tag-only action inventory.** Recording `@v4` without the resolved SHA misses exactly the attack that repoints tags.
- **Silent unknowns.** A service with no SBOM drops out of the query results and reads as "not affected."
- **Dead by commit date.** Archiving or ignoring a repo because it's quiet, while its image still serves traffic.

## Metrics for This Phase

These feed the coverage metrics in 3.6. Targets are Pinecart's, illustrative, not benchmarks.

| Metric | Definition | Illustrative target |
|---|---|---|
| Repo coverage | Repos in inventory ÷ repos in all SCM orgs (reconciled against the org APIs) | 100% |
| Owner coverage by confidence | % of active assets with a high / medium / low / no-confidence owner | High ≥ 90% for Tier 1 |
| Orphan count and age | Active assets with no confident owner; median days in the orphan queue | Trending down; ≤ 30 days |
| SBOM coverage | % of Tier 1 / all deployed services with a pipeline-generated SBOM for the running image digest | Tier 1: 100% |
| Code-to-cloud traceability | % of running image digests traceable to a repo and commit | Tier 1: 100% |
| Time to answer "are we affected?" | Minutes from advisory to owner-routed list, measured in quarterly drills | Minutes, not days |
| Freshness SLO attainment | % of assets whose `last_seen` is within the SLO for their layer | ≥ 99% |
| Posture on key checks | % of Tier 1 repos passing Dangerous-Workflow, Token-Permissions and Pinned-Dependencies | Rising each quarter |

## Takeaways and Templates

All of these are also in the [inventory starter kit](https://github.com/hijacksecurity/appsec-playbook-templates/tree/main/inventory), ready to copy.

**1. Minimum inventory record.** Every asset, every layer:

```yaml
asset:
  id: svc:checkout-api                 # stable, never reused
  layer: deployed-service              # repo | workflow | action-ref | iac-module | image | deployed-service | endpoint | platform-integration | saas-app | dev-tool | ai-component
  name: checkout-api
  status: active                       # active | dormant | archived | decommissioned
  owner:
    team: checkout-team                # a team, validated against the IdP
    source: catalog                    # catalog | codeowners | deploy-metadata | scm-team | commit-heuristic | default
    confidence: high                   # high | medium | low | none
    oncall: "#checkout-oncall"
  tier: 1                              # 1 | 2 | 3; maps to 3.4 impact.tier: top | middle | low
  tier_inputs:
    business: revenue-critical
    data_class: payment
    exposure: {internet_facing: true, indirect: true, auth: any-customer}   # 3.4's exposed: fields; auth: anonymous | any-customer | staff | service
  links:
    repo: github.com/pinecart/checkout-api
    images: ["ghcr.io/pinecart/checkout-api@sha256:..."]
    deployments: ["prod-eu/checkout", "prod-us/checkout"]
    sbom: {format: CycloneDX, spec_version: "1.7", generation_context: post-build, timestamp: "2026-10-14T09:12:00Z"}
  posture: {scorecard: {Dangerous-Workflow: 10, Token-Permissions: 10, Pinned-Dependencies: 9}}
  provenance: {first_seen: "2025-03-02", last_seen: "2026-10-15T08:00:00Z", source: webhook+nightly-crawl}
  unknowns: []                         # explicit: e.g. ["sbom: unknown", "exposure: unknown"]
```

**2. Tiering rubric.** The worst axis wins:

| Axis | Tier 1 | Tier 2 | Tier 3 |
|---|---|---|---|
| Business | Revenue path / critical journey / holds publish or deploy rights | Customer-facing, not critical path | Internal, not critical |
| Data | Sensitive data: payment data, credentials and secrets, bulk personal data | Limited personal data | No personal or sensitive data |
| Exposure | (exposure alone never sets Tier 1) | `internet_facing` or `indirect` | Internal and not indirectly exposed |

**3. Ownership resolution order:** catalog → CODEOWNERS default → deployment metadata → SCM team permission → commit heuristic → default owner plus orphan queue. Teams only. Validate against the IdP. Record source and confidence.

**4. Freshness SLOs** (illustrative starting points):

| Layer | Update mechanism | SLO |
|---|---|---|
| New or archived repo | SCM webhook + nightly reconciliation | ≤ 1 hour (event), ≤ 24 hours (worst case) |
| Workflows and action refs | Push webhook | ≤ 1 hour after merge |
| Build SBOM and image digest | Build step | At build, every build |
| Running deployments | Deploy events + cluster/cloud sync | ≤ 1 hour (event), ≤ 24 hours (reconcile) |
| External attack surface | Scheduled discovery | ≤ 7 days (matches BOD 23-01's discovery cadence) |
| Developer tooling | Repo crawl + endpoint inventory | ≤ 7 days |
| AI components | Repo crawl + gateway logs + catalog | ≤ 7 days |
| Owner validation | IdP sync | ≤ 24 hours; full re-confirmation ≤ 90 days |

**5. The "are we affected?" checklist**, for 3.5's intake step:

- [ ] Match the indicator (purl@version, action@SHA, module@ref, image digest, extension ID, MCP package) across **every** layer.
- [ ] Walk each hit to its deployed assets. Mark *unknown* explicitly where linkage is missing.
- [ ] Join owner, on-call channel, tier and exposure. Sort Tier 1 and exposed first.
- [ ] For CI hits, match the resolved SHA within the exposure window, not the tag. Pull the triggers, runner type, token permissions and secrets referenced, and save the run logs before they expire.
- [ ] Record "not affected" as VEX with evidence, and store its expiry in the inventory beside it.
- [ ] Hand the list to the 3.5 campaign, with the time you ran it.

## Next

The inventory tells you what exists, who owns it and how much it matters. By itself it doesn't secure anything. Next: put the right controls on those assets in a way teams actually adopt, so most of this data comes free as a side effect of shipping.

**Next:** <span data-cs="/2026/10/07/appsec-program-playbook-paved-road/">3.3 The AppSec Program Playbook: Build the Paved Road - Secure by Default, Self-Serve</span> *(coming soon)*
