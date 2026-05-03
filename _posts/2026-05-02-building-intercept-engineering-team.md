---
layout: post
title: "2.2 Building Intercept: The Engineering Team — Specialists, All the Way Down"
date: 2026-05-02 12:00:00 -0000
categories: ai engineering microservices architecture
tags: ["Building Intercept", "Engineering", "AI", "Architecture", "Microservices"]
series: "Building Intercept"
series_part: "2.2"
---

A cross-service change comes in. The **Project Manager**, running in a separate Claude Code window I keep open beside the engineering one, had filed it the day before — a clean GitHub issue, the right service labels applied, a place on the project board. A new tenant-level integration that needs to flow through three services: auth, the API, the frontend.

In the early days, I would have opened one chat with Claude, described what I wanted, and let it loose. By the end of the day there'd be commits in five places, two of them I didn't ask for, and a new test I hadn't authorized that almost-but-not-quite worked.

That's not how it goes anymore.

Today, the issue is picked up with `/project start`, and the **Technical Architect** writes a one-page plan: which services are touched, which schemas change, what the rollout sequence looks like. The plan goes to me. I read it, push back on one thing, approve the rest. Then the work fans out: the **Identity Engineer** picks up the auth piece, the **API Engineer** picks up the backend, the **Frontend Engineer** picks up the UI. The **Data Architect** owns the migration nobody else is allowed to touch. The **QA Lead** runs targeted tests after each service deploys, then runs full E2E before anything goes near production. The **Application Pen-Tester** runs a focused security pass on the new endpoint before it ships.

Seven agents touched that change. None of them stepped on each other. None of them did work they weren't supposed to do. None of them read code that wasn't theirs. And while they were working, the PM in the other window was already lining up the next three issues behind it.

<div style="text-align: center; margin: 30px 0;">
  <img src="/assets/images/engineering.png" alt="The Engineering Team — many specialist workshops stacked floor by floor in a vertical tower, with a glowing cathedral rising in the central column and the founder overseeing from below" style="max-width: 100%; height: auto; border-radius: 8px; box-shadow: 0 2px 12px rgba(0,0,0,0.15);">
  <p style="font-size: 0.9em; color: #666; margin-top: 8px;"><em>The Engineering Team — many specialists, strict separation, one product rising in the middle.</em></p>
</div>

This is the Engineering team. It is the absolute core of how Intercept gets built, and it is the part of this series I'm most proud of — because almost none of it was designed up front. Every agent on the team exists because at some point I needed it badly enough to make one.

## It Started With One Agent

In the beginning, the Engineering team was a single Claude session. I'd open a chat, describe what I wanted, and it would do everything — read code, write code, run tests, commit. For the first few weeks of building Intercept, this was glorious. The application was small enough to fit in one mind.

Then the codebase grew. The first service became three. Three became seven. And the same single agent that knew everything started to know everything *badly*. Context bled across services. Function signatures got hallucinated because the agent half-remembered the version it had seen ten messages ago. A change to one service quietly touched files in two others. Tests passed locally and broke in CI for reasons nobody in the chat could remember.

The single-agent setup didn't fail loudly. It failed slowly, in ways that took longer to debug than to write. That's the worst kind of failure — the kind you only notice in retrospect, after a week of small inefficiencies.

The first split was the obvious one: separate **planning** from **execution**.

## The First Split: Architect Before Builder

The Technical Architect was the first specialist on the team. The deal: any new feature, any cross-service work, any "this could be done two ways" question — start with the architect. The architect doesn't write code. It writes a plan. The plan names which services are touched, which agents will do the work, what the data flow is, what the rollout looks like. Then *I* read it. Then the work begins.

The first time it caught something useful was in a feature request that — in the planning doc — touched four services and required a non-trivial schema change. Reading the plan, I noticed the schema change broke backward compatibility for rolling deploys. The fix took five minutes in the doc. If I'd skipped the architect and let an executor agent run, the same fix would have taken half a day across three already-merged PRs.

That moment is the moment the team started to make sense.

## The Team Today

The Engineering team is now **22 specialized agents**, organized into three groups.

