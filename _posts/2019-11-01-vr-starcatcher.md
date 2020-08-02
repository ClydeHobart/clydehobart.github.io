---
layout: post
title:  "VR Starcatcher"
date:   2019-11-01 12:00:00 -0500
categories: jekyll update
---

This was another project for my AR & VR development class. Using the Oculus Quest, users were able to a projection of the 12 Zodiac constellations. Using the Quest controllers, they could trace out connecting beams between stars of the same constellation. Once all stars were traced, an object would appear in the other hand with a symbol and description of the constellation. I collected the star coordinates and used a Python script to translate the coordinates from the spherical star coordnates to cartesian coordinates, writing these Unity-suitable coordinates in a C# file. I also wrote the code to generate these stars in game, and interact with them via connecting beams.

<div style="text-align:center"><a href="https://youtu.be/U1_5_qmGX8k">
    <img
        src="/assets/vr-starcatcher/vr-starcatcher.png"
        alt="VR Starcatcher thumbnail"
        title="VR Starcatcher on YouTube"
        width="300px"/>
</a></div>

## Details

* Framework: **Unity**
* Languages: **C#**, **Python**
* Hardware: **Oculus Quest**
