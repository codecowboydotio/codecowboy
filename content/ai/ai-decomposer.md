+++
title = "AI shared goal decomposer"
date = "2026-07-12"
aliases = ["ai"]
tags = ["ai", "dev"]
categories = ["ai", "software", "dev"]
author = "codecowboy.io"
showtoc = true
+++

# Distributed shared goal decomposition

A goal goes in — **"plan a two-week trip to Japan"** — and a tree of executable prompts comes out, produced by four kinds of agent that never share a process, a database, or a coordinator. **No peer ever holds the whole plan.** Plan state is reconstructed locally by each peer.

---

## Intro

I've been messing around with AI agents a lot recently, and two things I've built have been sitting in the back of my mind together for a while.

The first is a generate → score → refine loop — get one agent to write something, get a second to score it against a rubric, and use that score to refine it, all in a loop, all automatic. I've written about that before. It works well, but it's got a single process sitting in the middle of it the whole time, deciding what "done" looks like.

The second is the peer-to-peer agent stack I built on libp2p — agents that discover each other over multicast, gossip on a pub/sub bus, and don't need a central process to coordinate who does what. I've also written about that one.

I kept thinking: what if I combine those two together? Not on code this time, on the actual planning step. Take a big vague goal, like "plan a two-week trip to Japan," and instead of one process deciding how to break it down, let a swarm of agents fight it out. One kind of agent proposes a split. Another kind scores the competing splits and decides which one wins. A third kind picks up the pieces once they're small enough to actually do, and does them.

The bit that actually interested me wasn't the AI part — decomposing a goal into subgoals is a solved enough problem with an LLM. It was the "no one's in charge" part. If a decomposer proposes a split and then its process dies, does the plan just stop? If two decomposers propose different splits for the same goal at the same time, who wins, and how do the other agents even agree on that without asking a boss? If a new agent joins halfway through, can it figure out what's going on just from gossip, with nobody handing it a summary?

So the things I wanted to explore were:

- Whether a goal → subgoals → executable prompts pipeline could work with no single process ever owning the whole plan
- Getting multiple decomposer agents to propose competing splits for the same goal, and letting scorer agents fight it out instead of one scorer deciding alone
- A claim/lease protocol so multiple executors racing for the same piece of work don't both do that work
- Whether a peer that joins late, or is just watching, can reconstruct the whole plan from nothing but what it's overheard

