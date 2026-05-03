---
layout: post
title: "2.3 Building Intercept: The Operations Team — Always On, On a Laptop"
date: 2026-05-02 13:00:00 -0000
categories: ai operations devops monitoring slack
tags: ["Building Intercept", "Operations", "AI", "Monitoring", "DevOps"]
series: "Building Intercept"
series_part: "2.3"
---

My phone buzzes in the morning. The Patch Engineer's report is in Slack — three CVEs in dependencies overnight, all bundled into a single PR for the day, green, merged, live in test. I read it on the lock screen, confirm everything looks healthy, and move on.

By the time I sit down at the laptop later, the rest of the day's reports are waiting. The QA Engineer's full Playwright walkthrough — fourteen screenshots, no regressions. The Junior Sysadmin's egress check — one suspicious outbound domain blocked by the proxy overnight, classified as "unknown," GitHub issue filed automatically. The hourly health checks fired through the day. Two flagged elevated latency on a backend service; both resolved before the next check fired and the issues auto-closed.

None of this happened because I was at the laptop. All of it happened because the laptop was running.

<div style="text-align: center; margin: 30px 0;">
  <img src="/assets/images/ops.png" alt="The Operations Team — four hooded watchkeepers around a circular workbench, a single open laptop glowing in the center, ascending sparks of Slack messages drifting upward through a circular skylight" style="max-width: 100%; height: auto; border-radius: 8px; box-shadow: 0 2px 12px rgba(0,0,0,0.15);">
  <p style="font-size: 0.9em; color: #666; margin-top: 8px;"><em>The Operations Team — four watchkeepers, one always-on laptop, messages drifting outward to the operator wherever they are.</em></p>
</div>

This is the **Operations team** — the always-on team. It runs on a single MacBook that doesn't sleep, uses Claude Code CLI (not the API), and posts everything to Slack. There is no server, no Kubernetes, no AWS Lambda. There is a laptop, a launchd configuration, and four agents that wake up on schedule, do their jobs, and exit.

This series ends here, and this team is the most unusual of the three — not because it's the most sophisticated, but because it's the most committed to a single design idea: **the laptop is the server.**

## The Laptop Is The Server

The Operations team is built on the **Claude Code CLI** — `claude -p --agent <name>` invocations spawned from shell scripts — not the **SDK**. The choice was originally pragmatic: I'm already paying for the Claude Code subscription to build the rest of Intercept, so the marginal cost of running ops work on the same subscription is zero. The API would have meant per-token billing for every health check, every patching run, every QA walkthrough — a fine answer at large scale, but not the right shape for a one-person setup.

Once you commit to "the laptop is the server and the CLI is the runtime," everything else falls into place. Stateless invocations. Single-shot processes. No long-running sessions. No retries inside an agent. Failures isolated to one job. Recovery via launchd. The whole architecture writes itself when you start from those primitives.

The laptop sits on my desk, plugged in, lid open, 24/7. It runs `caffeinate -dimsu` to stay awake, runs hourly health checks against production, runs daily patching at 6 AM, runs a daily app walkthrough at 7 AM, and posts to Slack throughout. The detail that took me a while to appreciate: **this old Mac doubles as my own ops bench.** When I sit down at it, the same machine that's been running checks all day is the one I use to dig into a kubectl logs, fix something on the website, or write through a problem. The proximity is the point. The team isn't off in some cloud account I have to context-switch into; it's right there on the desk, running the same scripts, hitting the same clusters, posting to the same Slack channels. That's what gives the whole setup its real ops vibe — it's a working operations environment, not an automation cron-stack.

## The Architecture: Launchd-Native, Stateless

Every scheduled job is its own launchd service. When the schedule fires, launchd executes a shell wrapper, the wrapper execs a Python module, the Python module spawns `claude -p --agent <name>` as a single-shot subprocess, the agent does its work, posts to Slack, exits. No long-running Claude session. No in-process orchestration. If a job fails, only that job fails — the next one fires on its own schedule independently.

