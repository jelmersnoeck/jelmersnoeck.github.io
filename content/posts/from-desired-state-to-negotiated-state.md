---
title: "From Desired State to Negotiated State"
date: 2026-01-17
---

GitOps, Terraform, ArgoCD—they all share the same promise: declare your desired state in code, and the tooling makes it real. Your code is the source of truth.

Except it is not. Not really.

Your HPA scaled replicas at 2am because traffic spiked. Your security scanner patched a vulnerable image. Your cost optimizer right-sized an instance. Your on-call engineer hotfixed a config during an incident.

None of them opened a PR. None of them asked your Terraform code for permission.

We have managed this tension for years. Drift detection, `terraform import`, cultural discipline ("do not touch the console"). It worked well enough when the actors were mostly human and changes happened at human speed.

That is changing. Here is what breaks when it does.

## The Mental Model We All Share

Whether you are using Terraform, Pulumi, CloudFormation, ArgoCD, or Flux, the underlying model is the same:

- **Code is authoritative.** If there is disagreement between your config and what exists in the cloud, the config is right.
- **State converges to code.** The reconciler's job is to make reality match the declaration.
- **Drift is aberrant.** Out-of-band changes are mistakes to be corrected.
- **Changes flow one direction.** Code to cloud. The PR is the entry point.

This model is elegant. Desired state vs actual state. Idempotent operations. Version control, code review, audit trails. Reproducibility.

But it assumes a single writer—or at least a small number of coordinated writers who all go through the same PR process.

## The Cracks Were Always There

Drift has always existed. We just did not talk about it much.

Console edits during incidents. Autoscalers doing their job. Operators mutating resources—cert-manager rotating certs, external-dns updating records. Cloud provider defaults that silently change. Security patches applied through a different pipeline.

That `terraform import` you ran last month? That was a quiet admission that state escaped your code.

We built workarounds. State locking. Continuous reconciliation. Policy enforcement at admission time. Cultural discipline: "do not touch the console."

These workarounds held because the conditions allowed it. Actors were few. Velocity was human-paced. When things got tangled, someone could untangle them manually.

Here is the uncomfortable truth: the cloud provider's API was always the real source of truth. Your IaC is a projection of intent. When `terraform plan` shows changes, it does not mean your code is right. It means reality diverged.

Kubernetes acknowledged this with [Server-Side Apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/). It tracks which actor last touched which field through `managedFields`. When two actors try to modify the same field, you get a conflict.

That is an admission that multi-actor is real, not aberrant.

## What Is Changing

It is not just autoscalers anymore.

Security scanners automatically patch vulnerable images. Cost optimization platforms right-size resources. Compliance tools enforce encryption and tagging. Self-healing systems remediate issues without human intervention.

And increasingly, AI-assisted workflows—autonomous remediation, LLM-driven ops—add more actors to the mix.

Each of these has opinions about what the infrastructure should look like. Each makes changes. Nobody asked the YAML file how it feels about this.

The coordination problem scales non-linearly:

- **2 actors:** Manageable with conventions. "HPA owns replicas, you own everything else."
- **5 actors:** Need explicit ownership boundaries and documentation.
- **20 actors:** Humans cannot track the interactions. Need automated coordination.
- **N actors at machine speed:** The model breaks.

Think about what breaks. PR review assumes human-speed changes. "Do not touch the console" assumes humans are the problem. Continuous reconciliation assumes Git should always win. None of these assumptions hold when authorized systems make concurrent changes at machine speed.

This forces us to confront two distinct problems we have been conflating: reconciliation and competing motivations.

## Problem 1: Reconciliation

The first problem is mechanical: how do we detect drift and sync changes back to IaC?

This is about observation and translation. Where does authoritative state live? (The cloud API.) How do we observe all changes, regardless of source? How do we update our code to reflect reality without breaking its structure?

There is real progress here.