TLDR — if you just want to see it running, skip to [Running it](#running-it) below. The code lives in `decentralized_decomposer/`.

## What is this?

At a high level: you submit a goal as plain text. A decomposer agent asks an LLM to split it into a handful of subgoals. A scorer agent (or several) rates that split against a rubric, and once enough scorers agree it's good, the split is accepted. Every subgoal that isn't yet a single concrete, do-able thing goes through the same propose/score/accept cycle again, one level deeper, until everything left is atomic. Executor agents race to claim those atomic pieces and actually run them through an LLM, and publish the result.

Four kinds of agent, each its own OS process:

- **Decomposer** — turns a goal into a proposed split
- **Scorer** — rates splits, decides when one's accepted, and decides whether each resulting subgoal is atomic or needs splitting again
- **Executor** — claims and runs the atomic leaf prompts
- **Observer** — does nothing but listen, and prints the plan tree it's reconstructed so far. No LLM calls, no capabilities, just there so I can actually see what's going on

None of these talk to each other directly. Everything goes over gossip.

## Peer to peer topology

Same as the multi-agent stack I built before, I'm using `libp2p` for the networking, specifically its GossipSub pubsub implementation for the message bus and mDNS for discovery. Agents find each other on localhost/LAN automatically — no bootstrap node, no DHT, nothing to configure. That's deliberate: the target here is three to eight processes on one laptop, not a real distributed deployment, so plain mDNS is more than enough.

Each agent also needs an identity that survives a restart. On first run it generates an Ed25519 keypair and writes it to `.identity/<role>-<port>.key`. Without that, restarting a process hands it a brand-new PeerId and it loses its place in every other peer's peerstore — as far as the network's concerned, it's just a stranger showing up.

![Gossip mesh topology](/images/gossip-mesh.svg)

> **Info**
> `py-libp2p`'s own `new_host(enable_mDNS=True)` wires its built-in mDNS discovery to a *hardcoded* port 8000, no matter what port the host is actually listening on. Took me a bit to work out why agents on any other port just weren't finding each other. Ended up constructing `MDNSDiscovery` myself with the real listen port instead of relying on the built-in wiring.

> **Info**
> `py-libp2p`'s own pubsub demo starts a background task that garbage-collects expired peer records. Turns out its sweep deletes a peer's *entire* record — both keys, not just the stale bits — once the TTL lapses, and something in the swarm/mDNS bookkeeping ends up giving your own peer id a short TTL too. About a minute in, it quietly deletes your own keypair out of your own peerstore, and the next publish crashes trying to sign with a key that's gone. I just don't start that task.

One more gossip quirk that bit me: GossipSub doesn't replay anything to a late subscriber. If I published the top-level goal the instant the decomposer process started, and the scorer process hadn't finished subscribing yet, it would just silently never see it — no error, no retry, the whole thing stalls forever. So before publishing a submitted goal, I wait until I can see at least one other subscriber has actually mesh-joined the topic, and only then publish — with a timeout so a solo run doesn't hang forever.

## The agents

Every agent is the same CLI entrypoint (`python -m decentralized_decomposer.main --role <role> --port <port>`), just started with a different `--role`. Here's what each one is actually doing.

### Decomposer

Watches for new goals, and for each one, asks the model to split it into somewhere between 2 and 6 independently actionable subgoals plus a short rationale for the split.

> **Info** — decomposer system prompt
> You are a task decomposition agent. Given a goal, break it into 2-6 subgoals that are each independently actionable. Do not produce subgoals that require judgment calls beyond what's stated in the goal. Return ONLY valid JSON matching this schema: `{"subgoals": ["string", ...], "rationale": "string"}`

I run more than one decomposer on purpose. They'll come back with genuinely different splits for the same goal, and that's the point — I wanted diversity of opinion, not one canonical answer.

Each proposal is a message that looks like this. This is then voted on by a scorer.

```json
{
  "goal_id": "1fb7d8c3a29e4410",
  "proposer_peer": "12D3KooWDecomposerPeerXyz",
  "subgoals": [
    "Book round-trip flights to Tokyo",
    "Reserve accommodations for 14 nights",
    "Draft a day-by-day itinerary"
  ],
  "rationale": "Splits the trip into booking, lodging, and planning tracks that can proceed independently.",
  "proposal_id": "a92f61e0c7d3b8aa"
}
```

### Scorer

Rates every competing split it sees, 0 to 1, against a fixed rubric: does it cover the whole goal, do the subgoals overlap, is each one actually atomic or still too coarse. Once enough scorers agree a split is good (more on exactly what "enough" means below), the scorer publishes the accepted split and moves each subgoal through a second, separate LLM call that decides whether it's atomic or still needs splitting.

> **Info** — scorer system prompt
> You are evaluating a proposed task decomposition against a fixed rubric. Score 0.0-1.0 on: (a) does the split fully cover the original goal, (b) are subgoals non-overlapping, (c) is each subgoal actually atomic/actionable or still too coarse. Return ONLY valid JSON matching: `{"score": float, "notes": "string"}`

> **Info** — atomicity classifier prompt
> Classify whether this subgoal is atomic (a single concrete deliverable requiring no further decomposition) or composite (still needs breaking down). Return ONLY: `{"is_atomic": bool, "reason": "string"}`

Composite subgoals get republished as brand new goals, one level deeper, and go through this whole cycle again. Atomic ones get announced for executors to pick up. There's a hard depth cap too (3 by default) — once depth+1 would hit that cap, a subgoal gets forced atomic no matter what the classifier says, so recursion can't run away on me.

Each scorer sends a score back with notes.

```json
{
  "goal_id": "1fb7d8c3a29e4410",
  "proposal_id": "a92f61e0c7d3b8aa",
  "scorer_peer": "12D3KooWScorerPeerAbc123",
  "score": 0.85,
  "notes": "Covers the full goal, subgoals don't overlap, all look atomic."
}
```

### Executor

Declares up front what it's willing to do — currently `can_run_code`, `can_query_api`, `can_write_text` — and races other executors to claim atomic leaf prompts that match. Whoever wins the claim actually runs the prompt through the model and publishes the result.

> **Info** — execution prompt
> You are an execution agent. Complete the following atomic task and return only the deliverable -- no preamble, no meta-commentary.

Worth being honest about: `can_run_code` right now just means the executor is willing to *produce* code as text. It doesn't actually run anything on the host. Building a real sandbox for that is on the list, not something I've done yet.

### Observer

The odd one out — no LLM calls, no capabilities, doesn't publish anything. It just subscribes to everything, keeps folding what it sees into a local copy of the plan state, and prints the reconstructed tree to stdout whenever it changes. This is genuinely the only reason I can actually see the whole plan while it's running — every other agent only ever sees its own slice.

![Decompoer Detailed view](/images/competing-proposals.svg)

## The protocol

GossipSub doesn't support wildcard subscriptions, so every goal — top-level or one spawned three levels deep by recursion — gets announced on one well-known topic, `goal/new`. From there, that goal's own propose/score/accepted topics hang off its own id:

```
goal/new                                  ← every Goal announced here
node/new                                  ← every atomic leaf announced here
capabilities/announce                     ← executors announce what they can do

goal/<goal_id>/propose
goal/<goal_id>/score
goal/<goal_id>/accepted
goal/<goal_id>/node/<node_id>/claim
goal/<goal_id>/node/<node_id>/result
goal/<goal_id>/node/<node_id>/failed
```

Every id in there — `goal_id`, `proposal_id`, `node_id` — is a content hash, not something the sender just gets to declare. `goal_id` is a hash of the goal text, timestamp and originating peer; `node_id` hashes the goal id, the node's text, and whether it's atomic. Any peer that receives a message can recompute the hash itself and check it matches before trusting anything in the message. Anything that fails that check, or just fails basic schema validation, gets logged and dropped, never handed to the agent logic.

## Claiming work

When an executor sees a new atomic node it can handle, it publishes a claim stamped with the current time, then waits out a short two-second window collecting any competing claims from other executors also going for it. Whoever's claim has the earliest timestamp wins — ties broken by peer id so every executor watching agrees on the same winner without needing to ask anyone.

One thing I had to guard against: GossipSub doesn't guarantee exactly-once delivery, so the same "new node" message can legitimately show up twice over different mesh paths. Without deduplicating by node id, two handlers for the same node both end up subscribing to the same claim topic, and whichever finishes first tears that subscription down out from under the other one — which crashes the process instead of just skipping cleanly.

If a node just sits there unclaimed for a minute, it gets re-announced once. If it's still stuck after a second minute, that's treated as a real capability gap — nobody present can do it — and a failure notice gets published instead of looping on it forever.

## How proposals get accepted

This part I wanted to get right, since it's the bit that actually makes the "no single scorer decides alone" idea real. A split isn't accepted just because one scorer liked it:

- The first proposal to collect confirmations from **2 distinct scorers**, each scoring **at least 0.6**, is accepted immediately, with its final score being the mean of every vote it got.
- If nothing clears that bar within 30 seconds, it falls back to the highest mean-scored proposal among whichever ones at least got 2 votes — a proposal with only one scorer's opinion never gets accepted just because time ran out.

Ties get broken by the order proposals showed up in, so given the same sequence of messages, every scorer lands on the same decision independently. That's what lets scorers that never talk to each other converge on one answer.

## The code

The whole thing is roughly 1,940 lines of Python. Biggest chunk is the agent logic (decomposer/scorer/executor/observer), then the libp2p plumbing (host, discovery, pubsub, the CRDT-ish plan state), then the message schemas and topic naming.

One implementation note: the original plan was `asyncio`, same as the agent stack I built last time. But the only maintained Python libp2p implementation — the one that actually gives you GossipSub and mDNS — is built entirely on `trio`, not `asyncio`. So this one runs on `trio` throughout instead. Same job either way: one event loop per process that keeps handling gossip and heartbeats while LLM calls are awaited without blocking anything else.

Plan state itself is a handful of grow-only sets (goals, accepted splits, atomic nodes, results), keyed by those content hashes. Adding the same entry twice is a no-op, merging two states is just a union, so replaying the same gossip in any order or any number of times lands on the same final tree for every peer. That's what the observer leans on to reconstruct things — and it's genuinely eventually consistent, not eventually complete: a branch it hasn't heard about yet just doesn't show up, rather than erroring.

Testing this one was its own thing to think about, because there's no single log to assert against. I split it into layers — pure logic (schema validation, the CRDT merge being order-independent, claim tie-breaks being deterministic) with no network or LLM involved, and agent decision logic tested against a fake in-memory pubsub with a mocked model client. Real multi-process races and live API calls are still a manual/future thing — reproducing claim races and gossip timing bugs reliably, even on one machine, is genuinely harder than the agent logic itself.

## Running it

No bootstrap node, no shared config file — just start each role in its own terminal and let mDNS do the rest:

```bash
export ANTHROPIC_API_KEY=sk-...

python -m decentralized_decomposer.main --role decomposer --port 4001 \
    --submit-goal "Plan a two-week trip to Japan"

python -m decentralized_decomposer.main --role scorer --port 4002

python -m decentralized_decomposer.main --role executor --port 4003 \
    --capabilities can_write_text,can_query_api

python -m decentralized_decomposer.main --role observer --port 4004
```

The observer doesn't need an API key or any capabilities — start it alongside the others and watch it print the plan tree as it fills in, straight from what it's overheard. Run a couple more decomposers or executors on other ports and you'll actually see competing proposals and claim races happen, instead of just reading about them.

The output from the observer shows that the high level goal is broken down into multiple steps, then each step is further broken down and expanded upon. Each one has either a **decomposed** or a **pending decomposition** after it denoting the current status.


```bash
=== plan state for goal ec724d522767d724 ===
Plan a two-week trip to Japan [decomposed]

  * Determine trip logistics: choose travel dates, set a total budget, and book round-trip flights to Japan [decomposed]
    * Research and select specific travel dates for the Japan trip, considering factors like season, holidays, and personal availability [pending_decomposition]
    * Define a total trip budget by estimating costs for flights, accommodation, food, transportation, and activities [pending_decomposition]
    * Search for and compare round-trip flights to Japan (e.g., Tokyo, Osaka) on platforms like Google Flights, Expedia, or Skyscanner for the chosen dates [done]
      * Result: # Round-Trip Flight Search to Japan: Comparison Report > **Important Note:** I'm a text-based AI and cannot browse the internet, access live flight databases, or perform real-time searches on Google F...
    * Book the selected round-trip flight by completing payment and saving confirmation details [pending_decomposition]
  * Decide which cities and regions to visit and create a day-by-day itinerary covering the two weeks [pending_decomposition]
  * Research and book accommodations (hotels, ryokans, hostels) for each location on the itinerary [pending_decomposition]
  * Identify and list key activities, attractions, and restaurants for each destination, and pre-book any that require reservations (e.g., teamLab, specific restaurants) [pending_decomposition]
  * Handle practical travel preparations: obtain a Japan Rail Pass, arrange airport transportation, set up a pocket Wi-Fi or SIM card, notify your bank, and check visa requirements [pending_decomposition]
```

## Summary

This ended up being a reasonably flexible way to turn a fuzzy goal into a tree of concrete, executable prompts without any one process owning the whole plan. Any single decomposer, scorer or executor can die mid-plan and the rest just keep going. Add more executors and it scales the way adding more people to a task does, not the way a bigger server does.

The trade-off is real too — a scoring round is slower than one LLM call would ever be, there's no global optimum being computed anywhere, just whatever a handful of independent peers happened to agree on, and gossip arriving out of order can leave two peers looking at slightly different plans for a few seconds at a time.

The use cases for this type of decentralised fuzzy logic are endless. The decentralised nature lends itself itself to being able to withstand multiple failures. Not having a centralised control plane also means that it's difficult to actually stop a goal being decomposed once it's started if there are enough executors and scorers present.

I'm thinking about what a real sandbox for `can_run_code` would look like next — right now it's just an executor that's willing to write code, not run it — but that's a separate rabbit hole for another post.
