---
title: Sandboxes Are Not Security
date: 2026-06-12
---

The industry has settled on an answer to the question of how to run AI agents safely: put them in a sandbox. [AWS](https://aws.amazon.com/blogs/machine-learning/introducing-the-amazon-bedrock-agentcore-code-interpreter/), [Daytona](https://www.daytona.io/), [Cloudflare](https://www.cloudflare.com/products/sandboxes/), [LangChain](https://www.langchain.com/blog/langsmith-sandboxes-generally-available), pick one: they all sell it. The category has a shape, a name, a TAM, and a comparison-table format. "Secure code execution for AI agents." "Zero risk to your infrastructure." "Real isolation, not just sandbox features."

It is not security. Not on its own.

A sandboxed agent with valid Postgres credentials can still drop the production table, send every customer an email through a working OAuth token, and spin up a hundred GPUs in eu-west-1 on an attached AWS role. The sandbox does not gate any of these outcomes, because the sandbox is not what stands between the agent and the resource. The credential is. And the credential is sitting inside the sandbox, ready to be used.

Sandboxes are part of the answer. They are not the whole answer, and the industry keeps selling them as if they were.

## What a sandbox actually does

A sandbox protects the host from the agent's own process. Filesystem isolation. Network namespacing. A bounded blast radius for whatever the agent writes to whatever directory the agent has. These are real properties and they solve real problems. State pollution between sessions. Reproducibility across runs. Containment of an agent that runs `rm -rf /` on what it thinks is its own scratch directory. None of that is fake.

It is also not what attackers are after.

An attacker who compromises an agent, whether through prompt injection, a poisoned tool description, or a malicious document the agent was asked to summarise, does not want the agent's scratch directory. They want what the agent is authorised to do: the Postgres connection string the agent uses to answer questions about customers, the GitHub token the agent uses to open pull requests, the AWS role the agent uses to read from S3. That last one probably also lets it write to S3, and write is where the damage lives. None of these are stored in the agent's filesystem in any meaningful sense. They are loaded into the agent's process at runtime, used by the agent on the attacker's behalf, and the sandbox watches the whole thing happen without interrupting once. The sandbox cannot interrupt. That is not its job. That has never been its job.

## How we got here

Code-execution sandboxes are a real and useful primitive, and the industry knew how to build them before agents existed. They came from Jupyter, from Repl.it, from the lineage of "run untrusted code from untrusted users in a multi-tenant environment without letting them escape onto our host." That threat model is coherent. The code is the attacker. The sandbox is the answer.

What happened next is that the agent platforms borrowed the primitive whole and kept the marketing. Agents look like code that needs to be run, so the sandbox vendors pivoted, and the new agent platforms shipped sandboxes as their first security feature. The pitch decks updated, the comparison tables updated, the buyer's mental model updated. "How do we secure our agent" got the same answer "how do we run untrusted code" got, and the answer was wrong, because the threat model changed under everyone's feet and nobody changed the slide.

Agents are not untrusted code. They are trusted code holding untrusted instructions and authorised credentials. The danger lives in what the agent is allowed to do on behalf of the human who deployed it, not in what the agent might run that escapes onto the host. Sandboxes were designed to stop the second thing. They were never designed to stop the first.

## Two controls, different jobs

Isolation gates what a process can touch on its own machine. Authorisation gates what a process can do on someone else's behalf. They are different security properties solving different threats. The industry collapsed them and called the result agent security.

A useful test. If your only safety claim about an agent is that it runs in a sandbox, ask what happens when the agent calls the API it was given a token for. If the answer is "the call goes through," the sandbox is irrelevant to the threat. Whether the sandbox is a microVM, a container, a WASM runtime, or a managed service with SOC 2 compliance: irrelevant. The token left the building the moment the agent decided to use it. The sandbox watched.

A sandbox can lock down egress and refuse to let the agent talk to anything except an allow-listed set of endpoints. That helps, until the endpoint is one the agent is allowed to reach and the compromise becomes a credential leak instead of an exfil. The right answer to that shape of problem is a gateway sitting between the agent and the resource, brokering the call and watching what the credential is used for; another post.

## The first move past the consensus

The first move has already been made in public, and not by a security vendor. [Mendral](https://www.mendral.com), an agent infra startup founded by ex-Docker engineers, has a post called [The Agent Harness Belongs Outside the Sandbox](https://www.mendral.com/blog/agent-harness-belongs-outside-sandbox), written by [Andrea Luzzardi](https://x.com/aluzzardi). The post does a lot of things at once (durability, suspension, multi-user state, security), and one paragraph in the middle is the move:

> Your credentials stay out of the sandbox. The loop holds the LLM API keys, the user tokens, the database access. The sandbox holds only the environment the agent needs to do its work. There's nothing in there for the agent to escape to, so there's no permission model to enforce and no credential leak to contain.

Take credentials out of the sandbox. Treat the loop as the trust boundary. Most of the industry hasn't gotten that far, and getting that far matters.

The vendors who built sandbox products are starting to admit the same thing, quietly. OpenAI's [Agents SDK post in April](https://openai.com/index/the-next-evolution-of-the-agents-sdk/) says, almost in the same breath as their sandbox marketing: *"Separating harness and compute helps keep credentials out of environments where model-generated code executes."* Their own [Codex cloud documentation](https://developers.openai.com/codex/agent-approvals-security) is more direct: *"Secrets configured for cloud environments are available only during setup and are removed before the agent phase starts."* The team building one of the most-shipped agent products in the world has decided that the sandbox is not the credential boundary.

This is real progress, and it is still halfway. Pulling credentials out of the sandbox shrinks the credential surface; it does not change its shape. The loop now holds every credential the agent might ever need, in one place, for the whole session. Compromise the loop through any of the things that compromise agents (prompt injection, a poisoned tool description, a malicious document the agent was asked to summarise) and the attacker walks away with all of it. The bag got smaller. It is still a bag.

The next move is the harder one. A credential needs to be bound to the actor *and* the action, not one or the other. Bound to which agent is calling, on behalf of which principal, doing what, this turn. Today's auth carries one half of that pair: the actor. The action gets scoped to whatever standing permission was attached when the credential was issued, which is usually a verb broad enough that the same credential works for "summarise this thread" and "forward every message in this thread to an external address." The specific protocol doesn't matter; the gap is structural. [Capabilities can't tell those apart either](https://jlmr.dev/posts/capabilities-cant-see-your-agents-objective/), for the same reason: a scope is standing permission, not per-call intent.

The identity community has started circling this. [AAuth](https://www.aauth.dev) gives every agent a cryptographic identity, carries identity claims and authorization claims in the same token, and evaluates per call rather than per session. It is not the only attempt, and the shape of the answer is not settled. The point is that the conversation is finally happening one layer up from the sandbox, where it should have started.

The shape of the answer is identity. The industry doesn't have it yet, and a sandbox isn't going to get them there.

## What sandboxes are actually for

Sandboxes are useful. State hygiene is real. Reproducibility is real. The agent that overwrites its own context window, or pollutes its scratch directory, or leaves a half-finished file from a previous run: these are real problems, and a sandbox is the right tool for them. Sandbox vendors should keep selling that, because that is what they built and it works.

What they should stop selling is the idea that this is the security story. The sandbox is a hygiene primitive. It is one component of a security story whose other parts haven't been built yet. Calling the part the whole has held the industry up for two years and it is starting to cost real companies real money.

The sandbox is not security. The sandbox is part of security, when the rest of the story shows up. Anything else is hygiene with a security pricetag.
