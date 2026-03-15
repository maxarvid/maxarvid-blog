+++
title = "Patchwork"
date = 2026-03-15
+++

![A screengrab of the modular websynth Patchwork](../patchwork.png)

At university I was fortunate to take some courses in computer music. The most fun I had was building MaxMSP patches that made a lot of bleeps and bloops. I've been wanting to create a kind of patch platform that imitates the flow of a modular synthesizer, but that runs entirely in the browser. Fast forward to Claude Code gaining all the attention it's been getting and I thought what a nice opportunity to challenge my hype-skepticism while also having a go at building a little nostalgia project. Result: [Patchwork](https://patchwork.maxarvid.com).

The bare bones project did indeed only take an afternoon to cobble together, relying on generating some specs, making a plan, editing those files and then executing. The code that Claude Code generated was... actually nice? But my main takeaway is this: I had previously not tackled this project because I felt I needed a deep dive in the inner workings of the Web Audio API. I of course never had the time or drive to start in that end. But with some trepidation, I let Claude take the reins in driving that part of the project. This has resulted in me still taking time to read up on the implementation of the Web Audio API, but after I have some code to play around with. Which is to say, complete success.