<div class="mermaid">
graph TB
    F[Founder<br/>Reviews every PR]
    TA[Technical Architect<br/>Plans every cross-service change]

    subgraph Service Engineers
        SE1[API Engineer]
        SE2[Identity Engineer]
        SE3[Worker Engineer]
        SE4[Frontend Engineer]
        SE5[and 7 more...]
    end

    subgraph Cross-Cutting Specialists
        DA[Data Architect]
        DO[DevOps Specialist]
        QA[QA Lead]
        TS[Specialized Testers]
        SEC[Security Specialists]
        UTIL[Utility Specialists]
    end

    F --> TA
    TA -.->|delegates to| SE1
    TA -.->|delegates to| SE2
    TA -.->|delegates to| SE3
    TA -.->|delegates to| SE4
    TA -.->|delegates to| SE5

    SE1 -.->|requests schema| DA
    SE2 -.->|requests schema| DA

    QA -.->|tests| SE1
    QA -.->|tests| SE2
    SEC -.->|pen-tests| SE1
</div>

### The Architect

| Role | Owns |
|------|------|
| **Technical Architect** | Plans every cross-service feature, every architectural decision, every "this could be done two ways" question. Produces blueprints; doesn't implement. |

### The Service Engineers (11 of them — one per microservice)

Intercept is a microservices platform. Every service has exactly one engineer responsible for it, and only that engineer is allowed to touch its code.

| Role | Owns |
|------|------|
| **API Engineer** | The main HTTP API. Endpoints, request flows, schema validation, auth integration. |
| **Identity Engineer** | Authentication, JWTs, OAuth flows, tenant management, sessions, credential storage. |
| **Worker Engineer** | Background job processing. Scanning modules, queue integration, result delivery. |
| **Threat-Intel Engineer** | Vulnerability intelligence. Feed integration, exposure analysis, tenant-scoped processing. |
| **Frontend Engineer** | The web UI. Components, pages, data fetching, routing. |
| **Admin Portal Engineer** | The internal operator portal. IP-restricted, used only by us. |
| **Alerting Engineer** | Alert delivery. Stream consumption, deduplication, email pipelines. |
| **Gateway Engineer** | The reverse proxy. Routing, rate limiting, TLS, health checks. |
| **Endpoint Agent Engineer** | The small agent users install on their machines for posture data collection. |
| **Egress Proxy Engineer** | The outbound proxy that restricts what worker traffic can reach. |
| **MCP Server Engineer** | Our public MCP server (more on this below). |

### The Cross-Cutting Specialists (10 of them)

These don't own a service. They own a *concern* that crosses every service.

| Role | Owns |
|------|------|
| **Data Architect** | Every schema. Every migration. Every shared data contract. The only agent allowed to write Alembic migrations or change `services/shared/`. |
| **DevOps Specialist** | CI/CD, Kubernetes manifests, Dockerfiles, GitHub Actions. The only agent allowed to touch deploy infrastructure. |
| **QA Lead** | Generalist quality. Runs tiered testing — unit, integration, E2E, security regression — appropriate to the size of the change. |
| **Identity QA Specialist** | Deep auth-and-tenant-isolation testing. Cross-tenant access attempts, JWT refresh, OAuth flows, role-based authorization boundaries. |
| **Threat-Intel QA Specialist** | Deep vulnerability-pipeline testing. Tenant overlay correctness, feed integrity, NULL tenant detection, service token boundaries. |
| **Application Pen-Tester** | OWASP- and PTES-aligned penetration testing. Reconnaissance, enumeration, exploitation attempts, structured findings. Runs against the test environment, not prod. |
| **Dependency Auditor** | Supply chain security. Scans Python and Node dependency trees for CVEs, license issues, outdated packages. |
| **Migration Validator** | Database migration safety. Tests every Alembic migration in both SQLite (dev-local) and PostgreSQL (dev-docker) to catch dialect divergence before it bites in prod. |
| **Code Simplifier** | Maintainability guardian. Engages after a feature ships. Removes dead code, reduces complexity, eliminates over-engineering, preserves all functionality. |
| **Troubleshooter** | Root-cause analysis only. Diagnoses runtime errors, deployment failures, environment divergence — and never does the fix. Hands the diagnosis off to the owning specialist. |

Twenty-two agents. None of them was designed in advance. Every single one came from a moment where I thought "I keep doing this same thing manually — there should be a specialist for it."

## The Folder Structure Is The Org Chart

