---
title: "How Architects Design Stuff That Actually Scales"
date: "2026-05-18T19:00:00+05:30"
description: "Breaking down Gregor Hohpe's approach to building frameworks that scale and survive failure — from the Beyond Coding interview that every sysadmin should watch"
summary: "A deep look at how architects like Gregor Hohpe (Google Cloud, AWS veteran) design frameworks that scale and handle failure. The 8-step framework, the cylinder problem, and why step 8 is the hardest."
draft: false
tags: ["tech", "architecture", "scalability", "fault-tolerance", "system-design", "gregor-hohpe", "beyond-coding"]
categories: ["Tech"]
author: "Sanjay Upadhyay"
toc: true
slug: "architects-design-complex-frameworks"
---

I stumbled on a YouTube video the other day that changed how I think about building systems. It's Gregor Hohpe — the guy who wrote *Enterprise Integration Patterns* and *The Software Architect Elevator*, spent years at both Google Cloud and AWS — being interviewed on the *Beyond Coding* podcast.

The title is "Google & AWS Veteran: What Top Tier Software Architects Do Differently." I went in expecting the usual architect-talk about UML diagrams and decision trees. Instead I got something much simpler.

One commenter on the video boiled the whole interview down to 8 steps. And honestly, it's the most useful architecture framework I've come across:

> 1. Identify goals
> 2. Identify requirements to achieve goals
> 3. Draft the simplest solution
> 4. List all the ways it will blow up
> 5. Address those
> 6. Explain it to someone and have them explain it back to you
> 7. Address misunderstandings
> 8. You should probably stop here

I've been thinking about this for weeks. Here's what I took away from Hohpe's work and that conversation.

## The first mistake every framework makes

Hohpe's approach is refreshingly anti-hype. Before you draw a single box in a diagram, you have to answer one thing: **what are we trying to achieve?**

This sounds obvious. But be honest — how many times have you started a project because you wanted to "use Kubernetes" or "go microservices" without actually defining what problem that solves?

> *"Excessive complexity is nature's punishment for organizations that are unable to make decisions."* — Gregor's Law

That quote lands hard if you've ever worked on a framework that tried to do everything for everyone. I have. It was terrible.

## The simplest solution then stop

Steps 3 and 8 are the ones that matter most. "Draft the simplest solution" and "You should probably stop here."

Hohpe argues that architecture isn't about adding layers of abstraction. It's the opposite. **Architecture is about removing unnecessary options.** A framework that tries to support every possible use case isn't powerful — it's a trap. Every option you add makes the API surface bigger, the documentation longer, and the bug surface wider.

This is where I've messed up before. I built a monitoring framework once that had 27 config options. Zero of them made it better. The next version had 4 and everyone loved it. Should have stopped earlier.

One commenter on the video captured a great Hohpe quote: **"Don't try to make it simpler than that, make it intuitive."** Simplicity and intuitiveness are different dimensions. A framework can be simple but unintuitive (looking at you, raw Unix pipes) or complex but intuitive (well-designed APIs). The goal is both.

## The cylinder problem

Hohpe's signature example: two people arguing about whether an object is a rectangle or a circle. Both are right — they're looking at a cylinder from different angles. The architect's job is to see the cylinder.

For framework design, this means seeing multiple dimensions of the same system:

- **Logical view**: What are the modules and how do they relate?
- **Process view**: How do components interact at runtime?
- **Development view**: How is the code organized?
- **Physical view**: Where does it run? What are the constraints?
- **Scenarios**: Walk through a use case

The 4+1 architectural view model isn't academic overhead. It's how you avoid building something that works perfectly in theory but falls apart under load.

## Communication is half the job

**"Knowing something and being able to express something are two different skills."**

Another commenter on the video, @steve-adams, put it well:

> Teams generally don't succeed based on a single person's knowledge... the ability for teams to collaborate, grow, and execute better over time is highly dependent on everyone's ability to articulate their knowledge.

Top-tier architects don't write 200-page documents. They draw sketches, build metaphors, and make models that let teams make good decisions without asking for permission every time. The "elevator" in Hohpe's blog title — riding between the *penthouse* (business strategy) and the *engine room* (implementation) — is about being fluent in both languages.

