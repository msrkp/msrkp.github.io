---
title: "Copenhagen-PCR Interpretation of LLMs"
date: 2025-04-21T17:17:59+05:30
draft: false
cover: /img/nz/IMG_5210.jpeg
layout: wide
images:
 - /img/nz/IMG_5604.jpeg
---

# Intro

What’s coming is both exciting and a little scary. I've been looking for the right analogy to explain this feeling, the sense that something fundamental is shifting in how machines interact with *knowledge* and how it might distrupt every corner of the economy.

From the beginning, knowledge was something confined to the walls of the human mind, that is discovered, refined and passed through language, intuition and teaching. But, now, for the first time, machines are not only consuming that knowledge, but learning how to replicate it(And replciation, frankly, is what most human jobs are). The machine doesn't understand it, it applies. At scale, and often better than us and without tiring.

I've came up with one analogy which combines two ideas from science: One from quantum physics, the other from biology. It probably doesnt map perfectly. But then again, has a poet ever drawn a perfect analogy? Maybe they’d do a better job than me. But let me try.

I call it Copenhagen-PCR Interpretation of LLMs.

# Most jobs/task have methodology
There are many ways to explain what’s happening, but the core idea is simple. This blog focuses specifically on vulnerability identification, but I believe the same pattern will generalize to many other fields.

In general, any task we do has a method, a way to do it. That method may not be obvious to outsiders, but for someone with domain expertise, it's usually well understood and repeatable.

Let’s take security research as an example.

1. Discovering a vulnerability is like finding a needle in a haystack.
2. But not every needle is equally hidden. In fact, many follow predictable patterns. Experienced pentesters and researchers use heuristics, mental shortcuts, and known bug patterns to find them quickly.
3. It’s true that some bugs are novel, hard to describe, and don’t follow any known pattern. But I would argue that’s only around 5%(even less maybe) of all exploitable bugs in the real world. The remaining 95% are detectable using heuristics.
4. And here’s the key: these heuristics can be written down — in language, or even as code by the domain expertise if tried hard
5. As far as LLMs are concerned, these heuristics are codifiable knowledge. And once codified, LLMs can replicate and apply them better than any human.

This is how LLMs form their "world model" of security. And this, I believe, will have serious consequences, not just in security research, but across every domain where knowledge can be made explicit.


# The Copenhagen-PCR Interpretation of Vulnerability Discovery

In quantum physics, the Copenhagen interpretation says that particles exist in multiple states at once, until they are observed. Before measurement, a system is in a superposition: the cat is both dead and alive. But once observed, the wavefunction collapses into a single, definite state.

That’s the metaphor I want to bring to vulnerability discovery.

A bug class, let's say, [“HTTP Desync Attacks: Request Smuggling Reborn”](https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn), might exist in thousands of codebases. But until someone like [James Kettle](https://portswigger.net/research/james-kettle) describes how to find it, it’s uncertain, invisible state — maybe already exploited by the NSA or North Korea, maybe not.

Security researchers collapse this uncertainty every time they write a blog post, give a talk, or publish a proof-of-concept. That act of observation, of defining the pattern clearly, is the moment the vulnerability class goes from abstract to real. And other researchers begin replicating the method and finding new variants[1](https://portswigger.net/research/browser-powered-desync-attacks), or same bug on different websites[2](https://bugzilla.mozilla.org/show_bug.cgi?id=1599259)

And now here’s the shift:

> Once the heuristic is clearly defined and expressed in a form an LLM can understand, it becomes accessible, and instantly scalable.

LLMs don’t need to invent the ideas. They just need to observe the codified knowledge. Once they do, they collapse the space across millions of codebases, no rest, no fatigue, no scrolling through X. The uncertainty is gone. What was once an obscure bug becomes a queryable pattern, and the model can now scan for it everywhere.

LLM deniers often point fingers and try to downplay their capabilities. But ask them this: How many of us actually invent novel ideas?
The truth is, only the top 0.1% of people are true inventors. The rest of us, including most professionals, are just replicators of those memes. We read, remix, and reapply.

But the story doesn’t end there.

In biology, there's a process called PCR (Polymerase Chain Reaction),which many of us heard about during COVID testing, a technique for taking a tiny trace of DNA and making millions of copies. It works by using a short primer sequence(from blood or mucus) to target specific parts of DNA. Each cycle doubles the amount, and after 30–40 cycles, that one fragment becomes billions of copies.

This is what LLMs do to vulnerability identification.

1. The heuristic is the primer which is created by the domain expertise.
2. The codebase is the template.
3. The LLM(Polymerase) is the enzyme that does the copying.

Once a bug pattern is codified and seen, LLMs replicate it across thousands of projects. They don’t just copy; they generalize. They find variants, edge cases, and cousins the original researcher didn’t even think to look for.

What used to be a niche discovery now becomes a self-replicating pattern, scaled globally in seconds.

# What This Means for Security Research
This shift changes the role of the human in the loop.

> Discovery is the last moat.

The real challenge now is not in applying knowledge, machines can do that better, faster, and at scale. The challenge is in creating it: collapsing new bug classes, defining new heuristics, and pushing the boundary of what’s observable.

Once the insight is out there, once the pattern is named, LLMs will pick it up, replicate it, and scan the world with it. What used to take a career’s worth of intuition can now be duplicated by a model in milliseconds.