Before any of the strict delegation rules can work, something more boring has to be true: **the codebase has to be organized so that "this code belongs to this agent" is unambiguous.**

The microservices architecture turned out to be the load-bearing precondition for the whole team. One folder per service. Each folder named clearly enough that a glance at the path tells you what's inside. Every service is self-contained — its own code, its own tests, its own VERSION file, its own CLAUDE.md describing service-specific conventions. The boundary between services is a directory boundary, not a vibes-based one.

Once the folders are clean, the agent assignment is trivial: **one folder, one agent.** Each service engineer owns exactly one service directory and is allowed to touch nothing else. There's no ambiguity about which agent should pick up a file change, and no ambiguity about whose code an agent is allowed to touch. The rules from the next section — strict delegation, generalist-forbidden-from-service-code, parallel work via labels — only work because the folder structure makes the rules legible to the harness.

This is also where the failure mode of "AI slop" gets eliminated. The thing that turns AI-written code into a maintenance nightmare is when an agent is allowed to drop files anywhere, write helpers in random places, leave scratch scripts at the repo root because "that's where it ran." The discipline that prevents it is rigid:

- **One folder per service. Named clearly. No exceptions.**
- **Documentation only goes in `docs/`.** Not at the repo root. Not in random subdirectories. Every documentation file has a place, and that place is `docs/`.
- **No random files at the repo root.** A `pre-commit-quality` hook blocks commits that would leave temp files, scratch scripts, or unclassified artifacts at the root. If a file exists there, it's there because it has a reason — `Dockerfile`, `pyproject.toml`, the things a Python project must have at the top level. Everything else is wrong.

These sound trivial. They aren't. The rule "documentation belongs in `docs/`, period" is a one-line policy that prevents an entire class of long-term decay — the slow accumulation of `NOTES.md` and `TODO.md` and `tmp-design.md` files at the root that confuse future agents about what's authoritative. By the time you have 22 agents working in the codebase, every one of them is trying to figure out where things go, and an inconsistent structure is an invitation for each of them to invent their own answer.

The single best thing I did for the Engineering team's effectiveness wasn't building any specific agent. It was committing — early and absolutely — to a folder structure that makes "where does this go?" answerable in one second by any agent who reads the path. The agents do the work. The structure makes the work legible.

## The Strict Delegation Rule

The single most important rule for the Engineering team is this: **the generalist agent is forbidden from touching service code.**

Not "discouraged." Forbidden. The repo's `CLAUDE.md` is explicit: the generalist cannot implement features in services, cannot read service code, cannot answer questions about service internals, cannot debug service issues, cannot review service code. Even **research** must be delegated — if the generalist needs to understand how the API service handles auth before drafting a plan, it doesn't open the file. It asks the API Engineer, who returns findings.

This sounds extreme. It is. The first few weeks of working this way felt deeply wasteful — *why am I asking another agent to read a file I could read myself in two seconds?* The answer became clear after the first time the API Engineer caught something the generalist would have missed: a service-specific convention, a tenant-scoping requirement, a load-bearing pattern that wasn't obvious from reading the file once but was obvious to the agent that had been living in that service for months.

The generalist sees breadth. The service expert sees depth. The rule prevents the generalist from pretending to do depth.

There's a second-order benefit that turns out to matter more: **the rule makes parallel work safe.** Two service engineers can work on two different services at the same time without coordination, because neither one is allowed to touch the other's code. The blast radius of any single agent is one folder.

## One Owner For Every Schema

The Data Architect owns the shared schemas directory. Every Alembic migration. Every column added to a table. Every new shared enum. Every message contract between services.

Service engineers are allowed to *request* schema changes. They are not allowed to make them. The workflow is mandatory:

1. Service engineer identifies a need
2. Files a request to the Data Architect
3. Data Architect designs the change with rolling-deploy safety in mind (expand-then-contract pattern)
4. Data Architect assesses risk (Low / Medium / High)
5. Data Architect tests the migration in both SQLite (dev-local) and PostgreSQL (dev-docker)
6. Data Architect verifies backward compatibility for the active deploy
7. Data Architect hands the completed migration back

The reason this is centralized: schemas are the only thing in the system where a single mistake can poison every service at once. Putting one specialist in charge means there's one mind tracking expand/contract patterns, one mind tracking which columns are still in use, one mind that's read every migration in the repo. It's the most expensive specialist on the team and the one I'd give up last.