A recent paper called [NSync](https://arxiv.org/abs/2510.20211) takes an interesting approach. The key insight: all infrastructure changes—console, CLI, SDK, Terraform—become cloud API calls. AWS CloudTrail, Azure Activity Logs, GCP Audit Logs see everything.

NSync uses these audit logs to detect drift, then uses LLMs to infer high-level actions from noisy API traces. A flurry of API calls becomes "someone added an S3 bucket with versioning enabled." It then synthesizes IaC patches that preserve the existing code's structure.

Other tools work this space too. [Driftctl](https://github.com/snyk/driftctl) detects drift between Terraform state and cloud reality. [Terraform Cloud has drift detection](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/health#drift-detection). GitOps tools reconcile continuously.

The mechanical problem—"what changed?"—is being solved.

But reconciliation answers "what changed." It does not answer "should we accept it?"

## Problem 2: Competing Motivations

The second problem is harder: when multiple actors make conflicting changes, whose change wins?

This is not resource contention (who gets the CPU) or task allocation (who does which job). We have mature solutions for those. This is about competing claims over what the desired state *should be*. It is semantic, not mechanical. And it breaks down into three sub-problems.

**Sub-problem A: Capturing the intent**

Intent is the goal behind a change. Not the specific value, but what it serves.

HPA sets replicas=10. The intent is availability. Cost optimizer sets replicas=3. The intent is cost efficiency. Security scanner patches an image. The intent is vulnerability remediation.

Intents are relatively stable categories: availability, cost, security, compliance, performance. Most infrastructure changes map to one of these.

But intent alone is not enough. Two actors can share the same intent and still conflict. Two different availability concerns. Two different compliance requirements. You need more context.

**Sub-problem B: Capturing the motivation**

Motivation is the *why now*. The context that explains urgency and priority.

The on-call engineer wants replicas=10. Intent: availability. Motivation: incident response, primary is throwing 500s, customers are affected.

The platform team also wants replicas=10. Intent: availability. Motivation: scheduled capacity for tomorrow's launch.

Same intent. Same desired state. Very different motivations. If you had to roll one back, you would pick the scheduled change over the active incident response.

Today, we capture almost none of this. Audit logs show *what* changed and *who* (which service account). Git captures prose for humans. Nothing is machine-readable. Nothing connects the change to its intent or motivation.

**Sub-problem C: Deciding who wins**

Even if we capture intent and motivation, we still need logic to arbitrate.

- **HPA vs cost optimizer:** Both valid. Intent: availability vs cost. But if HPA's motivation is "active traffic spike" and cost optimizer's motivation is "monthly budget target," the spike wins.
- **Security patch vs app stability:** Intent: security vs availability. Motivation: CVE with known exploit vs potential compatibility risk. Probably security wins—but what if the app is revenue-critical during a peak sales period?
- **Hotfix vs declared state:** On-call engineer's intent: availability. Motivation: incident response. Terraform's intent: consistency. Motivation: it is 9am and this is what the cron job does. The hotfix should win—but nothing in our tooling knows that.

Look at what we have today:

- **Kubernetes SSA:** Tracks *who* touched each field, not *why*. Conflict resolution is "force" or "fail."
- **OPA/Gatekeeper:** Binary allow/deny. Cannot arbitrate between two valid intents.
- **NSync:** Assumes out-of-band changes should be accepted. Does not question validity.
- **Terraform/Pulumi:** Your code is always correct. `apply` overwrites.
- **GitOps:** Git always wins.

None of these capture intent. None of them capture motivation. None of them have arbitration logic that weighs context.

We can record who changed what. We can infer what the change was. We cannot capture why it was made—the goal it serves or the context that explains it—and we cannot decide whose motivation should win.

## The Gap

The one domain that has thought carefully about "intent" is [Intent-Based Networking](https://datatracker.ietf.org/doc/rfc9315/).

RFC 9315 defines intent as "operational goals and outcomes defined in a declarative manner without specifying how to achieve them." Users declare what they want (high availability, low latency), the system figures out how to achieve it (specific routes, configurations).

But notice what that solves: the *what-vs-how* problem. This is about abstraction level.

It does not solve the *why-should-this-win* problem. When two intents conflict, RFC 9315 says the system should "help users to properly choose between intent alternatives."

In other words: escalate to a human. The arbitration logic itself is out of scope.

And even IBN's conflict model assumes intents come from *humans* who can be consulted. It does not address autonomous actors making changes at machine speed with no human in the loop.

Infrastructure has even less. Our tools do not capture intent or motivation. They do not have arbitration logic. They assume single-writer.

When that assumption breaks, we have no fallback—just "last writer wins" or "code always wins."

## Where This Leaves Us

The point here is not to propose solutions. It is to name the problem clearly.

We have been conflating two distinct challenges:

1. **Reconciliation**—the mechanical problem of detecting and syncing drift. This is being solved. Real progress.
2. **Competing motivations**—the semantic problem of capturing *intent* (what goal a change serves), *motivation* (why now, what context), and then deciding whose motivation wins when they conflict. This is largely unsolved for infrastructure.

The tools we have—Terraform, Pulumi, ArgoCD, SSA—were built for a world of human operators making deliberate changes through controlled processes. They do not capture intent. They do not capture motivation. They do not arbitrate.

Agents are going to force the issue. When you have autonomous systems making changes at machine speed, "just overwrite" and "Git always wins" stop being viable strategies.

The source of truth is not a file. It is a negotiation.
