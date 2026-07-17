---
title: "Infra Substrates Are Cattle, Not Pets"
date: 2026-07-16
---

Substrates are not artifacts. They are commodities.

The pets-vs-cattle argument won at the component layer years ago. Individual servers, individual containers, individual functions: nobody names them anymore, nobody nurses them back to health, nobody logs into them at 3am to run `sudo systemctl restart` on a specific box. We autoscale. We recycle. We replace. That fight is over.

The substrate, though. The substrate is still a pet.

The substrate is the whole thing sitting under your workloads: the cluster, the database, the message bus, the vector store, the identity system, the observability stack, the CI runners, the secrets manager. The full working environment. And in most companies I have worked at, that substrate has a birthday, a name in Slack, and a small team of people whose job title implies they will nurse it until the end of time. It accretes. It gains features nobody remembers requesting. It gains legacy nobody has time to remove. It boxes the company into decisions that were reasonable four years ago and are load on the roadmap now.

I have watched this pattern play out across every job I have had. The names change, the cloud provider changes, the flavour of Kubernetes changes; the shape does not. A substrate gets stood up. A platform team gets built around it. The substrate becomes irreplaceable, not because it is good, but because the cost of proving a replacement is safe has grown faster than the cost of tolerating what is there. The team's job stops being "provide infrastructure" and starts being "protect the current substrate from anything that might break it."

That is the pet pattern at the substrate level. It is what the 2012 argument was supposed to end. It didn't.

## The figure-it-out-later approach

The way I used to think about this was: eventually, we will have enough abstractions on top that the substrate will not matter. Managed services will absorb the operational load. IaC will let us describe the substrate declaratively. Ephemeral environments will let us prove things in parallel. Somewhere in the middle of all of that, the substrate will quietly become boring, and we will move on.

Half of that came true. Managed services took over most components. Terraform and Pulumi and Crossplane made substrates describable. Preview environments got cheap enough to run per pull request. The substrate got more replaceable at the component layer.

And the substrate is still a pet.

Three patterns kept showing up:

**The database stayed a pet.** Every component around it went cattle, but the primary datastore lived on. Migrations were quarterly events. Version upgrades required a war room. The database was where the substrate anchored itself into permanence, and every other choice bent around it.

**The platform team stayed the whole substrate's owner.** They inherited responsibility for anything that could not be trivially recreated, which quickly became "anything anyone cares about." The org chart baked the pet pattern into who could change what.

**Migrations kept being heroic.** Every substrate replacement I have participated in was a project. A named initiative. A quarter or two of work with dedicated people and a launch plan. That is not cattle behaviour. Cattle behaviour is a script run in a loop by a scheduler.

The through-line: the technology got more replaceable, and the operating model didn't. We had cattle-shaped components arranged into a pet-shaped substrate, held together by a team whose job was to keep the pet alive.

## What changed

Two things happened in the last eighteen months that I did not expect to line up the way they have.

The first is that state started disaggregating. [Neon](https://neon.com/docs/introduction/architecture-overview) runs Postgres with compute and storage as separate services; the source of truth is S3. [WarpStream](https://www.warpstream.com/) runs a Kafka-compatible streaming platform with no local disks, storing everything in object storage, which is why Confluent bought them. [turbopuffer](https://turbopuffer.com/) runs vector and full-text search on top of object storage with cache tiers layered above, and it is what Notion, Linear, and Cursor use. AWS shipped [S3 Vectors](https://mixpeek.com/guides/vector-database-on-s3-object-storage) as a first-class service. The pattern is not one company's bet, it is the direction the industry moved in a year.

What the disaggregated-storage wave means for the pets-vs-cattle argument is specific: the runtime becomes stateless, the state lives in object storage as the durable source of truth, and everything on top is derived. Which means the runtime is finally replaceable at the substrate level, not just the component level. Spin up a parallel Neon project against the same durable state; cut over; discard the old runtime. That was not a thing you could say five years ago about the Postgres primary, and it is a thing you can say now.

The second is that agents showed up. Not chatbots; agents that take actions against infrastructure. The frontier-model conversation has spent two years treating this as a capability question, as if the blocker was whether the model could write Terraform. It was never that. I wrote about the shape of the actual blocker in [From Desired State to Negotiated State](https://jlmr.dev/posts/from-desired-state-to-negotiated-state/): IaC as we ship it today assumes a human in the loop, and agent-owned infrastructure needs a different contract. This post is the substrate-level corollary of that one. If IaC has to change for agents, so does the thing IaC is describing.

Because here is the awkward truth. You cannot hand a pet to an agent. Not because the agent lacks the tokens or the tools; because pets are, by definition, things that require a human to interpret the accreted history. What did this label mean when it was added in 2022? Why does the staging cluster have this one node pool with a taint on it? Why is the primary Postgres running two versions behind? An agent handed a substrate like that will either freeze, or, worse, act with confidence that is not warranted. The substrate's history is not in the substrate. It is in the platform team's heads.

## The agent-fitness test

Here is the qualifying test I keep coming back to.

If you would not let an agent rebuild your substrate from state, tonight, unattended, then your substrate is a pet.

Not "unattended in production." Unattended anywhere. If the answer to "can I ask an agent to spin up a fresh copy of our substrate from source, run the smoke tests, and let me know when it is ready" is anything other than yes, the substrate has pet-shaped properties somewhere. There is history the agent cannot recover. There is drift the source-of-truth does not capture. There is a person who has to be paged for one specific step. Something is being nursed.

The test is uncomfortable in a useful way. It exposes the parts of the substrate that everyone knew were fragile but had learned to route around. Every "well, we don't touch that" is a pet. Every "you need to ask $person before changing that" is a pet. Every "the runbook says X but actually you have to Y" is a pet. Substrates that pass this test look boring: they can be described, replicated, verified, and thrown away.

Two things follow.

The first is that state has to be disaggregated. If your Postgres cannot be rebuilt from a durable log or a snapshot in object storage, no agent is going to safely rebuild it. Neon, WarpStream, turbopuffer, S3 Vectors: those are not just cost plays. They are the substrates that pass the fitness test, because the state and the runtime are separable, and the runtime is the disposable part.

The second is that the dedicated platform team, as an institution, has to go. Not because the people are not good at their jobs; they usually are. Because a team whose reason to exist is nursing a specific substrate is structurally incapable of shipping a substrate an agent can own. Their incentives run the wrong way. The substrate they build has to justify their headcount, and that pressure produces pets.

## What replaces it

If the dedicated platform team dissolves, something has to take its place, and I do not think the answer is "everyone runs their own everything." That gets you a different pet per product team.

The shape I keep landing on has three parts. A small team owns the substrate template: the declarative description of what a substrate is, how it composes, and how you validate one. Agents instantiate substrates from that template on demand: per environment, per team, per pull request, per whatever the unit of ownership turns out to be. Product teams own the boundary between their workload and the substrate, and treat the substrate underneath as something they can throw away.

The template team ships primitives, not substrates. They never nurse a specific instance. If a substrate breaks in a way the template did not anticipate, the answer is a fix to the template, not a hand-tune to the instance. Instances are cattle.

That model needs the disaggregated-storage layer to work at all, because without durable state that survives runtime replacement, you cannot instantiate freely. And it needs agents that can be trusted with the instantiation, because without them, you have moved the pet from the platform team's Kubernetes cluster to the platform team's Terraform state file. But given both, the substrate finally becomes what it was supposed to be a decade ago: a commodity underneath the work, not an asset on top of it.

## Coda

The substrate is cattle. The state lives elsewhere. The template is the thing you nurse.

Replace the substrate. Let the agent run the loop.
