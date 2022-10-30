---
layout: post
title:  "Pentultimate Solver"
date:   2022-10-30 17:14:00 -0400
categories: jekyll update
---

Revisiting a [concept from my senior year of college](/jekyll/update/2019/12/20/pentultimate-state-space-filler.html), I wanted to put together something that can solve the Pentultimate puzzle. Given that my physical copy of the puzzle frequently jams up and has pieces pop out, I also wanted this to offer a more comfortable puzzle solving experience than what I can physically achieve. I'd say I succeeded in about 1.5 of these two items. The UI/UX isn't amazing, but it's reliable and capable, even if understanding all of the application's functionality requires some technical knowledge about how things are orchestrated behind the scenes. Most importantly, it can definitely solve the puzzle. To properly demonstrate the application, I put together a [demo video](https://youtu.be/rNOOAjImquU). The project [README.md](https://github.com/ClydeHobart/pentultimate_solver) covers the same content as the demo video, if not slightly more in depth.

<div style="text-align:center"><a href="https://youtu.be/rNOOAjImquU">
    <img
        src="/assets/pentultimate-solver/thumbnail.png"
        alt="The Pentultimate puzzle framed in the center, against a gray background"
        title="The Pentultimate Solver demo video on YouTube"
        width="300px"/>
</a></div>

## Details

* Framework: **Bevy**
* Language: **Rust**