## The QA Lead

QA on the team is one agent doing most of the work: the **QA Lead**. It ran tests in the early days. It still runs them now. The thing that's changed is how much more it does than a basic test runner would.

The QA Lead runs tiered tests calibrated to the size of the change. Most deploys are routine — a small fix or a single-service feature — and get *targeted* QA: the modified service gets unit and integration tests, plus a smoke test through the rest of the system to confirm nothing obvious broke. Auth or tenant-isolation changes get *security-focused* QA, where the QA Lead specifically tries to break the tenant boundaries. Pre-production releases get the *full* sweep: cross-tenant security tests, UI regression, exploratory testing, the works.

The thing that surprised me about the QA Lead is how much further it goes than a basic tester would. It thinks about negative cases. It tries cross-tenant access on its own initiative. It reads recent diffs and asks what's likely to be brittle. It writes findings that look like a junior engineer's incident report, not a stack trace. For a one-person team, that thoroughness is the difference between "I have tests" and "I have QA."

There are also two **specialized QA agents** wired up — one for Identity, one for Threat-Intel — both intended for services where a bug means a security incident, not a refund. Right now they're more "available" than "load-bearing." The QA Lead handles most of the everyday QA, including for those services, and handles it well enough that I haven't needed the specialists yet. They're there for the day I want to push deeper coverage in those areas. The model around when to delegate to them — what the QA Lead hands off, what stays in-house — is still being figured out. A few months from now I'll have a clearer story; today, the generalist is doing more than enough.

## The Pen-Tester

The **Application Pen-Tester** is unusual on a team this size, and worth a moment on its own. It runs OWASP-aligned penetration tests against the test environment — reconnaissance, attack surface enumeration, authentication bypass attempts, injection probing, IDOR checks, business logic flaws. The full methodology, structured findings, no prod targets.

It's not always-on. I invoke it before major releases, after auth changes, and on a quarterly cadence. It's not a replacement for an external assessment when one is needed, but it catches the long tail of "did we leave something obvious open" that would otherwise sit there until someone else found it. For an application security platform that's about to land in users' hands, having this loop running on my own product feels less like a luxury and more like the bare minimum.

## The Troubleshooter: No Fixes, Only Diagnoses

When something breaks in a way I can't immediately explain — a flaky test that started failing two days after the code changed, a deployment that succeeds but the service doesn't accept traffic, an integration that worked yesterday and silently doesn't today — I don't ask the service engineer to fix it. I call the **Troubleshooter**.

The Troubleshooter has one operating rule, and it's the rule that defines the role: **find the root cause before anyone fixes anything. And never do the fix yourself.**

Not the symptom. Not the proximate trigger. The actual underlying cause. The Troubleshooter's job ends when he hands a clean diagnosis to the right specialist with a clear "here is what's wrong, here is the evidence, here is what would resolve it." The fix happens in the owning agent's hands — never in the Troubleshooter's.

This sounds obvious. It isn't. The natural pull when something is broken is to start trying things — restart the service, bump a version, tweak a config, see if the error goes away. Sometimes that "fixes" the problem in the sense that the symptom stops recurring. Sometimes the underlying cause is still there, waiting to come back in a slightly different form a week later. The Troubleshooter is structurally prevented from taking that shortcut. He diagnoses; he doesn't fix.

By the time he's done, four things exist that didn't before:

1. The actual cause is named, with evidence
2. The chain from cause to symptom is documented step by step
3. A specific recommendation goes to a specific specialist (Data Architect, Identity Engineer, DevOps Specialist — whoever owns the area where the cause lives)
4. The fix happens in that specialist's hands, with the diagnosis as context

The split is the same pattern as Architect-vs-Builder, just for incidents instead of features. **Planning is one job. Execution is another. Putting them in different agents prevents the most expensive mistake of all — fixing the wrong thing because you stopped looking for the right cause.**

This is the agent I call for the *harder* issues — the ones where I can stare at the symptoms for ten minutes and not even know which service to start in. Easy bugs go straight to the service engineer; the Troubleshooter is reserved for the ones where the surface and the source aren't obviously related. It's also what `/incident` invokes by default — the skill triages service health, checks recent deploys, inspects logs, and launches the Troubleshooter with structured context so he starts with a real picture instead of a blank prompt.

