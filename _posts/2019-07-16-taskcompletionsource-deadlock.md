---
layout: post
title: Resolve deadlock when using TaskCompletionSource
category: coding
description: Learn how to resolve deadlocks when using TaskCompletionSource
tags: [coding, c#, .net]
---

MMALSharp is a library I maintain for the Raspberry Pi camera module. It's been an important goal of mine to make the library asynchronous so that it can be used within a GUI application without blocking user interaction. 

Early development of MMALSharp featured Stephen Cleary's excellent AsyncEx library, which I made use of for its `AsyncCountdownEvent` in order to detect when processing on the camera had completed so then I could cleanup any unmanaged resources. This worked well, except I would infrequently experience deadlocks and the application would hang. I spent a long time trying to figure out what was causing this - I eventually realised that what was happening was when the `AsyncCountdownEvent` was triggered, processing would continue on the thread currently being awaited and the background thread which I triggered on would never complete. This unfortunately resulted in me stopping using AsyncEx within MMALSharp, and I went with my own, rather crude way of detecting when the camera had finished processing - fire up a new task and asynchronously wait, periodically checking if we'd finished. For reference, it looks like this, brace yourself...

```
tasks.Add(Task.Run(async () =>
    {
        while (!port.Trigger)
        {
            await Task.Delay(50).ConfigureAwait(false);
        }
    }, cancellationToken));
```

I've never been happy with this solution and was convinced there must be a better way. I've recently come across the `TaskCompletionSource` class, and wondered if this could solve my problems. Essentially what I wanted to do was await the TaskCompletionSource's `Task` and call `SetResult` when processing had completed (for reference: the native MMAL library calls a function pointer delegate which is handled by a C# background thread) - sounds simple enough, so I wrote the required code and tried to take a picture on the Pi's camera...deadlock! I was quite shocked but also intrigued to understand why this was happening. So after a bit of research, I came across this [GitHub issue](https://github.com/EventStore/EventStore/issues/1179) which states that after calling `SetResult`, processing immediately continues on the thread awaiting and breaks away from the thread who called `SetResult`... sound familiar? I suspect this issue is what was causing AsyncEx's `AsyncCountdownEvent` to behave in such a way too.

Luckily there is a simple solution to this problem and that is to wrap calls to `SetResult` in `Task.Run`, queuing it up to run on the thread pool. After implementing that change, I receive no deadlocks and my code looks much cleaner now.

This change is due to feature in v0.6 of MMALSharp and I'm hoping to receive some performance improvements from it too.

Have you ever come across this problem before? It appears to be undocumented behaviour ([Stephen Cleary's blog](https://blog.stephencleary.com/2012/12/dont-block-in-asynchronous-code.html)).