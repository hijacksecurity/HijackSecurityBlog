---
layout: post
title: "2.0 Building Intercept: One Founder, Three AI Teams"
date: 2026-05-02 10:00:00 -0000
categories: ai founder devops architecture
tags: ["Building Intercept", "AI", "Solo Founder", "Architecture"]
series: "Building Intercept"
series_part: "2.0"
---

I'm building an application security platform. Solo.

The platform is **Intercept**, under the Hijack Security name. Real product, real AWS bill, days away from early access for a small group of trusted users. The thing that makes any of this work — what lets one person actually build something at this scope — is that I don't treat AI as an assistant. I treat it as three distinct teams, with their own repos, their own rules, and their own boundaries. They argue with me about things I get wrong. They wait for my review before doing anything destructive. And they almost never bleed context into each other, because the context lives inside each team.

This series is about how that's organized, why it's organized that way, and what I've learned in the process.

- **[2.1 Building Intercept: The Infrastructure Team — Scripts First, Always](/2026/05/02/building-intercept-infrastructure-team/)**
- **[2.2 Building Intercept: The Engineering Team — Specialists, All the Way Down](/2026/05/02/building-intercept-engineering-team/)**
- **[2.3 Building Intercept: The Operations Team — Always On, On a Laptop](/2026/05/02/building-intercept-operations-team/)**

This post is the frame: what each team owns, how they hand off to each other, and the shared rules that make the whole thing actually work.

<div style="text-align: center; margin: 30px 0;">
  <img src="/assets/images/intro.png" alt="A solo founder standing in a Renaissance courtyard at dusk, three illuminated arches around them framing scenes from the Infrastructure, Engineering, and Operations teams, with a tall vertical tower visible through the center arch" style="max-width: 100%; height: auto; border-radius: 8px; box-shadow: 0 2px 12px rgba(0,0,0,0.15);">
  <p style="font-size: 0.9em; color: #666; margin-top: 8px;"><em>Three teams. One founder. One product.</em></p>
</div>

## The Three Teams

Three repositories on disk. Three "departments" of AI work. One founder reading every PR.

<div class="mermaid">
graph TB
    F[Founder<br/>One human, every PR]

    subgraph Infrastructure
        I[Infrastructure Team]
        I --> AWS[AWS / EKS / ECR]
        I --> GH[GitHub Actions / OIDC]
        I --> SEC[DevSecOps]
    end

    subgraph Engineering
        E[Engineering Team]
        E --> SVC[Microservices]
        E --> WEB[Web App]
        E --> APIs[APIs and Workers]
    end

    subgraph Operations
        O[Operations Team]
        O --> MON[Monitoring]
        O --> PATCH[Patching]
        O --> QA[Daily QA]
    end

    F --> I
    F --> E
    F --> O

    E -.->|requests resources| I
    O -.->|watches| E
    O -.->|reads| AWS
</div>

| Team | What They Own |
|------|---------------|
| **Infrastructure** | AWS, EKS, GitHub Actions, secrets, DevSecOps |
| **Engineering** | The product — microservices, APIs, the web app |
| **Operations** | Monitoring, patching, on-call, daily app checks |

When Engineering needs a new database for a new service, it doesn't build the database — it asks Infrastructure to. When Operations notices a service is degraded, it pings me on Slack and (within its allowed scope) starts triaging. The teams don't share context. They share me.

## Why Three Teams Instead of One Giant Chat

The temptation when you start building with AI is to put everything in one session. One agent that knows your whole company. One context window holding your code, your infra, your deploys, your users. It feels efficient.

It isn't. Three reasons:

**1. Attention is finite, even when context isn't.** A model juggling Kubernetes configs, FastAPI handlers, and yesterday's CloudWatch alarm is worse at all three than a model thinking about one. Specialization isn't a limitation — it's a feature.

**2. Permissions should match the job.** The agent that writes application code doesn't need to be able to delete an RDS instance. The agent that monitors production doesn't need to be able to push to `main`. Splitting into teams splits the blast radius.

**3. It mirrors how a real company works.** I didn't invent the Infra / Eng / Ops split — every engineering org of more than a handful of people lands on something like it. There's a reason. Mirroring it gives me clean handoffs, clean accountability, and clean repos. When Engineering needs an OIDC binding, it opens a request. Infrastructure builds it. Engineering wires it up. Same workflow as a real team — just compressed into one person reviewing the PRs.

## The Constitution

Every team operates under the same shared rules. These aren't suggestions — they're enforced by repo conventions, hooks, or the simple fact that I won't merge what breaks them.

### 1. Writes are gated by a human

No agent on any team executes a destructive or state-changing write directly. The *mechanism* varies — Infrastructure uses script review (AI drafts, I read, I run), Engineering uses branch protection and PR review, Operations uses Claude Code permission hooks that block tool calls outside an agent's allowed scope. The mechanism is per team. The rule is invariant.