The diagnosis-without-fixing rule has a second-order effect that took me a while to appreciate: it slows me down in exactly the right places. The instinct in an outage is to act fast. The Troubleshooter forces me to act *correctly* fast — by separating "I know what's wrong" from "let me make it stop." Most outages I've worked through were quicker to resolve once that split was enforced, because the second attempt at a fix is far more expensive than the first attempt at a diagnosis.

## Skills: Muscle Memory You Can Name

Beyond agents, the team has accumulated **9 skills** — small, named workflows that codify "the way we do this here." Each one started as a sequence of commands I was running by hand until I got tired of typing it.

| Skill | What It Codifies |
|------|------|
| `/project start {issue}` | Creates a branch, links to a project board card, sets up the workflow for a single issue. **Mandatory entry point for every work item.** |
| `/test {service}` | Runs the right test suite for the right service in the right environment (dev-local, dev-docker, or both). |
| `/migrate {service} {description}` | Orchestrates a database migration safely — delegates to the Data Architect, runs the dual-dialect test. |
| `/qa` | Runs comprehensive 7-stage E2E testing against the test environment. The QA Lead's standard playbook. |
| `/preprod` | The pre-production gate. Smoke + API E2E + browser E2E. Returns a green-or-red verdict. Never skipped. |
| `/deploy` | Full production rollout workflow. Validates VERSION bumps, runs tests, creates a GitHub release, promotes to prod with checkpoints. |
| `/incident` | Systematic incident response. Triages service health, checks recent deploys, inspects logs, launches the Troubleshooter with structured context. |
| `/review` | Pre-push code review. Tenant-isolation checks, auth-gap checks, service-specific rules. |
| `/changelog` | Generates a changelog entry from commits and diffs. Drops it in `docs/changelog/`. |

Skills are not magic. They are commands you reuse so often that giving them a name is faster than describing them. The pattern: *if you've done it three times the same way, it's a skill.*

## MCP, In Two Directions

This is where the story gets interesting.

**What the team consumes.** Three MCP servers are wired into the team's setup:
- **Playwright MCP** — used by the QA Lead and the Application Pen-Tester for browser automation. Real pages, real forms, real screenshots.
- **WebFetch / WebSearch** — used by every agent for research. Looking up CVE details. Reading vendor docs. Checking if a library still ships.

**What the team ships.** This is the meta-twist. Intercept itself **distributes its own MCP server** — `intercept-mcp` — as a public PyPI package. Users run `uvx intercept-mcp`, point it at their Intercept tenant, and their own Claude Code (or any MCP-compatible agent) can read their security data: organizations, repositories, scans, findings, exposures, posture, secrets, container findings, IaC findings, the full tool catalog.

The reason that matters: **the platform was built with AI, and the platform was built for AI.** Users will work with Intercept the way I built it — with an agent in the loop, asking questions, making changes, integrating into their own workflows. Shipping our own MCP server isn't a side feature; it's the bet that the next generation of security tooling lives inside the developer's AI workflow, not outside of it.

There's more in the same direction. The product also ships:
- **AI Repository Analysis** — per-commit LLM analysis of every repository in a tenant. Generates a business summary, identifies the application domain, classifies the architecture style, rates security maturity, enumerates threats with high-risk counts, inventories components and frameworks. Token usage and model are tracked per analysis. This is product-level threat modeling, generated from the code itself.
- **Universal AI Provider integration** — two options for any user: bring your own LLM key (Anthropic, OpenAI, etc.) with a privacy gate, identity wiring, and per-tenant scope, or use Hijack's. Users who bring their own key keep their data on their chosen provider; users who pick the Hijack option get the same AI features without managing a key themselves.

The pattern: every place AI is making my engineering work easier, AI is also being built into the product for the user's benefit.

## Hooks: The Constitution, Enforced

Rules in prompts can be ignored — by accident, by hallucination, by an agent in a hurry. **Rules in hooks cannot.** The Engineering team's constitution is enforced by **8 hooks** that fire automatically inside Claude Code:

| Hook | Fires On | What It Blocks |
|------|----------|----------------|
| `pre-commit-quality.sh` | Bash commands (commit) | Direct commits to `main`. Commits without an issue reference. Commits missing a VERSION bump for modified services. Commits missing doc updates. Temp files. |
| `pre-push-quality.sh` | Bash commands (push) | Pushes that fail validation gates. |
| `block-sensitive-files.sh` | Write / Edit | Staging of `.env`, `.env.*`, or any secrets file. |
| `block-worktrees.sh` | Agent / Bash | The `isolation: "worktree"` parameter. The `git worktree` command in any form. Hard block, exit code 2. |
| `cleanup-temp-files.sh` | After Bash | Removes test artifacts before push. |
| `ruff-autoformat.sh` | After Write / Edit | Auto-formats Python with Ruff. No formatter wars. |
| `post-edit-reminder.sh` | After Write / Edit | Reminds the agent to test/review what it just changed. |
| `check-tenant-scoping.sh` | After Write / Edit | Validates that tenant filtering is present in queries. Catches the subclass of bugs that lead to data leaks. |

The hooks are the difference between "the rule exists" and "the rule binds." When an agent tries to commit to main, it doesn't get a polite refusal — it gets a hard error and the commit doesn't happen.

## Why Worktrees Don't Work For Us

Claude Code supports **worktrees** as a pattern for parallel work — spawn multiple agents in their own isolated checkouts, let them work in parallel, merge later. It's elegant. It's powerful. It does not work for us.

The reason is the hooks. The pre-commit-quality hook checks branch state, VERSION files, doc updates, and tenant scoping across the *entire repository*. Worktrees create parallel checkouts that diverge from each other in ways the hooks weren't designed for. The result was a steady drip of failed hooks, stale state, and broken commits that took longer to debug than the parallelism saved. The fix was to ban worktrees outright (`block-worktrees.sh`) and find a different way to parallelize.

Which leads us to:

## The Planning Window

There's an agent on the periphery of the Engineering team that doesn't fit on the org chart, because it doesn't live in the team's main Claude Code session at all. It lives in **another Claude Code window**, open beside the engineering one, all day. I call it the **Project Manager**.

The PM doesn't write code. It writes plans. When I have an idea, a user concern, a half-formed feature thought, a bug I need to remember to file — I bring it to the PM, not to the engineering team. We talk it through. We figure out what's actually being asked. The PM helps me decide what's in scope, which services would be touched, what the rough size is, where it sits in the priority queue.

When something is ready to be work, the PM creates the GitHub issue: clean title, clear description, the right service labels applied, added to the project board. The issue then sits in the backlog. The engineering team doesn't see it until I switch windows and run `/project start {issue}`.

The reason the PM lives in a separate window: **planning needs a different headspace than execution.** Trying to do both in the same session means the engineering session keeps getting pulled out of focus, asked about prioritization, dragged into "should we build this?" conversations that have nothing to do with the code at hand. Splitting the windows splits the modes. The PM is in a permanent planning mindset; the engineering team is in a permanent building mindset.

The second-order benefit is that **I can plan ahead while the team is building.** While Engineering is mid-flight on one issue, I can be in the PM window thinking through the next three — three issues out, three different services, three different scopes — and the engineering work doesn't pause. By the time the team finishes, the next three issues are queued, labeled, and ready to start.

It's the closest thing this setup has to two people working at the same time. Just two of me, in two windows, doing two different jobs.

<div class="mermaid">
graph LR
    ME[Founder]
    PM[PM<br/>separate Claude Code window]
    BOARD[Project Board<br/>labeled by service]
    TEAM[Engineering Team<br/>main Claude Code window]

    ME -.->|ideas, requests, half-thoughts| PM
    PM -.->|drafts issues + service labels| BOARD
    ME -->|switches windows| TEAM
    BOARD -->|"/project start {issue}"| TEAM
</div>

## Parallel Work Via Issue Labels (And Sometimes Two Machines)

Every issue on the project board is created by the PM (above), with the right service labels applied before the engineering team ever sees it. Once an issue exists, those labels do double duty: they tell *me* what's in flight, and they tell the *agents* whose work it is.

When two changes target two different services, they can move through the team in parallel. The Identity Engineer is working on one (label: `identity`). The Worker Engineer is working on another (label: `worker`). They never read each other's code. The Technical Architect's plan for one doesn't bleed into the other. Both branches go through their own pre-prod gate. Both get reviewed. Both ship.

