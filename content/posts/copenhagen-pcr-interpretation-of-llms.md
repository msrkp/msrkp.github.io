---
title: "Copenhagen-PCR Interpretation of LLMs"
date: 2025-06-06T17:17:59+05:30
draft: false
cover: /img/nz/IMG_5210.jpeg
layout: wide
images:
 - /img/nz/IMG_5210.jpeg
---

# Intro

What’s coming is both exciting and a little scary. I've been looking for the right analogy to explain this feeling, the sense that something fundamental is shifting in how machines interact with *knowledge* and how it might distrupt every corner of the economy.

From the beginning, knowledge was something confined to the walls of the human mind, that is discovered, refined and passed through language, intuition and teaching. But, now, for the first time, machines are not only consuming that knowledge, but learning how to replicate it(And replication, frankly, is what most human jobs are). The machine doesn't understand it, it applies. At scale, and often better than us and without tiring.

I've came up with one analogy which combines two ideas from science: One from quantum physics, the other from biology. It probably doesnt map perfectly. But I hope it pins the essence of what's happening.

I call it Copenhagen-PCR Interpretation of LLMs.

# Most jobs/task have methodology
There are many ways to explain what’s happening, but the core idea is simple. This blog focuses specifically on vulnerability identification, but I believe the same pattern will generalize to many other fields.

In general, any task we do has a method, a way to do it. That method may not be obvious to outsiders, but for someone with domain expertise, it's usually well understood and repeatable.

Let’s take security research as an example.

1. Discovering a vulnerability is like finding a needle in a haystack.
2. But not every needle is equally hidden. In fact, many follow predictable patterns. Experienced pentesters and researchers use heuristics, mental shortcuts, and known bug patterns to find them quickly.
3. It’s true that some bugs are novel, hard to describe, and don’t follow any known pattern. But I would argue that’s only around 5%(even less maybe) of all exploitable bugs in the real world. The remaining 95% are detectable using heuristics.
4. But the important point is, these heuristics can be written down — in plain language, or even as code by the domain expertise if they try hard enough. (And with every new generation of intelligent models, that translation step becomes easier.)
5. As far as LLMs are concerned, these heuristics are codifiable knowledge, meaning they can be represented in a way that the model can understand and act on. And once that happens, LLMs can replicate and apply them faster and more reliably than any human.

This is how LLMs form their "world model" of security. And this, I believe, will have serious consequences, not just in security research, but across every domain where knowledge can be made explicit.

The moment a method becomes describable, it becomes automatable.

And once it’s automatable, it becomes scalable, at a pace no human team can match.

# The Copenhagen Interpretation: Collapse the Vulnerability

In quantum physics, the Copenhagen interpretation says that particles exist in multiple states at once, until they are observed. Before measurement, a system is in a superposition: the cat is both dead and alive. But once observed, the wavefunction collapses into a single, definite state. In our case: vulnerable or not vulnerable.

That’s the metaphor I want to bring to vulnerability discovery.

A bug class, let's say, [“HTTP Desync Attacks: Request Smuggling Reborn”](https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn), might exist in thousands of codebases. But until someone like [James Kettle](https://portswigger.net/research/james-kettle) describes how to find it, it’s uncertain, invisible state — maybe already exploited by the NSA or North Korea, maybe not. As security researchers(the good ones), our goal is to collapse that uncertainty through observation.

That collapse happens every time a researcher writes a blog post, paper, gives a talk, or publishes a proof-of-concept. That act of observation, of defining the pattern clearly, is the moment the vulnerability class goes from abstract to real.

This is the 5% of the work (or likely less) where a cracked researcher defines a new class of vulnerability. And I’d argue the ratio is similar, or even smaller, in other fields.

Then begins replication by the remaining percentage of experts in the field. The research paper is compressed into knowledge, distilled and absorbed into the minds of other expertise in the field, who now carry the pattern forward, applying it in new contexts and codebases, finding new [variants](https://portswigger.net/research/browser-powered-desync-attacks), or same bug on different [software](https://bugzilla.mozilla.org/show_bug.cgi?id=1599259).

Some of these replications are clever or complex, but the essence is the same: once a bug class is observable, replication begins.

To find the variants of the bug class, however, the knowledge must first be deeply understood and translated into action. Not everyone can do this. It takes domain expertise to replicate.

But this is changing. The era of exclusive human expertise, at least when it comes to replicating the knowledge created by the top 5%, is coming to an end.

### The PCR Analogy: Replicate at Scale

> When a vulnerability heuristic is distilled into a form an LLM can understand, be it language, logic, or pseudocode, it stops being static knowledge and becomes executable, scalable action.

LLMs don’t need to invent the ideas. They just need to observe the codified knowledge. Once they do, they collapse the uncertainty across millions of codebases. No rest, no fatigue, no scrolling through X. The uncertainty is gone. 

What was once an uncertain state, observable only through human expertise, is now something a machine can detect, repeatably and at scale.

LLM deniers often point fingers and try to downplay their capabilities. But ask them this: How many of us actually invent novel ideas? The truth is, only the top <5% of people are true inventors. The rest of us, including most professionals, are just replicators of those memes. We read, remix, and reapply. The inevitable truth is that these LLMs just do that better, faster and without burnout.

In biology, there's a process called PCR (Polymerase Chain Reaction) — you probably heard about it during COVID testing. It's a technique used to take a tiny trace of genetic material and make millions of copies. 

It works by using short primer sequences that target specific parts of the viral genome (like SARS-CoV-2) in a sample taken from mucus or blood. An enzyme called DNA polymerase then builds new copies of the target sequence. Each cycle doubles the amount of DNA, so after 30–40 cycles, that one fragment becomes billions of copies, enough to easily detect.

This is what LLMs do to vulnerability identification.

1. The heuristic or codified knowledge is the primer which is created by the domain expertise.
2. The codebase is the template.
3. The LLM(Polymerase) is the enzyme that does the copying.

Once a bug pattern is codified and observed, LLMs can replicate it across thousands of projects — faster, more consistently, and with high precision. They might even uncover variants and edge cases, though that may still require some RL or fine-tuning to guide the exploration.

What once took years of learning to become domain expertise can now be duplicated by a model in milliseconds.

# What This Means for Security Research
This fundamentally changes the role of the human in the loop.

> Discovery is the last moat.

The real challenge now is not in applying knowledge, machines can do that better, faster, and at scale. The challenge is in creating it: collapsing new bug classes, defining new heuristics, and pushing the boundary of what’s observable.

Once the insight is out there, once the pattern is named, LLMs will pick it up, replicate it, and scan the world with it. What used to take a career’s worth of intuition can now be duplicated by a model in milliseconds.

If you are not creating or discovering, you will be replaced.