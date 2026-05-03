---
layout: post
title: "2.1 Building Intercept: The Infrastructure Team — Scripts First, Always"
date: 2026-05-02 11:00:00 -0000
categories: ai infrastructure aws devsecops eksctl
tags: ["Building Intercept", "Infrastructure", "AWS", "AI", "DevSecOps"]
series: "Building Intercept"
series_part: "2.1"
---

It's an ordinary Tuesday. The Engineering team has finished a service that needs its own database — isolated, encrypted, accessible only from the cluster. I switch to the Infra repo and start a conversation with Claude Code: what instance class makes sense for this workload, what backup retention, what subnet group, do we need a parameter group override. We talk it through until we're aligned on the shape of the thing. Only then do I ask for the script.

A few seconds later, two files appear in the diff: `install-rds.sh` and `delete-rds.sh`. I read both. The parameter group is still slightly off — easier to spot in the script than in conversation. I ask for a fix. I read the diff again. I run install. A few minutes later, the database is live. I close the laptop.

What didn't happen, and won't ever happen on this team:

- Claude Code did not call `aws rds create-db-instance` directly.
- Claude Code did not "just run a quick test" against my AWS account.
- Claude Code did not improvise.

Everything that mutates AWS in this codebase happens inside a script that exists on disk, was reviewed by me, and has a paired delete. That single rule — write it to a script first, *always* — is the spine of how the Infrastructure Team operates.

<div style="text-align: center; margin: 30px 0;">
  <img src="/assets/images/infra.png" alt="The Infrastructure Team — two AI craftsmen building modular cloud infrastructure under the human architect's review" style="max-width: 100%; height: auto; border-radius: 8px; box-shadow: 0 2px 12px rgba(0,0,0,0.15);">
  <p style="font-size: 0.9em; color: #666; margin-top: 8px;"><em>The Infrastructure Team — two AI specialists, one human reviewer, every change reviewed before it touches the cloud.</em></p>
</div>

## The Team Has Two Members

The Infra team is two AI tools with very different jobs.

| Tool | Role | Allowed To |
|------|------|-----------|
| **Claude Code** | The builder | Read repo, draft scripts, draft IAM policies, draft K8s manifests, open PRs |
| **Kiro CLI** | The AWS analyst | Query AWS — costs, running resources, recent changes, CloudWatch metrics, what's currently available in a region |

The split is the point. **Claude Code never touches AWS directly. Kiro never writes anything. The two never talk to each other.** I am the bridge between them. When I need to know what a current setup looks like before asking Claude to draft a change — what subnet IDs exist, what an existing security group allows, what the last month cost on NAT data transfer — I open Kiro and ask. The answer comes back to me. I bring that answer into my conversation with Claude as context. The script then gets written with the real state in mind, not a hallucinated version of it.

A note on the tool itself: **Kiro CLI is AWS's official agentic CLI** — the direct evolution of the Amazon Q Developer CLI, renamed and re-launched in November 2025. It's built on Claude frontier models, which means reasoning quality is in the same family as Claude Code; the difference is the *specialization*. Kiro is unusually current on AWS — newest instance generations, latest addon versions, what services are actually available in a given region today — and it has tight integration with the AWS APIs it queries. Its "Auto agent" picks the right model per question, so quick lookups stay cheap. The fact that I treat it as read-only is **my operational choice**, not a tool limitation — Kiro can do plenty of writing if you let it. I don't, because the two-tool split is the safety property.

In practice it's good at the questions you'd otherwise spend ten minutes piecing together with `aws ec2 describe-*` and `jq`. *"What's running in us-east-1 right now and what's it cost me this month, broken down by service?"* — one prompt, one answer.

The reason this two-tool split matters is exactly the reason permissions in the rest of the system matter: **a tool that can only read can't break anything.** The blast radius of a wrong Kiro answer is "I waste a few minutes." The blast radius of a wrong `aws ... delete` is "I waste a Sunday."

<div class="mermaid">
graph LR
    REQ[Request from<br/>Engineering or Ops]
    ME[Founder]
    KIRO[Kiro CLI<br/>read-only AWS]
    CC[Claude Code<br/>script writer]
    SCRIPT[install-X.sh<br/>+ delete-X.sh]
    REVIEW[Human review]
    RUN[Run script]
    AWS[AWS resource live]

    REQ --> ME
    ME -.->|queries state| KIRO
    KIRO -.->|answers| ME
    ME -->|context + intent| CC
    CC --> SCRIPT
    SCRIPT --> REVIEW
    REVIEW --> RUN
    RUN --> AWS
</div>

## Why eksctl, Not Terraform

Picking the IaC tool is one of the first decisions an Infra team makes. I went with `eksctl` plus shell scripts. No Terraform. No Pulumi. No CDK.

