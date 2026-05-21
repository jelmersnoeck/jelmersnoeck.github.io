---
title: "Agents Are Not One Thing"
date: 2026-05-21
---

Agents are not one thing. They come in at least three deployment shapes, and the trust boundary scales with where they sit. Personal, team, autonomous. Each one demands a different identity story, a different audit story, a different credentials story.

The industry talks about "agents" as if they're a flat category. They're not. And the cost of pretending they are shows up the first time a research agent gets a personal GitHub token and decides to be helpful.

I run several agents in my daily work, and they fall into three archetypes: personal, team, autonomous. They share an LLM call at the core and almost nothing else.

## Personal: the agent that knows your inbox

The personal agent is tailored to the end user. It reads their calendar, their email, their tickets, their PRs, and it builds context the way a good chief of staff would, except it does so before the user's first coffee.

Mine is called `catchup`. It connects to Google Calendar and Gmail, Slack, Granola, Linear, and GitHub. Every morning it produces a brief: what happened overnight, what needs your attention today, what's coming up. The brief matters more on the days I'm not at full attention. I'm on parental leave right now and working part time, which means a Monday morning involves catching up on three or four days of email and chat across a team that didn't stop, and I do not have the time or the focus to grep through that by hand. The meetings get per-event prep, so every conversation, email, PR, and ticket related to the meeting is surfaced before I walk in.

That is the easy end of the trust spectrum. There is one principal: me. There is one user whose data the agent touches: me. The blast radius of a mistake is bounded by my own surfaces.

It's still the case that personal agents read deeply private data: your inbox, your DMs, your calendar invites. The temptation is to give the agent your personal API keys and move on. People do it. Disclosure: I work at Keycard, so I built `catchup` on it from day one. That isn't because I'm disciplined, it's because I knew where this was going.

The real cost of personal API keys is that they make the agent single-tenant by construction. The key belongs to one person, the agent serves one person, and that's the entire universe the design can imagine. It works fine until the day someone asks whether they can have it too.

The answer at that point is usually "sure, give me a week." Because going from single-tenant to multi-tenant is not a configuration change, it's a rewrite. The credentials were never scoped to *a* user, they were scoped to *the* user. There's no user dimension to peel out, no audit trail of who-saw-what, no way to revoke one person's access without revoking everyone's. The fact that the code accidentally works for one user is not the same as the code being ready for two.

If the agent was built multi-tenant from day one, with credentials minted per-user against a real identity system, adding the second user is a configuration change. Adding the tenth is the same operation as adding the second. This is why the framework has to bite at the personal end. Not because the personal blast radius is large, but because the personal stage is the only window in which the architecture is still cheap to choose.

## Team: the agent that does the work the team forgets

The team agent is shared and produces outputs the whole team consumes. It runs on a schedule, touches systems the team collectively owns, and acts on behalf of the team rather than on behalf of any one person.

Mine is an ops-review agent. Once a week it pulls Datadog data, categorizes the incidents and pages and errors that fired, and produces an overview of where the system spent its budget of attention. Without it, the weekly ops review is a chore: someone has to grep dashboards and assemble the picture by hand. With it, the review starts from a tight summary and the time goes to the interesting question. What do we change.

That single shift in deployment shape changes everything about the trust story.

There is no longer one principal. There is a team: a rotating set of humans whose only common identifier is the org. The blast radius is no longer "my own surfaces". It's whatever production telemetry the team can see, and by extension whatever production reality the team can act on.

Personal API keys do not work here. They never did, but at the personal end you could pretend. At the team end the pretending fails immediately:

- Whose Datadog key is it? When that person leaves, does the agent stop?
- When the agent reads a customer incident, whose audit trail records the read?
- If the agent's credentials leak, who is accountable for the rotation?

The team agent needs its own identity. Not a borrowed one. It has to be a first-class principal that the team grants access to in its own right, and from there it has to be able to act on behalf of individual team members when the task calls for it: read this person's tickets, post as that person, fetch the on-call's Datadog view. That's impersonation as a feature, not as a workaround. It only works if the agent has a stable identity *and* the platform has a way to delegate user authority to it on a per-action basis. Personal API keys provide neither half of that.

## Autonomous: the agent with a leash

The autonomous agent runs without a human in the seat for each action. It decides what to do next. It executes. It loops. The human is there to set the goal and to catch the agent if it heads somewhere it shouldn't.

Mine is called `keycard-scout`. It runs continuously, watches the agent-identity ecosystem for new developments (new standards, competitor moves, integration surfaces, partnership announcements), and where it spots a gap it builds a proof-of-concept to test the seam. It doesn't ask permission to start. It picks a candidate, scaffolds the POC, wires up the integration, and runs it. The 24-hour heartbeat it runs on today is a stand-in for the event-driven version it should eventually be.

Where it asks for help is at the points where it genuinely cannot continue. A new SaaS account it cannot self-register. A credential it has no scope to mint. A judgment call about whether a finding is worth pursuing. When it hits one of those, it pauses, posts to Discord, and waits. I steer it, it continues.

That is what HITL (human in the loop) actually buys an autonomous agent. Not approval before every action. A recovery channel for when the agent has done all it can and reached a wall. The leash is not there to choke the agent. It's there for the moments where the agent stops and looks back.

The trust shape at this end is unlike the previous two:

- The principal is the agent itself, acting on a delegated mandate.
- The mandate has to be precise. "Research the ecosystem and build POCs" is not a credential, it's a charter that has to be cashed out into scoped, time-bound permissions per task.
- Every action needs to be auditable not for accountability theatre, but because the human supervising the leash needs to know what the agent actually did when it comes back with a finding.
- Credentials need to be derivable per-task, not handed over as a bundle. The pentest sub-task gets one shape of credential. The Reddit-scrape sub-task gets another. The "post to Discord" sub-task gets a third.

This is the end of the trust spectrum where "give it a personal API key" becomes obviously catastrophic. The autonomous agent is going to act in your name dozens of times an hour. If those actions are indistinguishable from your own (same key, same audit signature, same surface), the audit trail has no useful information in it. Worse, you have no way to revoke just the agent's authority without revoking your own.

## What the three have in common

All three of my agents are wired through Keycard, and that is not a coincidence: I expected each of them to need it. None of them use personal API keys, none of them hold long-lived bearer tokens, and each agent is a named principal whose every action is attributable and short-lived and scoped to the task at hand.

The trust boundary doesn't start at autonomous. It starts the moment the agent needs to make an authorized call on behalf of you or on behalf of itself, and the real question from that moment on is how you govern it. Who is the principal making the call? Whose authority is it acting under? What scope was granted, by whom, for how long, and how is that decision audited after the fact? Agents are not one thing, and the cost of pretending otherwise is small at the personal end and ruinous at the autonomous end. The only stage where the governance shape is still cheap to choose is the first one. Get it right at the cheap end, or pay for it at the expensive end.