<div class="mermaid">
graph TB
    LAUNCHD[launchd]

    subgraph Pure-Shell Services
        HB[heartbeat<br/>every 15m]
        DS[daily-summary<br/>21:00]
        WS[wake-sweeper<br/>every 30m + on wake]
        CAF[caffeinate<br/>always-on]
    end

    subgraph Scheduled Agent Jobs
        HC[health-check<br/>hourly → Junior Sysadmin]
        PT[patching<br/>06:00 → Patch Engineer]
        AC[app-check<br/>07:00 → QA Engineer]
        EC[egress-check<br/>08:00 → Junior Sysadmin]
    end

    subgraph On-Demand
        MP[message-poller<br/>every 15s]
        RT[Shift Supervisor Router<br/>spawned when messages arrive]
    end

    LAUNCHD --> HB
    LAUNCHD --> DS
    LAUNCHD --> WS
    LAUNCHD --> CAF
    LAUNCHD --> HC
    LAUNCHD --> PT
    LAUNCHD --> AC
    LAUNCHD --> EC
    LAUNCHD --> MP
    MP -->|new messages?| RT

    HB -.-> SLACK[Slack]
    DS -.-> SLACK
    WS -.-> SLACK
    HC -.-> SLACK
    PT -.-> SLACK
    AC -.-> SLACK
    EC -.-> SLACK
    RT -.-> SLACK
</div>

There are three tiers of service:

1. **Pure-shell services** — `heartbeat`, `daily-summary`, `wake-sweeper`, `caffeinate`. Plain bash. No Claude invocation. Cheapest possible runtime for the work that doesn't need reasoning.
2. **Scheduled agent jobs** — the four daily/hourly cron-style jobs that each spawn a fresh Claude CLI process, run one task, exit.
3. **The on-demand router** — `poll_messages.py` checks Slack every 15 seconds; if there are new operator messages, it spawns the Shift Supervisor Router agent with the message batch. The router routes to the right specialist subagent and exits. No persistent supervisor process running in the background.

That's the entire runtime. Boring on purpose.

## The Team Today: Four Agents

Smaller team than the Engineering side, by design. Each agent has one job and one job only.

| Agent | Model | What It Does |
|------|------|---------|
| **Junior Sysadmin** | Haiku | L1 read-only platform monitor. Hourly health checks across the cluster, application, and egress proxy. Escalates anomalies via a dedup-aware GitHub issue + Slack alert; closes the issue itself on a later run if the condition has cleared. Cannot modify anything. |
| **Patch Engineer** | Opus | Daily dependency patching. Pulls Dependabot alerts, patches, runs tests, opens PRs, deploys to test, asks for Slack approval before prod. |
| **QA Engineer** | Sonnet | Daily browser walkthrough of the application via Playwright. Fourteen screenshots per run, posted to Slack. Catches visual regressions, broken auth, dead pages. |
| **Shift Supervisor Router** | Sonnet | Stateless per-message router. Spawned only when there's an operator message in Slack. Routes to the right subagent, replies, exits. Never runs operational commands itself. |

Three things to notice about the model choices:

- **Haiku for L1.** The Junior Sysadmin runs hourly. That's 24 invocations a day before counting catch-ups. It does read-only health checks — the work is high-frequency and low-reasoning-depth. Haiku is the right tool, and it's the cheapest one.
- **Opus for patching, but only once a day.** The Patch Engineer is the most expensive agent on the team, but it only fires at 06:00. Once. The work it does is genuinely complex — assess Dependabot alerts across multiple repos, patch dependencies without breaking things, run tests, open PRs, monitor deploys, request approval. Opus earns its keep on the one run a day where everything has to go right.
- **Sonnet in the middle.** QA and the router get Sonnet — more reasoning than Haiku can do reliably, less than Opus needs to bring.

The whole team is built around matching model cost to work value. A team that ran everything on Opus would be ~3x more expensive and not meaningfully better.

## The Permission Model: Hooks as Guardrails

Like the Engineering team, the Operations team relies on **hooks** to enforce what each agent is allowed to do. But where the Engineering team's hooks block specific actions (no commits to main, no worktrees, no missing VERSION bumps), the Operations team's hooks enforce role-based **command-level allowlists**.

