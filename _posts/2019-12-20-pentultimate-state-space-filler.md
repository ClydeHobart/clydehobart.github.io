---
layout: post
title:  "Pentultimate State Space Filler"
date:   2019-12-20 12:00:00 -0500
categories: jekyll update
---

This project was intended to fill out the state space for my favorite twisty puzzle, the Pentultimate (pictured below). The idea was to write a program that would assemble, through BFS, an "all roads lead to Rome"-type graph to provide the shortest move list to solve the puzzle. It didn't take long into testing for me to realize that not only was the time this would need to fill the graph extremely large, but also the memory needed to store it was more than I had really thought about and/or anticipated. As a result, the problem still lies unsolved, but the code up until the BFS (and the BFS process itself) is still solid, just not reasonable for space and time constraints.

<div style="text-align:center">
    <img
        src="http://www.twistypuzzles.com/museum/large/01741-07.jpg"
        alt="Pentultimate puzzle"
        title="The Pentultimate (from twistypuzzles.com)"
        width="300px"/>
</div>

## Details

* Language: **Python**
* GitHub Gist link: `https://gist.github.com/ClydeHobart/0f59cb84b4f7241c219225a88cdaa451`

<script src="https://gist.github.com/ClydeHobart/0f59cb84b4f7241c219225a88cdaa451.js"></script>