Occasionally — when the changes are big and time is short — I'll have **two machines** running simultaneously, one branch per laptop. Issue labels are what makes that safe. Without the strict per-service ownership rule, two parallel sessions could collide. With it, they can't.

Labels turned out to be the cheapest, most powerful coordination mechanism on the team. They don't require a special tool. They don't require a hook. They require the PM to label issues correctly when filing them, and the rest is downstream.

## What The System Catches

Composite stories — generic, but representative:

- **The schema migration that would have broken rolling deploys.** Service engineer requested a migration that dropped a column. The Data Architect rewrote it as expand-then-contract: add the new column, deploy, backfill, deploy, drop the old column. Three deploys instead of one. No downtime, no broken rolling restart.
- **The cross-service feature that didn't fan out cleanly.** Architect's plan touched four services. Reading the plan, I noticed the auth flow assumed a header that the gateway didn't pass through. Two-line fix in the plan. Without the architect step, the bug would have shown up in QA.
- **The tenant-scoping miss that the post-edit hook caught.** Service engineer wrote a query that filtered by repository ID but not tenant ID. The `check-tenant-scoping.sh` hook flagged it the moment the file was saved. The engineer fixed it before opening a PR.
- **The version-bump miss that pre-commit-quality blocked.** Engineer made a one-line fix and forgot to bump the service VERSION. Hook blocked the commit, named the service, named the file. Fix was three keystrokes.

None of these are heroic. All of them are unglamorous. That's the point: the system catches the small, frequent mistakes so I don't have to remember to.

## Honest Limits

The Engineering team is not magic. Things it doesn't do well:

- **Cross-team handoffs.** The Engineering team coordinates well within itself, but handoffs to the Infrastructure team (Part 2.1) still go through me. I want to wire the two teams together more directly, but the trade-off (more coupling, more surface to review) hasn't tipped yet.
- **Long-horizon refactors.** A multi-week refactor that requires holding a complex architectural intent across many sessions is still my job to coordinate. The agents can execute beautifully on a one-day plan; a one-month plan is still a human-shaped problem.
- **Judgment calls on product direction.** "Should we build this feature?" is not an Engineering team question. The team builds what gets asked for.

## Lessons That Travel

If you're building something with AI — whether you're solo or running a team — six things from this experience that survive transplant:

1. **Specialize as you grow, not in advance.** Don't pre-design 22 agents. Start with one, and split off a specialist the moment generalist-grade work in that area starts costing more than the specialist would. The reason subagents are powerful isn't just cost — **each subagent maintains its own context**, doesn't get overwhelmed by the rest of the repo, can go deep into its specific responsibility, and can run in parallel with other subagents working on unrelated changes. Specialization is a context-window strategy as much as it is an org-design strategy.
2. **Make permissions match responsibility.** The generalist isn't allowed to touch service code, not because of a polite norm but because of a hard rule. The Data Architect is the only one who writes migrations, period. Permissions are how you make rules binding.
3. **Single owner for shared concerns.** Schemas. Deploys. Security policy. The thing that touches everything needs exactly one mind tracking it. Anything else and you'll get drift.
4. **Skills are just named workflows.** If you've done it three times the same way, name it. Future-you will reach for the name instead of remembering the steps.
5. **Hooks beat prompts.** A rule in an instruction file is a suggestion. A rule in a hook is a constraint. When the rule actually matters, put it in the hook.
6. **Parallelize via labels, not worktrees, when your standards are strict.** Issue labels + per-service ownership + branch protection gets you most of the benefit of parallel work without the cost of divergent checkouts.

There's a seventh, harder one to articulate: **trust the team to surprise you.** The first few weeks of strict delegation feel slow. The first time the system catches something you wouldn't have caught is when you understand why you built it.

## Next

[2.3 Building Intercept: The Operations Team — Always On, On a Laptop](/2026/05/02/building-intercept-operations-team/) covers the team that watches it all run — four launchd-scheduled agents on a single MacBook, daily Dependabot patching with Slack-thread approval, role-based hooks bounding what each agent can do, and the 53-hour battery-hibernation outage that taught me what 24/7 actually requires.

---

*Part of [Building Intercept](/2026/05/02/building-intercept-introduction/) — a series on building an application security platform with three AI teams.*
