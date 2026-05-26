---
title: "Autonomy Is a Harness Property"
date: 2026-05-26T07:00:00-07:00
---

Autonomy is not a model property. It is a harness property.

The frontier-model conversation has spent two years assuming the opposite: that smarter models will produce better code, that bigger context windows will mean fewer mistakes, and that improving tool use will, on its own, let an agent run unattended. The implicit promise is that at some point the model becomes good enough that the scaffolding around it stops mattering.

It does not work that way in the systems we are shipping today, and it will not work that way in the ones we ship next year. The path from model to autonomy runs through predictability, and predictability is a property of the loop the model runs in, not a property of the weights behind it. Swap the model under a well-constrained harness and the output stays roughly the same shape; swap the harness under the same model and everything changes.

This is what I have learned from running [Forge](https://github.com/jelmersnoeck/forge) against my own repositories over the last few months.

## Two pillars

Forge constrains the model along two pillars, and both of them matter independently.

The first is the **spec**. Every change starts with an artifact that says what is being built and what is explicitly out of scope, and that artifact stays load-bearing for the entire run. Code gets generated against the spec, the generated code gets reviewed against the spec, and the spec itself is a living contract: when we find a bug, we clarify the spec; when something needs to change, we update the spec; then the next round of code follows. The spec is not a planning document that gets thrown away once code starts arriving.

The second is the **loop**. The phases are explicit and ordered (intent, spec, coder, review, finalize), and they are not skills the agent decides to invoke or capabilities the model gets to compose at runtime. They are the structure the run executes, in sequence, every time.

Specs make outcomes predictable. The loop makes flow predictable. The consequence is what makes the whole arrangement worth the effort: less human in the loop, not because the model got smarter, but because the system around it got more predictable.

## The figure-it-out approach

The first version of how I used coding agents was the obvious one: give the model the repository, hand it some skills, give it tool access, and let it work out what to do. If it missed an instruction, that was fine, because it probably got the gist; smart enough, fast enough, ship it. This is still the default shape of most agentic frameworks today. The agent is given capabilities and asked to use them wisely, and the wisdom is supposed to come from the model.

I ran that way for a while and the results were mixed. Not catastrophic, not bad enough to make me stop, but inconsistent in a way that did not scale. Some runs were great, others quietly drifted, and the variance turned out to be the problem rather than any single output.

Three patterns showed up repeatedly.

**The coder hallucinated extra functionality.** The agent would read the request, read the codebase, and decide to be helpful in ways nobody had asked for. It would add a feature flag I had not specified, refactor a neighboring function it thought looked messy, or quietly slip retry logic into a path that should fail loudly. Without anything to check against, none of this looked wrong; it looked like a thorough job, and I only caught it when I noticed the diff was bigger than the change should have been.

**The loop missed steps.** When the agent decides which skill to invoke next, sometimes it remembers to run the security review and sometimes it does not, sometimes it pushes without watching the CI checks and sometimes it waits. The variance was not in the quality of any single output but in whether the work ran end-to-end at all.

**I kept having to steer.** Even on runs that went well, I was the one nudging the agent between steps: "now run the review," "now push," "now look at the checks." That is overhead I should not be in the loop for. Every time I had to redirect the agent toward the obvious next step, the agent was not really autonomous; it was a fast typist with a human dispatcher attached.

Both failures are invisible without structure. A spec gives review something concrete to check against, and a hard-coded loop makes sure the review and the check-watching happen at all, because they are phases of the run rather than choices the model has to make.

## What this looks like in practice

The clearest evidence is not a single incident but the same sequence running every day. A change goes in, the spec gets written, the coder writes the code, the review phase runs and finds a security issue, the loop fixes the issue, the change is pushed, the GitHub checks are watched, a check fails, the loop fixes the failure, the change is pushed again, and the checks pass.

I am not in any of those steps. None of them paged me, none of them asked.

In a skills-based agent, each of those steps is a separate skill that the agent has to decide to invoke: run the review skill, then the fix skill, then the push skill, then the watch-checks skill. At every step the agent has to choose that the next skill is the right one, and sometimes it does the right thing while sometimes it skips review because the change "looked small" or pushes without watching the checks because that was not in the original instruction.

In Forge, none of those are skills. They are phases of the loop, and they run because the loop runs them rather than because the model decided they were a good idea. The security review happens whether the model thinks the change needs it or not, and the check-watching happens because the loop does not finish until the checks have been observed.

This is what produces autonomy: not capability, but structure.

## What this trades off

Constraining the agent costs flexibility. A spec-first loop cannot handle a request that does not fit the loop's shape, and "just try something and see if it works" is not a Forge phase. Exploratory work, the kind where you do not yet know what you are building, lives outside this harness, and that is fine. Exploratory coding is what humans and chat-style agents are for; production coding, the kind that ships unattended and gets reviewed against a contract, is what the harness is for. The mistake is using a chat-style agent for production work and then being surprised when the output drifts.

The other tradeoff is upfront cost. Writing a spec, even a generated one, is slower than typing a prompt and watching code appear, and the first ten minutes of a Forge run feel like overhead. They are not overhead; they are the part that makes the next two hours run without me.

## The principle

The harness is the part of the system the human controls, the spec is what the human and the agent both refer to, and the loop is what runs whether the model "feels like it" or not. When people say an agent is reliable, what they usually mean is that its harness is reliable. The model is the engine; the harness is everything that decides what the engine is allowed to do and when.

Frontier models will keep getting better. They will not, on their own, get more predictable. Predictability is what we build around them.

The autonomy is in the loop. The trust is in the spec. The model is the part that does what it is told.
