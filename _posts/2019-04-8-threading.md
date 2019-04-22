---
layout: post
title: Predictive threading in Coldsnap
date:   2019-04-8 19:44
description: An explanation of the threading model used in our custom 3D game engine, Coldsnap
toc: true
tags: engine threading
---

# Introduction
During the development of [Hilo]({{ '2019/04/hilo' | relative_url }}), the first game we developed as [Flatline]({{ 'flatline' | relative_url }}), one of the main questions we had before moving forward with development was what the threading architecture of our eventual engine would look like.

We thought that establishing a solid threading model and building our engine with threading in mind from the very beginning would benefit us a lot later down the road, and save us a lot of time we'd otherwise spend refactoring code as systems grew too big and slow. It would also provide more room for performance earlier on in development, by giving us more headroom in frame-time.

I took it upon myself to be the architect of our threading engine, and applied a method that I had been toying around with in my free time but had yet to bring to scale; predictive, dynamic threading.

*Note: I have no idea if that's the correct name for this, but I haven't come across a similar model before.*

# Overview
In short, any work that is to be performed once per frame in our engine is to be sumitted as a **WorkItem**, a small self-contained package that holds a callback to the work to be performed and some telemetry the scheduler uses to optimize frame-time. **WorkItem**s are fire-and-forget, you queue them all up as the engine starts, and never have to think about them again; they will be performed once per frame, but you cannot guarentee in what order or on what thread.

There are a few specialized types of items that can be submitted for non-repeating work: **IOWorkItem**, **ProcessingWorkItem**, and **NetworkWorkItem**, which are handled separately from the main bulk of work.

# Internals
## Prediction
Each **WorkItem** contains telemetry that the scheduler can use to predict how long it will take next frame. The main two pieces of data used are the average time it has taken over the past 60 frames, the delta duration since the last frame, and an average delta over 10 frames. With this information the scheduler can determine with pretty good certainty how long it will take to perform that work next frame.

The case for this being so reactive and dynamic is to handle, say, scene transitions where the new scene is more demanding on certain subsystems than others. This gives us great flexibility in game and level design; allowing for one scene to be graphically and effects-intensive and for the next to focus heavily on AI and pathfinding, with the engine seamlessly adjusting its architecture to match.

## Scheduling
Upon startup the engine submits an internal **WorkItem**, called "Schedule frame". It's job is to schedule the upcomming frame using the data collected in the work items.
It performs the following work:

- Gather up all **WorkItem**s
- Predict how long they will take
- Sort them by predicted execution duration
- Use a best-fit algorithm to pack them as tightly as possible into #-thread queues
- Submit the new queues back to the scheduler for use in the next frame

*Note: The number of threads used depends on the availible threads on the target hardware, with an exception; more on that later*

## Synchronization
Upon completion of their work queue, each thread will report back to the scheduler via condition variable, telling it that the work is complete and that it's ready to recieve a new queue and start a new frame.

The thread that originally dispatched the scheduler is responsible for waiting for it to report completion, but other work can of course also be done on that thread in the meantime. In Coldsnap, the dispatcher thread is also the main program thread, and does the following:

- Singal beginning of frame
- Launch thread scheduler
- Handle windows events and input
- Poll active watched files for changes and queue them for reloading if needed
- Draw debug information and overlays
- Singal end of frame and push to the backbuffer

# Types of threads
As mentioned earlier, the scheduler doesn't utilize every hardware thread for per-frame **WorkItem**s, it actually leaves one thread free, the main program thread. The work it performs is very minimal, though, so it frees up the hardware thread for other things, which is there the other types of work come in.

**IOWorkItem**s are run on an auxilliary **IOThread**, that's responsible for any and all IO to the host device, including logging to the console, as these are operations that aren't inheretly thread-safe, and can't all be done from their respective callee threads. **NetworkWorkItem**s work in a similar fashion, and treat the network interface as an IO device of its own, they are mainly used for our [in-engine bug reporter]({{ '2019/04/bug' | relative_url }}).

In a typical 4-core/4-thread desktop system, that leaves us with 3 threads dedicated to work, and the main thread bouncing between IO, Network, and unthreadable OS-specific work.

# Improvements
There are many improvements that could be made to this system; the most glaring being that a single **IOThread** is responsible for all IO on the host device. This should be split into one **IOThread** per IO interface on the system automatically, for example one thread for console output, one for the main drive, and one for the eventual data drive that the game perhaps could be running off of. This wouldn't just improve IO thoughput a lot by not doing the work in sequence, but also the latency when submitting work; it's not uncommon for console logging to appear a few seconds late due to large files being queued up for loading.