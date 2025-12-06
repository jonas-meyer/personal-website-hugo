---
title: "Developing a city-builder; A Prologue"
date: 2025-12-06
draft: false
tags: ["game-dev", "rust", "city-planning", "bevy"]
---

### Personal background

My first glimpse at city planning and the city building game genre was in elementary school. As a reward for finishing our typing exercises quickly, we were allowed to play SimCity 3000, which was installed on the school computers for some unknown reason. Now, back then, just like the rest of my class, I wasn't very interested in the intricacies of road hierarchy, or the systems behind balancing land use zoning. Instead, we just lobbed some meteorites on unsuspecting sims and called it a day.

But the initial spark was there, and this continued with the release of SimCity 4. I received it for Christmas and spent 2 weeks just reading the game manual (back when manuals were still made!), instead of spending time at the beach like everyone else (we were on holiday). My parents were probably very happy with the money they spent on that trip! This time, unlike my previous destructive rampages, I interacted with the different domains in this game, setting up electricity and water, zoning residential, commercial and industrial areas (probably closer than they should have been), setting up fire and police departments.

It was only when Cities Skylines came out, though, that things really clicked. Everything was just a bit more fluid than in SimCity 4. You could track individual citizens, see where they lived, how their surroundings impacted their commute, and how a lack of fire department coverage caused their untimely demise. While I probably spent more time playing SimCity 4, I learnt more from Cities Skylines. Of course, not being a kid probably helped.

The release of Cities Skylines II made me start to think that something better must be possible. The sequel had a horrendous start, with massive performance issues, buggy pathfinding and not enough improvements over the old game. This brings me to the next section, where I will analyse the generational improvements of roads and the surrounding building zones in city builders.

### Generational improvements and Limitations

{{< theme-image src="roads.svg" alt="Description" title=" Left: Simcity 3000. Center: Simcity 4. Right: Cities Skylines" >}}

The above image captures the changes between games quite well. SimCity 3000 has the least amount of freedom, roads are limited to 90-degree junctions, with zones placed as squares around the roads.

Moving to SimCity 4, roads can be diagonal but are limited to 45 degrees. Although Buildings can be placed around the diagonals, they are limited in orientation which leads to repetitive blocks.

In Cities Skylines, roads can finally be curved, with zones placed tangentially along the curve. Cities Skylines II, doesn’t change this formula, mainly focusing on improvements in road building tools.

While the road systems have evolved over time to mirror real-life quite closely, zoning around them has not seen the same improvements. Plots don’t hug the road and building assets remain on a static square grid tangential with the road. This leads to empty gaps between plots, especially at angled junctions where you would normally see corner buildings in real life.

### What I want to achieve

It is clear that a more realistic zoning system can be achieved. The below image shows a curved street where plots take up all available space. This is made possible by allowing for various shaped plots, procedural asset generation that follow road curves (fences, bushes, parking, sheds, yards) and assets that naturally join at plot edges on curved roads.

{{< theme-image src="roads-goal.svg" alt="Description" title="Goal layout" width="33%">}}

To make this improved vision come to life, I plan on using [Bevy](https://bevy.org). It is a rust based open-source data-driven game engine which will let me implement a lot of the systems with relative ease while still learning about game programming and rust.

The book Game Programming Patterns, by Robert Nystrom, will be another useful resource since it explains many performance-based concepts that are not applicable in my day-to-day job. Hopefully this combination will let me explore both a new programming language to me, and game programming!

To start we’ll need to implement some kind of road generation system, focusing on simpler polyline roads and then move onto curved variants. See you soon!