If you can't explain your architecture to someone else in 5 minutes, it's either too complex or you don't understand it well enough. And frankly in a lot of cases it's both.

## How to build stuff that doesn't fall over

Here's where Hohpe's framework meets practical fault tolerance.

### 1. Assume everything blows up

Before you write error-handling code, write down every way your system can fail:

- What happens when the database is unreachable?
- When a dependency returns garbage data?
- When traffic spikes 100x?
- When a deployment goes wrong?

Each answer is a defensive design decision. This is basically failure mode analysis without the academic jargon.

### 2. Trade-offs are the job

Architects make trade-offs visible. Fault tolerance isn't about eliminating failures — it's about **bounded recovery**. The question isn't "will this fail?" but "when it fails, how fast do we recover and how much data do we lose?"

I've started framing every architectural decision this way. "If this component goes down, what happens?" If the answer is "everything stops working," you need bulkheads. Simple.

### 3. Skills multiply, not add

Hohpe argues that being an architect isn't the sum of your skills — it's the product. Technical skills × communication × organizational awareness. If any one is zero, the whole thing is zero.

For fault tolerance, this means:
- **Technical**: Circuit breakers, retries, backpressure, isolation
- **Communication**: Making failure scenarios visible, setting expectations on recovery
- **Organizational**: Designing teams that match the architecture (Conway's Law, but deliberately)

### 4. Simplicity IS fault tolerance

This is the counterintuitive one. The more complex your framework, the more ways it can break. Every abstraction layer, every dynamic config path, every "clever" optimization is a potential failure mode.

Hohpe's approach: constrain the options. Force good decisions. Make the wrong path harder than the right one.

At Google, Borg was deliberately opinionated about resource management. It didn't give developers unlimited knobs. That's why it could schedule workloads at planetary scale without constant firefighting.

## Real examples that stuck with me

### The cylinder (again)
A developer sees a REST API. A product manager sees a feature endpoint. An architect sees both, plus the deployment topology, latency budget, cost per request, and failure domain. The cylinder problem in practice.

### People process technology
Hohpe spent time at a Big 4 firm decades ago where "people, process, technology" was the slogan. He notes these are just lists. The architecture isn't in the list — it's in how they relate. A fault-tolerant system isn't just good code. It's good code + good operational runbooks + a team structure that works when someone's paged at 3 AM.

### What Google taught him
At Google scale, you can't have a DBA manually tuning queries. The system has to self-heal. This philosophy is what makes frameworks truly scalable: automate everything, make failure modes explicit, and never assume a human will be awake when things break.

### Don't build what you didn't build
Hohpe's article on [not running what you didn't build](https://architectelevator.com/cloud/dont-run-what-didnt-build/) argues for managed services. In framework design: don't build your own database, message queue, or service mesh unless you absolutely have to. And if you have to, treat it like a product, not a side project.

## What this means for the home lab crowd

If you're like me — running Kubernetes on bare metal, monitoring with duct tape and Prometheus, building things that break at 2 AM — Hohpe's framework applies directly:

1. **Start with goals.** Before adding a service mesh to your cluster, ask what problem it solves. If it's "learning service meshes," call it a learning project. Don't pretend it's an architecture improvement.

2. **Document failure modes.** Your home lab will fail. The SD card will corrupt. The SSL cert will expire. List the ways it'll blow up and fix the ones that actually matter.

3. **Explain it to someone.** If you can't explain your Docker networking setup to another person, it's too complex. Simplify.

4. **Stop early.** The hardest step. That Redis cluster for a blog with 3 visitors a month? You should probably stop here.

5. **Use the 4+1 views.** Try it on your next project. If you can't draw one of the five views, you don't fully understand your architecture. And something will break.

## Where to go from here

Hohpe's [Architect Elevator](https://architectelevator.com/) blog is worth a read. Start with "Gregor's Law" and "Think Like An Architect" and see where you end up.

The whole video's on YouTube if you want to watch the conversation directly. Just search "Beyond Coding Gregor Hohpe" — it's a 52-minute discussion that'll change how you think about building systems.

Or don't watch it. The 8 steps above capture the essence anyway. And if you follow them — especially step 8 — you'll build better frameworks than most people. Sometimes the best architecture decision is knowing when to stop.

{{< comments >}}