This is a load-bearing call worth defending.

| Choice | eksctl + shell | Terraform |
|--------|---------------|-----------|
| Setup time to first cluster | Minutes | Hours |
| Learning curve | Low (shell + a CLI) | Significant |
| State management | None — eksctl reads AWS | State file, locking, drift |
| Power | EKS-shaped | Multi-cloud, all resources |
| What an AI agent has to know to be useful | A few CLI flags | A whole DSL + provider model |
| Solo-builder fit | High | Lower |

`eksctl` wraps CloudFormation under the hood — it's not magic, just a focused front-end for one job. For a one-person setup that runs entirely on AWS and lives entirely on EKS, that focus is a feature.

The trade is real: I lose multi-cloud portability and the rich Terraform ecosystem. I'm fine with that trade *today*. If I were running ten teams across three clouds, I'd reconsider — but I'm not. The point of picking simpler tools as a solo builder is that complexity is not free, and the bill is paid in attention.

There's a second reason that matters more: **shell scripts are easy for AI to write correctly and easy for me to review.** A 60-line shell script with `aws` and `eksctl` calls is something I can read top to bottom in two minutes and reason about completely. A 600-line Terraform module that calls four other modules is something I can review *theatrically* but not actually understand in one sitting. When AI is the author, the easier the artifact is to read, the better the gate works.

## The Script-Pair Pattern

This is the pattern that everything else hangs on.

Every directory in the Infra repo that *creates* something in AWS has at least two scripts:

```
infra/test/eksctl/external-secrets/
├── install-external-secrets.sh
└── delete-external-secrets.sh

infra/shared/ecr/
├── create-ecr.sh
└── delete-ecr.sh

devops/<env>/rds/<service>/
├── install-rds.sh
└── delete-rds.sh
```

If you provision it, you can tear it down. If you can't tear it down, you didn't really build it — you grew it.

Some directories add a third or fourth script — `update-X.sh` for in-place changes (a policy revision, a parameter tweak), or `sleep-cluster.sh` / `wake-cluster.sh` to scale the test cluster nodes down to zero between sessions and back up when I'm ready to work. Same discipline applies: any change to live state goes through a reviewed script. Resizing a nodegroup, bumping a max, swapping an instance class — all just edits to a script that gets read before it runs.

Above all of it, every environment has a `cleanup-all.sh` that walks the entire dependency graph in reverse and ends at `$0` charges. For the test cluster this matters every session: spin it up when I sit down to work, tear it down when I'm done, never watch the AWS bill creep upward between sessions.

Three things this pattern enforces, almost as a side effect:

1. **AI has to write reversible work.** The model can't draft an `install-X.sh` and shrug about cleanup. The pattern is the assignment.
2. **Deletes are first-class.** The delete script is reviewed with the same rigor as the install. Most production incidents I read about online involve a delete that was never tested. Mine are tested every time the test cluster cycles.
3. **The IAM policies, security groups, and odd little side resources get cleaned up too.** Custom IAM policies created during install are deleted on teardown. So is anything else the install touched. "$0 after teardown" is the contract, and it's only honored if the delete actually deletes everything.

## Pin It or Don't Ship It

Nothing in this repo runs on `latest`.

- Kubernetes version: pinned in `create-cluster.sh`
- Cluster Autoscaler: image tag pinned in a patch manifest
- KEDA, Kyverno, External Secrets Operator: all pinned in their install scripts
- Container images going to ECR: immutable tags enabled at the registry level — you can't overwrite a tag once it exists

The reason "no `latest`" is a hard rule when AI is writing your scripts is straightforward: AI is helpful and forward-looking. Left to its own devices, it will reach for the newest version because the newest version is, in the abstract, the best one. In practice, the newest version is the one that hasn't been tested against my cluster, my addons, and my K8s API version.

Pinning makes "what changed" answerable in one diff. When something breaks after an upgrade, the upgrade is in the git history, not in some implicit `helm install` that resolved a tag at runtime.

The version-bump workflow is the same as any code change: AI proposes a new pinned version (with a note on what release that maps to), I read the changelog, I review, I merge. The cluster-creation script even has a one-liner comment with the exact `aws eks describe-addon-versions` query needed to check for updates — the AI added it the first time I asked "how do I know when to bump this," and now it's there forever.

## DevSecOps Lives in This Repo

There is no separate "Security Team." Security work lives inside the Infra repo and the same script-pair discipline applies.

A non-exhaustive list of what that means in practice:

- **GitHub Actions auth via OIDC.** Every CI workflow assumes an AWS role via OIDC. Long-lived AWS access keys do not exist for any GitHub repo. Setup is two scripts: `setup-github-actions.sh` (creates trust policy and IAM role) and `delete-github-actions.sh` (tears it all down).
- **Per-service IAM policy isolation.** Each backend service that needs AWS access gets its own narrowly-scoped IAM policy (e.g. only the secrets *that service* uses). The script that applies the policy lives next to the script that builds the resource.
- **Per-service database isolation.** The same applies at the data layer. Services that need their own database get their own RDS instance. Cross-service blast radius is bounded by the fact that nobody shares a database. The pattern: a separate folder per service under the RDS scripts directory, each with its own `install-rds.sh` and matching `delete-rds.sh`.
- **Pre-commit hook: Gitleaks.** Every commit runs through Gitleaks before it can land. If the diff contains anything credential-shaped — AWS keys, tokens, suspiciously long random strings — the commit is blocked. AI commits get held to the same bar as human commits.
- **AWS Config on-demand, not always-on.** AWS Config is great. It's also expensive when it runs continuously. Instead, we run it on demand — start the recorder, wait 30–60 minutes for evaluations, read the Security Hub findings, stop the recorder. Quarterly audits and pre-release reviews. The cost goes from "always on" to "$0 between audits," and we still get the findings.
- **Kyverno as cluster admission control.** Pod security standards aren't a memo — they're enforced at the API server. If a manifest tries to run a privileged container, the cluster says no. AI can't ship a workload that violates policy because the cluster won't accept it.

None of this is exotic. All of it adds up to a security posture that doesn't depend on me remembering to do something.

## The Day a Delete Script Forgot Something

A while back, AI drafted an `install` script that did the right thing — provisioned the resource, attached the right IAM policy, set up the right pod identity binding. The matching `delete` script torn down the resource and the binding. It did not delete the custom IAM policy.

If I had run that delete script and walked away, I'd have left an orphan policy in the account. Not a security problem. Not a money problem. But a *correctness* problem — "$0 after teardown" was the contract, and the contract was being broken silently.

I caught it because I read the delete script. I always do. The fix took two minutes; the catch took thirty seconds.

This is the unglamorous reality of the script-first pattern. Most reviews are boring. Most catches are small. The point isn't that AI gets things drastically wrong all the time — it's that **AI is fast and confident**, and speed times confidence times write-access is how you end up with a Sunday spent fixing Saturday's mistake. The script in the middle is the brake.

I also catch the opposite: scripts that are *too* defensive, full of `|| true` and silent failures that paper over real problems. AI is trained to make scripts run; I'm trained to make scripts work. We split that work.

## What This Team Can't Do

Honest limits:

- **Multi-environment promotion.** Test → Prod is still a manual diff-and-apply for me. There's no automated promotion pipeline. I want one eventually, but the trade-off (more automation, more surface to review) hasn't tipped yet.
- **Cost forecasting.** Kiro is great at "what *did* I spend." It is not a forecasting tool. AWS Budgets covers the floor (alarm if monthly bill exceeds X), but real per-feature cost attribution is something I do by hand when I care.
- **Deep CloudWatch sleuthing.** Both tools can read CloudWatch. Neither is great at the kind of multi-step "trace this latency back to a slow downstream call" investigation that a senior SRE would do. That's a problem for the Operations team, and I have ideas.
- **Anything novel without a paved path.** When I add a new AWS service for the first time (recently: SES, WAF), the first install/delete pair takes longer because there's no template. Once the pattern exists, the next one is fast. AI is excellent at "follow this pattern with these parameters," less excellent at "invent the pattern."

## Lessons That Travel

If you're applying this pattern to your own setup — not necessarily as a solo founder — five things that survive transplant:

1. **Make the AI write reversible work.** Whether it's IaC, database migrations, or anything else. Pair every create with a delete and you'll get cleaner reasoning out of the model, not just safer artifacts.
2. **Split read tools from write tools.** Even if both are AI. The agent that *looks* and the agent that *changes* should be different processes, with different permissions.
3. **Pick the simplest IaC that fits.** Complexity is not free. The cost is paid in review attention, which is the most finite resource a small team has.
4. **Ban `latest`.** The convenience is real and the cost is invisible right up until it isn't.
5. **Read the diff. Always.** The script-first pattern only works because the human actually reads the script. The day you start scrolling past the diff is the day the brake disappears.

## Next

[2.2 Building Intercept: The Engineering Team — Specialists, All the Way Down](/2026/05/02/building-intercept-engineering-team/) covers the product itself: how 22 specialized agents — one per microservice, plus cross-cutting specialists for data, QA, pen-testing, and more — got built one at a time as the codebase grew, and how branch protection, strict delegation, and hooks-as-a-constitution keep the blast radius of any single change small.

---

*Part of [Building Intercept](/2026/05/02/building-intercept-introduction/) — a series on building an application security platform with three AI teams.*
