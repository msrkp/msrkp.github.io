---
title: "Ask the Right Question"
date: 2025-11-07T16:18:33+05:30
draft: false
categories:
    - security
keywords:
    - hacker
images:
 - /img/nz/IMG_5356.jpeg.jpg 
---


> To ask the right question is already half the solution of a problem. - Carl Jung

Whether solving a hard problem, building something people use, or breaking a system the first step is “asking a good question”. 

But that’s only half the journey. The other half is doing the work, pushing through the fog until the next question appears. Most questions start broad and shallow, and that’s perfectly fine; the key is to keep digging deeper, layer by layer, until real understanding emerges.

interesting questions doesn’t come easy and they are recursive, deeper the stack trace deeper the expertise

Questions that lead to generational products, scientific breakthrough, or critical vulnerabilities don’t appear magically. One has to have clear understanding of the problem space, one has to be deep down the stack trace, exploring the edges of what’s known and working at its frontier. Only then do your questions evolve in quality, and with that evolution comes true novelty.

Recently i found a vulnerability in Atlas. It started with a simple question “Can I hack the Atlas browser?”. It is a shallow question to ask, but its the good first step. Then the stack trace deepens

* How does Atlas work?
* How does the agent work?
* How does it talk to the browser?
* How does it use Mojo IPC?
* Can I send arbitrary Mojo messages to the browser that could start agent?

I reached that final question in about 10 mins and it’s an important question or hypothesis that led to a vulnerability. Not genius, but privilege. A privilege of last few years grinding on client side, AI agent and browser security. Anyone could do this, but experience and right knowledge makes it faster to come up with better questions or hypothesis that lead to results with certainty.

People are different, each uniquely positioned in space and time, shaped by their own mix of opportunities, curiosities, and constraints. Striving to reach the edge of a niche you care about can sometimes lead to a billion-dollar company or sometimes to nothing at all. But that’s the beauty of it. As they say, art for art’s sake; knowledge for knowledge’s sake. (Classic convenient myth and a psychological trick we play on ourselves to justify chasing an unknown result)

This a common scenario of all successful companies, they are at right time with right expertise which knowingly or unknowingly put themselves in. This morning, I was reading the Supabase founder’s blog and exploring how they started supabase. The central idea of supabase is to provide backend-as-a-service at scale. But, that’s not where it started.

> I wrote this originally to replace Firebase’s firestore database, which I wasn’t too pleased with. I needed the realtime functionality for messaging inside my apps.
> Thought the community here might like it. Postgres is an amazing database - with realtime functionality I was able to consolidate everything into one database

The question of “how to improve firebase firestore?” led to “Can postgres do realtime well?”, than question stack deepened in four years of working on this question to building a complete backend as a service that serves millions of users now. Maybe this isn’t a accurate representation of Supabase’s origin, but it’s a clean example of sitting at a real pain point and digging with the right questions until a solution appears.

All of this is to say that you have to ask the right questions, dig deep into the knowledge space, and push to the frontier, where the next question can change the world or just your trajectory.