Each agent has a role YAML file. The role file lists exactly which tools the agent can use and which Bash commands it can run, down to glob patterns. A `PreToolUse` hook (`permission_guard.py`) reads the agent's role from an environment variable, looks up the allowlist, and decides — before the tool call ever fires — whether the action is permitted.

What this looks like in practice:

| Role | Can Run | Cannot Run |
|------|---------|------------|
| **Junior Sysadmin** | `kubectl get *`, `kubectl describe *`, `kubectl logs *`, `aws cloudwatch *`, `aws ec2 describe-*`, `curl *` (for health checks), `gh issue create/list/view` | Anything that mutates state. Period. |
| **Patch Engineer** | All of the above plus: `git *`, `gh pr *`, `uv lock/sync/run *`, `npm update/install/test`, `pytest *`, `docker compose up/down`, `gh workflow run *` | Infrastructure changes. Direct prod deploys without approval. |
| **QA Engineer** | `curl *`, MCP Playwright (full browser automation), `gh issue create`, file operations | Anything outside the test environment. Anything that changes state in production. |
| **Shift Supervisor Router** | MCP Slack send, GitHub issue helpers, reading `state/` files. That's it. | Operational commands. Period. The router routes; it doesn't act. |

There's also a **global blocklist** that overrides every role: `kubectl port-forward` and `kubectl exec` are denied no matter who's asking, because the cost of getting those wrong is too high to leave to a per-role exception. Some commands are dangerous enough that the answer is always no.

The whole thing fails closed: if the role file can't be loaded, if the Python interpreter can't be found, if the command doesn't match any allowlist pattern — the answer is deny. An agent that's confused about what it can do does nothing.

## Daily Patching: The Workflow That Earns Its Keep

If the Operations team has a marquee feature, it's daily patching. This is the agent that pays for itself in saved hours every single day.

The workflow runs at 06:00 and goes like this:

