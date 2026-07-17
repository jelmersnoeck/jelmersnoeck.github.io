---
title: "Specs Are the Deliverable"
date: 2026-07-16
---

The bottleneck in AI-assisted engineering is not code review. It is spec review, and before that, spec definition. Not many people are doing this yet, or are not doing it well. A good spec is what turns an agent's output from a gamble into a coding outcome you can sign your name to. We have not noticed the bottleneck because the spec is usually invisible, sitting in someone's head or in a Slack thread or in a comment on a ticket, and the code is the only artifact that gets a formal sign-off. That arrangement worked when a senior engineer wrote a few hundred lines a day. It does not work when an agent writes thousands.

## Code generation got cheap, review didn't

Two pieces converge on the same observation. Facundo Olano in [`--dangerously-skip-reading-code`](https://olano.dev/blog/dangerously-skip/) argues that if your organisation is going to push you toward maximum-speed agentic coding, then rigor has to land somewhere new, and his bet is a Markdown specification checked into the repo with PR checks that verify the code conforms to it. [Simon Willison](https://simonwillison.net/2026/May/6/vibe-coding-and-agentic-engineering/) frames it as a personal drift: he is reading less code, treating his agent more like another team he depends on, and feeling uneasy about it. Same phenomenon from opposite ends. Code generation got cheap while code review stayed expensive, and the two used to be matched. They are not matched anymore.

Once you accept that the trade is real, the next question is what the spec actually is.

## "Spec is just code" is almost right

[Birgitta Böckeler](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html) has a useful frame. Three levels of how a spec can relate to the code it produces:

- **Spec-first**: you write the spec, generate code from it, and after that the code is the source of truth. The spec is a starting point that drifts.
- **Spec-anchored**: the spec and the code coexist. Both are maintained, both are reviewable, and the relationship between them is the unit of work.
- **Spec-as-source**: the spec is the only thing a human edits. The code is a regenerable output of the spec, the way a binary is a regenerable output of source.

Spec-as-source is the destination. Not because it is clever, but because it is the only position that takes the bottleneck seriously. When generation is cheap and human attention is not, you put the attention on the artifact that decides what gets built. The code becomes downstream of that decision, not the place where it is made.

[Forge](https://github.com/jelmersnoeck/forge), the coding agent I have been building, aims for spec-as-source and gets close in practice. I do not write code in Forge. I edit and review specs. If I want a new feature, fix a bug, change behavior, I get the spec updated, and the code flows out of it. Forge's loop automatically checks the code against the spec before anything reaches me.

The remaining human pass on the diff is habit, not requirement. Once the conformance check is enforced automatically instead of run once and trusted, it drops.

The objection that "a very very detailed spec is just the code" is where the argument usually stops, and it stops one step too early. The goal is not a spec so detailed it becomes a programming language with Markdown syntax. The goal is a spec detailed enough about the *what* — desired outcomes, constraints, failure modes, invariants — that the agent can be trusted to figure out the *how*. If I am writing "if this then that" in prose, I have taken a wrong turn. That is what programming languages are for, and the agent is better at that layer than I am.

Every previous wave of spec-driven anything tried to land here and failed. UML, BPML, BRMS, Drools, all of those attempts collapsed for the same reason: the spec language was wrong. They tried to be executable in a syntax humans hated. They demanded the precision of a programming language without the tooling, the type checker, or the community to support it — which is exactly the mistake of treating a spec as "code, but slower." The thing that changed is the compiler. An LLM is a worse compiler than gcc on every dimension except one: it can take an outcome-level description in a language humans already speak and produce code that implements it. That is what makes spec-as-source viable for the first time, and [Tessl](https://tessl.io) is the most ambitious live attempt at it today.

## What changes when specs are the deliverable

If you take this seriously, three things change about how you work, and they get sharper the closer you get to spec-as-source.

First, the unit of review shifts from code diff to spec diff. A code diff is concrete, low-level, and easy to disagree about for the wrong reasons; people will argue about naming and indentation in a 200-line PR while the architectural decision that produced it goes unexamined. A spec diff is the inverse. It is high-level, abstract, and forces the disagreement onto the decision rather than the implementation of the decision. The first time you review a spec change instead of a code change, it feels strange, because the conversation you are now having is the one you used to skip.

Second, the granularity of accountability changes. The spec is the thing the team owns. Not in the trivial sense of "we wrote it," but in the sense that if the spec is wrong, the team is wrong, regardless of how good the code generated from it is. You cannot hide behind "the implementation has a bug" anymore, because the implementation is a function of the spec and a non-deterministic generator, and the only stable artifact across regenerations is the spec itself. This is uncomfortable. It should be.

Third, code review collapses into conformance checking. Most teams argue about code for hours and ship the architecture decision in a five-minute Slack thread. With specs as the deliverable, the architecture decision is the spec, and the code review is a check that the generated code does what the spec says. Olano calls it PR-as-conformance-check and treats it as the destination. It is the transitional position. At the far end, the conformance check is automated and the human review surface is the spec alone. We are not there yet, but that is where the line goes.

## What stands between here and spec-as-source

Non-determinism is real. The same spec, fed to the same agent twice, can produce different code, and will keep doing so until models get better at reproducibility. The spec-as-source answer is not to wait it out. It is end-to-end tests you can actually trust — the kind that exercise the product the way a user would, and that an agent can run against its own output to check the behavior matches the spec. Plus independent review by other agents, unbiased, not the one that wrote the code. That is the software-factory shape, and it is a topic for another post. Without both of those, you cannot stop reading code, and reading code is exactly what has to stop.

Granularity is unresolved. There is no obvious right answer to "one spec per file" versus "one spec per system" versus "one spec per feature," and the answer probably depends on the codebase. Forge currently lands on one spec per system area, which has worked for me, but I have not stress-tested it on a brownfield codebase with twenty years of accumulated decisions.

Brownfield is the hardest case. Spec-as-source assumes you can either start fresh or regenerate at will. Real codebases have hand-tuned hot paths, embedded historical compromises, and patches that exist because of a specific incident in 2019 that nobody remembers but everyone respects. None of that lives naturally in a spec. The current answer is "do not regenerate those parts," which works but is unsatisfying as a general strategy. I do not have a good general answer for this one yet.

Regeneration scope is the other open piece. If the agent rewrites everything on every run, code regen diffs are noise and no human can review them. The tooling has to keep regeneration narrow — touch what the spec change asked for, leave the rest alone. Get that right and diff review becomes tractable again. Get it wrong and you are back to code review as the bottleneck, only worse, because the diffs are now the agent's stylistic drift on top of the actual change.
