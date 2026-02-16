---
title: "Learning DOS2 Modding by Stealing Things"
date: "2026-02-16T20:00:00Z"
description: "What contributing a steal feature to a 740-file Divinity: Original Sin 2 mod taught me about reverse-engineering game engines and collaborating on unfamiliar codebases."
tags: ["gamedev", "lua", "modding", "reverse-engineering"]
---

## The Itch

I've been playing Divinity: Original Sin 2 with [Epip Encounters](https://www.pinewood.team/epip/), a quality-of-life mod that, among other things, adds a Quick Loot feature. Hold a keybind, search nearby containers and corpses, loot everything from one UI. It's great. But it respects the law: NPC-owned items don't show up.

I wanted to steal with it. Sneaking through a merchant's back room and cherry-picking items from a unified loot list sounded like the way theft in an RPG *should* feel. Epip's author, Pip, had considered the idea before but shelved it because the crime system seemed too convoluted. I figured I'd try anyway, partly because I wanted the feature and partly as a challenge: could I onboard myself into a 740-file Lua codebase I'd never touched and ship something that met the project's standards?

## Reading the Codebase

Epip isn't a simple mod. It has a custom OOP system with multiple inheritance, a Feature/Library architecture for organizing modules, a client-server networking layer with typed payloads, and a Flash-based UI framework driven from Lua. The Quick Loot feature alone spans four files across shared declarations, client logic, server logic, and UI rendering.

Before writing anything, I traced how searches work end to end. The client starts a search, expands a radius over time, discovers containers and corpses within range, asks the server to generate loot in containers that haven't been opened yet, waits for item sync, then throws a completion event that the UI consumes. Items pass through a chain of hookable lootability checks. Picking up an item sends a net message to the server, which moves it into the player's inventory.

The codebase made this tractable. Epip uses extensive EmmyLua type annotations, so my editor could show me class hierarchies, parameter types, and hook signatures. The Feature pattern is consistent: once you understand one feature, you can read any of them. Good architecture pays forward.

## Collaborating with the Author

I opened a [pull request](https://github.com/PinewoodPip/EpipEncounters/pull/27) with my initial implementation and wasn't sure what to expect. Pip's first response was encouraging: he noted the PR was impressively consistent with the codebase's conventions and overall design, and said the feature "kinda proves me wrong" about the crime system being too convoluted to attempt.

What followed was a genuinely productive design discussion. My initial version had a separate "Search (Steal)" keybind with its own input action. Pip pushed back: keybind space is exhausted for controller users, and a second bind adds configuration fatigue. His suggestion was elegant; activate steal mode when the player is already sneaking. Since you'd never want to steal without sneaking anyway, the sneaking state *is* the intent signal.

That simplification had a consequence I spotted while working through the redesign: the "accuse on container access" setting I'd built became unnecessary. If sneaking is a prerequisite for steal mode, NPCs can't see you access the container in the first place. A setting and its associated server-side crime registration code could be removed entirely. Pip agreed, and the feature got simpler.

He also suggested red enemy-style highlights on NPC-owned containers during steal searches, escape and right-click to cancel searches (which helped avoid accidental crimes), and naming tweaks for the UI labels to match Epip's existing style. Each suggestion made the feature simpler or more polished. The final design was meaningfully better than what I'd submitted.

## The Crime System

The most interesting technical problem was crime registration. DOS2 has several crime types, and they behave differently in ways that aren't documented outside the engine.

My first attempt used `"Steal"`, the crime type the engine uses when you pick items up off the ground. It works from Lua, but it resolves witnesses near the *criminal's current position* at registration time. If you register the crime after a delay (to give the player time to leave), random NPCs near the player react instantly instead of the victim investigating. Not the behaviour I wanted.

After digging through the game's goal scripts, I found how vanilla pickpocket detection works. It uses `"EmptyPocketNoticed"`, a delayed-detection crime type where the *victim* investigates after a delay. I wired this up with a randomized 5-15 second timer via `ProcObjectTimer`, which conveniently restarts from the last theft if you steal multiple items from the same NPC. The result matches vanilla's "the owner eventually notices" behaviour.

Getting here required trying several crime types, watching NPCs react in wrong or broken ways, reading Osiris scripting goals to find vanilla patterns, and piecing together undocumented parameter semantics (the fourth parameter to `CharacterRegisterCrimeWithPosition` is the crime *witness*, not the victim, despite what you'd assume). That kind of reverse engineering is slow, but there's no substitute for it when the documentation doesn't exist.

## What the Engine Won't Let You Do

I spent time trying to change the search circle effect to red during steal mode. The Script Extender exposes effect objects with methods like `OverrideVec4MaterialParameter` that look like they should control material properties. In practice, the method's signature is undocumented and I couldn't get it to produce visible changes. The effect structures (decals, particle systems) expose property arrays that appear empty through the Lua API.

Pip confirmed that while the particle system structures are technically accessible via the extender, modifying them in code would be "ridiculously overkill" compared to creating a variant effect in the Divinity Engine editor. He offered to add a red reskin after the merge. Sometimes the right answer is knowing which tool to use, even if it means someone else handles that part.

## The Simplification That Fell Out of Review

In the final review pass, Pip noticed that my pickup net message included an `IsSteal` flag sent from client to server. In non-steal mode, the client already filters out NPC-owned items through lootability hooks. If the server receives a request for an NPC-owned item, it's stolen by definition. The server can check ownership itself.

I'd needed the flag in an earlier, more complex implementation and hadn't noticed it was vestigial. This is the kind of thing that only surfaces when someone else reads your code with fresh eyes. We removed the flag and let the server always check item ownership: simpler, fewer fields to trust across the network boundary, and one less thing that could desync.

## What Carried Over

The specifics of DOS2 modding are niche, but the underlying skills aren't. Reading a large unfamiliar codebase systematically before writing anything. Tracing behaviour through layers (Lua, Osiris, engine internals) to understand *why* something works the way it does. Testing hypotheses against undocumented systems. Designing around platform constraints. Iterating on a design through code review until it's simpler than what you started with.

Working in someone else's well-structured codebase is one of the fastest ways to pick up new patterns. And contributing upstream, rather than forking, forces you to meet the project's standards rather than cutting corners. The four commits that landed were tighter and more thoughtful than anything I would have shipped on my own.