1. **Assess.** Patch Engineer fetches open Dependabot alerts across all monitored repos. If nothing's open, posts an "all clear" to `#patching` and exits.
2. **Track.** For each repo with alerts, opens (or updates) a tracking GitHub issue. The helper script is dedup-aware — if there's already an open issue for today, it updates that one instead of stacking duplicates.
3. **Patch (one branch per repo per day, all that day's CVEs bundled).** For each repo, on a single fresh branch named `patch/<primary-package>-YYYY-MM-DD`: bump every alerted dependency via the right tool (`uv` for Python, `npm` for Node), bump service VERSION files where the project has them (CI/CD won't deploy without them), run the test suite. On test failure: one fix attempt, then escalate. On test pass: commit and push.
4. **PR + test deploy (one PR per repo, bundling that repo's whole day of patches).** PR target depends on the repo's deploy model: `main` for the Kubernetes-deployed service (merging triggers test deploy via CI), `test` branch for Vercel-deployed sites (merging triggers a Vercel test deploy; prod stays separate). Merge after CI passes.
5. **Verify.** Check pods healthy on the cluster, or check that the test URL returns 200, depending on the deploy target.
6. **Report and approval prompt — one Slack message.** Post a rich status to `#patching` listing every patched package across every repo: old → new versions, CVE/GHSA IDs, severity, PR links, test verification links, tracking issue links. The same message ends with `Reply *Approve* to this message to deploy to prod.` That's the trick: **all the day's patches across all the monitored repos collapse to a single Slack message and a single approval reply.**
7. **Deploy to prod.** When I reply "Approve" in the Slack thread — often from my phone, before I've sat down at the laptop — the message poller picks it up and triggers prod deploy for each patched repo. Verify pods. Close tracking issues.

Three monitored repos: the Intercept platform itself, the marketing website, and a portfolio site. Most days only one of them has a new CVE; some days two; rarely all three. Whatever the day looks like, the bundling rule is the same — one PR per repo, one Slack message at the end, one Approve reply that covers the whole day's prod deploys.

The reason this workflow matters more than the others isn't sophistication — it's **frequency**. CVEs drop constantly. Deploys are tedious. The five minutes I'd spend manually triaging each Dependabot alert add up to hours per week. The Patch Engineer compresses that into the time it takes me to read a Slack thread and tap "Approve."

## The Slack Interface

Everything the Operations team produces ends up in Slack. Three channels do all the work:

- **`#operators`** — operator visibility. Loud alerts (battery low, host offline, anything urgent), my conversations with the team, escalations from the Junior Sysadmin.
- **`#ops-status`** — muted by design. Heartbeat every 15 minutes, daily summary at 21:00, routine completion messages from scheduled jobs. The channel I scroll through when I want to know "is everything fine," not the channel that pings me.
- **`#patching`** — the patching workflow. Every day's patch run, every approval thread, every prod deploy, isolated from the rest of the operator chatter.

Outbound goes through the **MCP Slack plugin** — agents post via the MCP tool and the message renders with rich Slack formatting. If the MCP post fails for any reason, there's a fallback path in `run_job.py` that posts via `chat.postMessage` directly with a bot token. Two paths, one of which always works.

Inbound is `poll_messages.py` calling `conversations.history` every 15 seconds against `#operators`. When new operator messages exist, the poller spawns the Shift Supervisor Router with the message batch embedded in the prompt. The router decides which agent should handle each message, hands off, and exits. No persistent listener, no WebSocket, no Slack Events API integration to maintain — just polling, which is dumber and harder to break.

The important thing this enables: **the team is not a black box of cron jobs running on autopilot.** I can talk back. If I want a status check between scheduled runs, I send a message in `#operators` and the Junior Sysadmin picks it up within fifteen seconds. If I want the QA Engineer to walk through a specific page after a deploy instead of waiting for tomorrow morning's full sweep, I ask. If I want the Patch Engineer to look at one specific Dependabot alert that just dropped, I ask. Whatever the request, the router parses it, the right specialist takes it on, and the answer comes back in the channel. The same way I'd Slack a real ops engineer, except the ops engineer's response time is measured in seconds.

This is unglamorous on purpose. The Slack integration could be more sophisticated. It doesn't need to be.

## Issues As Support Tickets

When the Junior Sysadmin finds something it can't or shouldn't fix on its own — an unhealthy pod that won't come back, an unfamiliar domain blocked by the egress proxy, a CloudWatch alarm that's been firing for the past hour — it doesn't just shout in Slack and forget. It opens a GitHub issue.

The issue functions like a support ticket. It captures what was found, when, and what context the agent had at the moment of escalation. The issue gets added to the project board automatically and tagged with the relevant service label. Slack gets a one-line alert pointing at the issue link.

Two things make this work as an actual ticket queue rather than a noise stream:

1. **Tickets are dedup-aware.** The helper script that creates issues checks for an open issue with the same title before opening a new one. Hourly health checks against the same flaky service don't stack twelve duplicate issues — there's one ticket, and it gets new context as the check repeats.
2. **Tickets auto-close when the condition clears.** Every scheduled run scans the project's open issues. If an issue exists for a problem that no longer reproduces — pod is healthy now, alarm has cleared, the egress request hasn't recurred — the agent closes the issue with a short comment about what changed. By the time I open the laptop, half the tickets that were open the night before are already gone.

The result is a ticket queue that mirrors a real on-call rotation. If something matters and persists, it stays in the queue with my name on it — or routes to the engineer who owns the affected service. If something flapped and resolved on its own, it's already cleaned up before I look. I only deal with the tickets that earned my attention.

The same pattern works in the other direction: when I (or another agent) ask the Junior Sysadmin to investigate something, it can file a ticket on my behalf if the answer is "this needs a fix you'll want to track." Tickets become the unit of escalation that survives the agent's exit — Slack messages scroll, but a ticket stays open until something happens to it.

## The 24/7 Story (And The Outage That Taught Me)

The hardest part of running ops on a laptop isn't writing the agents. It's keeping the laptop awake.

`caffeinate -dimsu` runs as a launchd service to prevent the Mac from sleeping or dimming. That covers the easy case. But `caffeinate` cannot prevent the harder case: a battery that runs to zero. If the Mac unplugs and drains overnight, it forces a Hibernate, and Hibernate doesn't auto-recover when you plug it back in.

I learned this the hard way over a 53-hour outage at the end of April 2026. The Mac unplugged itself — a loose cable. The battery drained. The Mac hibernated. None of the agents fired. The heartbeat channel was muted, so I didn't see the absence of heartbeats. I noticed Sunday morning when I went to check on something and the laptop was off.

That outage produced the three layers of defense the team runs with now:

1. **Power-aware heartbeat.** `heartbeat.sh` now reads `pmset -g batt`. The moment the Mac is on battery, it posts a loud alert to `#operators` (not to the muted `#ops-status` where I'd never see it). The alerts escalate at 5 minutes, at 30 minutes, and below 30% charge. One battery dip and I get a phone notification.
2. **Downtime-detecting wake-sweeper.** `wake-sweeper.sh` runs every 30 minutes *and* on every wake event. It tracks its last-run timestamp in a state file. If the gap between sweeps exceeds 90 minutes, it knows the Mac was offline and posts a recovery alert. The signal that was missing during the outage — *"hey, the host was just down for X hours"* — is now the first thing in `#operators` after a recovery.
3. **Auto-wake via `pmset`.** `install.sh power-setup` runs `pmset repeat wakeorpoweron MTWRFSU 05:55:00` once. The Mac now wakes itself up every weekday at 5:55 AM, five minutes before the 06:00 patching job. Even if it slept overnight for some reason, it'll be awake for the morning rotation.

The hard rule is documented in the team's `CLAUDE.md`: **the Mac must stay on AC power.** Software cannot prevent dead-battery hibernation; it can only detect it and alert. Everything else is recovery, not prevention.

This is the kind of operational lesson you only learn by losing 53 hours.

## Cost-Conscious By Design

A few patterns that keep the team affordable:

- **CLI not SDK.** Every Claude invocation is a `claude -p` subprocess against the Claude Code subscription, not a metered API call. The subscription is paid; the marginal cost of one more invocation is zero.
- **Per-agent model tier.** Haiku for L1, Sonnet for routing and QA, Opus for patching. Matching the model to the work is the difference between a sustainable team and a $500/month ops bill.
- **Single-shot, stateless.** No long-running sessions to drain context. Every job starts fresh, runs to completion, exits. Lower memory pressure on the laptop, simpler failure model, no surprise compute.
- **Pure-shell services where possible.** Heartbeat doesn't need an agent. Daily summary doesn't need an agent. Wake-sweeper doesn't need an agent. Bash plus `gh` plus `curl` does the work for free.
- **Per-job timeouts.** Every spawn has a hard timeout (600 seconds for jobs, 900 for the router). A runaway agent gets killed before it can eat the day.

The result: an always-on ops team that costs effectively nothing on top of what I'm already paying for the Claude Code subscription.

## What's Coming

The team works well today. It is also obviously incomplete.

Three directions I expect to grow it in over the next few months:

- **Incident response.** Right now, the Junior Sysadmin escalates by filing a GitHub issue and pinging me on Slack. There is no L2 agent that can actually remediate beyond the patching workflow. A "Senior Sysadmin" role with limited remediation rights — restart a deployment, scale a nodegroup, rotate a credential — would close the loop on the alerts the L1 surfaces.
- **Deeper monitoring.** Today's checks are health-and-egress shaped. There's room for cost monitoring (Kiro CLI from the Infrastructure team would slot in well here), latency tracking, error-rate alerting, anything CloudWatch-shaped that's currently a manual look.
- **More security.** A scheduled application security scan, periodic external pen-test runs (the Engineering team's pen-tester could become an Operations job too), credential rotation reminders, security audit log reviews.

The architecture supports all of this without restructuring. Add a new agent file, add a new role YAML, add a new entry to `config.yaml`, regenerate the launchd plist. The pattern handles it.

## What This Team Can't Do

Honest limits:

- **Anything when the laptop is off.** The Mac is the server. If the Mac is down, the team is down. That's the trade for not paying for cloud infrastructure.
- **Anything outside its narrow tool allowlist.** Hooks fail closed. If an agent encounters a situation that needs a command that isn't on its allowlist, it can't act — it can only escalate. Which is correct, but it means novel problems come back to me.
- **Real-time response.** The hourly health check fires hourly. The message poller polls every 15 seconds. If something breaks just after a poll, the team won't notice until the next one. For an application security platform, this is fine. For something life-critical, it wouldn't be.
- **Anything requiring infrastructure changes.** The Operations team can patch dependencies and restart services within its scope. Real infrastructure changes go through the [Infrastructure team](/2026/05/02/building-intercept-infrastructure-team/) — me, Claude Code, and Kiro CLI on a different laptop session.

## Lessons That Travel

Six things from this experience that survive transplant:

1. **Cost is a design constraint, not a footnote.** Picking CLI over SDK shaped everything that followed. If you can't afford the elegant solution, build the frugal one carefully and it'll teach you things the elegant one wouldn't have.
2. **Stateless single-shot beats long-running.** The simplest possible runtime for an always-on agent is "spawn fresh, do one thing, exit." Failure isolation comes for free. Recovery comes for free. State management goes from a problem to a non-issue.
3. **Match the model to the work.** Haiku at the bottom, Opus at the top, Sonnet in the middle. A team that runs everything on the most powerful model is a team that runs out of budget.
4. **Hooks for safety, allowlists for blast radius.** The same hook pattern from the Engineering team works in Ops with a different shape — instead of blocking specific bad actions, the Ops hooks enforce a positive allowlist of *only the actions this role can take*. Both patterns are useful; both should be in the toolkit.
5. **Pure-shell where possible.** Not every scheduled task needs an LLM. Heartbeats, summaries, sweepers — bash plus standard CLI tools can handle most operational glue, and the parts you'd want an agent for stand out more clearly.
6. **The outage that hurt is the outage that teaches.** The 53-hour battery hibernation took me three weeks to fully process and harden against. The defenses that came out of it — power-aware alerts, downtime detection, auto-wake — are the kind of thing you'd never sit down to write proactively.

There's a seventh that's harder to articulate: **the team I built around a deliberate frugal design ended up being one of the most satisfying parts of the whole project.** Watching the laptop run its rounds in the background, seeing Slack come alive with work that happened on its own — it feels less like running infrastructure and more like having a small reliable team I trust. Which is what an ops team is supposed to feel like.

## Closing

This is the third and final part of the *Building Intercept* series.

[2.1 Building Intercept: The Infrastructure Team — Scripts First, Always](/2026/05/02/building-intercept-infrastructure-team/) covered the layer underneath everything: AWS, EKS, scripts as the artifact, two AI tools mediated by a human bridge.

[2.2 Building Intercept: The Engineering Team — Specialists, All the Way Down](/2026/05/02/building-intercept-engineering-team/) covered the platform itself: 22 specialists, strict delegation, hooks as a constitution, the planning window in another Claude session.

This part covered the team that watches it all run: four agents on a laptop, launchd-scheduled, Slack-routed, cost-conscious by design, designed to grow.

The thing that ties all three together is the same insight, said three different ways. **You don't build with AI by handing one giant chat your whole project.** You build by organizing AI the way you'd organize a team — specialists with scopes, rules with consequences, and a human who reads every PR and approves every prod deploy. The structure is what makes the speed possible. Without the structure, you have a faster way to make the same mistakes.

Three teams. Twenty-eight agents in total. One founder. One product, days from going to a small group of trusted users. The teams have, without exaggeration, made building an entire application security platform alone something I can do in a sustainable amount of hours per week — without burning out, without dropping work on the floor, and without the structure getting in the way.

If you're considering building something like this — solo, small team, AI-augmented — the only honest advice I have is this: **the structure matters more than the model.** Pick any of the major frontier models and it'll be enough. The structure is what determines whether the system grows with you or collapses under its own weight.

And one more thing — I've learned a lot building this. About AI, about systems design, about what it actually takes to ship a software product alone. The teams are the visible artifact; the harder-won thing is everything I now understand about how to organize work between humans and machines so that both contribute their best. The series is over, but the learning isn't.

I'm glad you read this far. Thanks.

---

*This concludes the Building Intercept series. The [introduction](/2026/05/02/building-intercept-introduction/) ties the three parts together; each part stands alone but they're better read in order.*
