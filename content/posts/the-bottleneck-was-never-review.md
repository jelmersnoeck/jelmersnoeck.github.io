---
title: "The Bottleneck Was Never Review"
date: 2026-07-24
---

The reviewer bottleneck is real, but also temporary.

Every serious conversation about agents in engineering organizations right now ends up at the same worry: models write code faster than humans can review it, the review queue grows, quality collapses, and the whole exercise turns into a slow-motion regression.

That worry is correct, today. In three years it will not be. Evals will improve, behavior will get more predictable as a [harness property rather than a model property](https://jlmr.dev/posts/autonomy-is-a-harness-property/), and attribution and audit will move into infrastructure instead of pull request screenshots and Slack threads. That is a bet, not a law of nature, but it is a bet the model vendors are funding themselves, because evals are what make their agents sellable. Scaling review is a problem people are already paid to solve.

The bottleneck we are not going to engineer our way out of is a different one: most companies cannot state what "correct" means in a form an agent can consume. That one is not going away on its own, and it changes what an engineering organization actually is.

We keep framing it as the engineer and their agent: the [copilot](https://github.com/features/copilot), the [Cursor](https://www.cursor.com) session, the IC and their pair. That is the frame that gets funded and demoed, and it is wrong. An engineer still drives the agent, and that is what keeps the frame alive. The driver is doing two jobs at once: checking that the output is correct, and translating the company into the prompt, supplying the context that was never written down. The first job is temporary; a good harness and good evals absorb it. The second one does not go away. It works for one engineer steering one session, and it stops working when the agents outnumber the drivers, which is the entire point of agents.

Agents are not personal productivity tools; they work inside the organization, and they need its context, not the engineer's. The engineer's context is in their head, in the DMs from last Tuesday, in the call with the security team where nothing got written down. An agent cannot read any of that, and neither can the next engineer, which is the same problem the company already had, just less prominent.

## Accountability and trust are two different problems

People use two words interchangeably in this conversation, and they are worth pulling apart, because they get solved at different speeds with different tools.

Accountability is being able to answer, after the fact, what an agent did, on whose authority, and against which policy. It is a paper trail that holds up, and it is an infrastructure problem: identity, scoped credentials, policy-as-code, and audit logs going somewhere queryable. This is roughly the problem [Keycard](https://keycard.ai), where I work, is trying to solve at the platform layer, and it is solvable: people are already building the pieces.

Trust is the belief, in advance, that the agent will do the correct thing well enough that we do not need to inspect every output. You earn it by watching an agent handle a class of work under policy for long enough that you know how it fails and what that costs. This is the problem the industry is currently obsessed with, mostly framed as "how do we make review scale", and better evals, better models, and structured verification will all help.

The trap is treating both of these as model problems. They are not. Both of them need the same thing first: a policy that the audit and the agent can both read, and a description of the domain that lives somewhere other than an engineer's head. Without that, "did the agent follow policy" and "can I trust the agent with this work" stall in the same place: there is nothing to check either one against.

Make it concrete: ask an agent to rotate a database credential, a task today's models can already do. To do it correctly, the agent needs to know which services use that credential, what the rotation window is, who signs off on production changes, and the exception: the one legacy service that caches connections and falls over unless it gets restarted in the same window. In most companies, every one of those answers lives in a person. The agent is capable of the work and has no way to be correct at it, because correct was never written down.

This is the part that gets skipped, because it is nobody's obvious lane. The infrastructure people assume the policy already exists and just needs to be enforced, while the evals people assume the domain is well specified and just needs to be tested against. In most companies, neither is true, and the thing the whole stack depends on is the thing nobody is being paid to build yet.

## The company has to be readable

What agents need from a company is easy to state and hard to have: its policies, its architectural decisions, its ownership boundaries, its exceptions and their reasons, written down in a form that something that was not in the room can read.

That is not the same as documentation. Documentation is what a company writes on purpose, and it decays the moment someone changes something without updating the doc. A readable company is harder than a documented one: the reasoning is captured, the exceptions are named, and the artifacts sit close enough to the work that they get updated as part of doing it, not as a separate chore. A wiki full of stale onboarding pages does not make a company readable. A codebase with clear ownership, a policy repo the CI reads, an ADR log people actually write, and a spec catalog the agents consult, does.

This is what makes accountability and trust possible in the first place. If policy is something a human can read and an agent can act on, you can audit against it and you can constrain the agent to it; if the domain is described in specs the agents work from, you can measure whether their output matches the specs and catch drift when it does not.

None of this is new. Companies that scaled ran into it before agents were part of the conversation: how do we onboard a hundredth engineer, how do we let a team in another timezone ship without a call, how do we survive a critical person taking a month off. The answer in each case was the same: push more of the operating knowledge out of people's heads and into artifacts. Agents just raise the tax on failing to do it, because they cannot ask around, read the room, or infer from a shared history they were not part of. They can only work from what is written down.

## The accidental optimizers

Remote-first companies solved this by accident, not because they were smarter about knowledge management but because they had no other option. With no office, "just ask" stops working; with no shared timezone, decisions have to be written down to survive the handoff, and whoever needs context has to find it in an artifact, because the person who has it is asleep. Over years this compounds into a habit: capture decisions in writing, keep the reasoning next to the decision, treat the async trail as the record of truth. That habit is exactly what agents need.

This is not an argument that remote-first companies are better companies; plenty of them still keep their knowledge in DMs and Zoom recordings nobody watches, and plenty of in-office companies built the same discipline through other means. The seating chart is not the point. The point is whether the company's knowledge lives outside the heads of the people who happen to hold it.

If your company runs on hallway conversations and tribal knowledge, agents will not save you time. What they will do is show you, at speed and at scale, exactly how much of your company's operating model was living in three people's heads. That is a useful finding, just not the one the demo promised.

## What this means for engineering leaders

The instinct is to treat this as a tooling problem: buy the platform, deploy the agents, run the pilot. That works for the individual-productivity framing, which is why every vendor demo is a solo engineer completing a ticket faster. It does not work for the organizational framing, because that one needs the company to already be readable enough for the agents to plug into. The tooling does not create that; it only shows you whether it exists.

The software factory pitch makes the same mistake at a bigger scale. A factory is not something you install, it is a process: how work comes in, how it gets checked, who signs off, what happens when something fails. The machines are the cheap part. Most companies do not have that process for software today, and agents will not create it for them. They will run whatever process exists, faster.

The real work sits upstream of the agents, and it is boring: own the policy repo the way you own the codebase, review ADRs like PRs, write specs that capture intent and constraints instead of just requirements, and make the on-call runbook the thing that actually runs the on-call instead of a document that lags reality by six months.

This looks like housekeeping. It is not. It decides which companies come out ahead, and they will not be the ones with the shiniest agent stack; they will be the ones where an agent can join a team, read the artifacts, and be productive without a human translating the company on the fly. The individual framing lets everyone off the hook. The organizational framing hands the responsibility to the leaders who decide what the company writes down and what it does not.

Agents do not change the software engineer's job because they replace the coder. They change it because, for the first time, everything the company never wrote down has a price, and the bill comes to engineering.

The bottleneck was never review. Review is a symptom. The bottleneck is the company being able to explain itself to something that was not in the room.