This is the load-bearing principle of the whole setup. Everything else is built on top of it.

### 2. No secrets in the repo, ever

Pre-commit hooks (Gitleaks) block any commit with credential-shaped strings. Real secrets live in AWS Secrets Manager and reach pods at runtime via External Secrets Operator. Long-lived AWS access keys do not exist anywhere in the system — GitHub Actions authenticates via OIDC; pods authenticate via EKS Pod Identity.

### 3. Pin everything

Kubernetes versions, addon versions, container image tags, dependency versions — all pinned, all reviewable, none on `latest`. When AI is writing your scripts, "use the newest version" is a foot-gun. Pinning makes "what changed" answerable.

### 4. The human is the merge button

Every team can branch, commit, and open PRs. None of them can merge to `main`. I am the merge button. AI proposes; the founder disposes.

## Tools, By Team

Each team uses a different mix of tooling, chosen to fit the work:

| Team | Primary Tools | Why |
|------|---------------|-----|
| **Infrastructure** | Claude Code (writes scripts) + Kiro CLI (read-only AWS analyst) | Two-tool split: write vs. read, cleanly separated. The two tools never talk to each other — I'm the bridge. Kiro CLI is AWS's official agentic CLI (the evolution of Amazon Q Developer CLI), built on Claude frontier models, and unusually current on AWS itself — what's running, what's it costing, what versions are available today — without ever touching write APIs in this setup. |
| **Engineering** | Claude Code with per-service subagents, plus a Project Manager in a parallel window | The product is a set of microservices; each one has its own scoped agent. The generalist isn't allowed to touch service code — it must delegate. A separate Claude Code window runs as the PM for planning and issue creation, so I can plan ahead while the engineering session keeps building. |
| **Operations** | Single-shot `claude -p` CLI invocations under launchd, with per-role permission hooks | Built on the Claude Code subscription (CLI), not the API — the cost story is what makes 24/7 viable. Every scheduled job spawns a fresh agent, runs one task, exits. Hooks enforce what each role can actually do at the tool level. |

There's overlap — Claude Code is the workhorse across all three — but each team adds its own constraints on top.

## What Each Part Covers

**[2.1 Building Intercept: The Infrastructure Team — Scripts First, Always](/2026/05/02/building-intercept-infrastructure-team/).** How AWS gets built and torn down. Why I picked `eksctl` over Terraform. The script-pair pattern (every create has a paired delete). DevSecOps as built-in, not bolt-on. The two-tool split between Claude Code and Kiro CLI — and why the human is deliberately the bridge between them, not an integration.

**[2.2 Building Intercept: The Engineering Team — Specialists, All the Way Down](/2026/05/02/building-intercept-engineering-team/).** The product itself: how 22 specialized agents — one per microservice, plus cross-cutting specialists for data, QA, pen-testing, troubleshooting, and more — got built one at a time as the codebase grew. The folder structure that makes per-agent ownership unambiguous. Strict delegation, hooks as a constitution, the Project Manager that lives in a parallel Claude Code window so I can plan ahead while the engineering session keeps building. The MCP server we ship for our users' agents, and the AI features built into the product itself.

**[2.3 Building Intercept: The Operations Team — Always On, On a Laptop](/2026/05/02/building-intercept-operations-team/).** The always-on team built on the Claude Code CLI subscription (not the API) — the laptop is the server, and the design leans into that. A single MacBook running 24/7 with four launchd-scheduled agents, role-based permission hooks, daily Dependabot patching with one PR per repo and one Slack-thread approval. Bidirectional Slack — I can ask the team things, not just read their reports. Auto-closing GitHub tickets that work like a real on-call queue. And the 53-hour battery-hibernation outage that taught me what 24/7 actually requires.

## Who This Is For

There's a bigger question lurking under all of this: **what does it look like for AI agents to run a company?** This series is one possible answer — at least for the *IT* side of a software startup, which for a product company is most of what running the company actually is. The three teams together cover infrastructure, engineering, and operations: every part of building, shipping, and maintaining the product itself. The business side — sales, marketing, fundraising, partnerships, the work that lives on judgment and trust — stays in my hands. But the engineering organization behind the product? This is what one AI-native version of it looks like.

If you're a founder wondering whether you can build something real with a tiny team and a lot of AI — yes, but the structure matters more than the model. If you're an engineer trying to picture what an "AI-native company" actually looks like in practice (not in a keynote) — this is what one looks like from the inside. If you're a leader thinking about org design — there may be ideas here that scale up to teams of humans, too.

This setup works. The teams ship. The structure is what makes the speed possible.

---

*The series is now complete. Each part stands alone, but they're better read in order.*

**Next:** [2.1 Building Intercept: The Infrastructure Team — Scripts First, Always](/2026/05/02/building-intercept-infrastructure-team/)
