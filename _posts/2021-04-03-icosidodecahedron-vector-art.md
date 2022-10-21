---
layout: post
title:  "Icosidodecahedron Vector Art"
date:   2021-04-03 19:38:00 -0500
categories: jekyll update
---

Inspired by some time spent messing around with fractals in Blender in early high school, I have spent the greater part of a year working on a program to generate `.svg` image files of fractals constructed with icosahedra and icosidodecahedra. After realizing that the files for the higher-order fractals were massive, over-inflated with elements for faces, lines, and vertices obstructed by elements rendered above them, I set out to improve the program by culling these hidden elements. The majority of the total development time was spent iterating on this culling algorithm to take the union of a collection of polygons and another polygon, determining if the newly introduced polygon contributed to the result or not. I've written about this in more depth on the project [README.md](https://github.com/ClydeHobart/IcosidodecahedronVectorArt/blob/main/README.md).

<div style="text-align:center"><a href="https://github.com/ClydeHobart/IcosidodecahedronVectorArt">
    <img
        src="/assets/icosidodecahedron-vector-art/Icosidodecahedron_5FoldSymmetry_3.png"
        alt="Iteration 3 of the icosidodecahedron fractal, viewed from the perspective with 5-fold rotational symmetry. It's mostly cyan with a large quasi-circular hole in the center. Various other holes can be seen, forming pentagons and dodecagons with the other holes"
        title="The IcosidodecahedronVectorArt project repo on GitHub"
        width="300px"/>
</a></div>

## Details

* Language: **C++**
