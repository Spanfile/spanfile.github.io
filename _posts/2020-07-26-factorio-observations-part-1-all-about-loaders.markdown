---
layout: post
title: 'Factorio observations part 1: All about loaders'
featured: true
date: '2020-07-26 09:55:49'
tags:
- factorio
redirect_from: /factorio-observations-part-1-all-about-loaders
---

This post is part of an ongoing series about designs I've come up with and commonly use to do things in a heavily modded Factorio playthrough. These posts are kind-of living documents as well, which I update with corrections and observations as time goes along.

I am using the wonderful [Loaders Redux](https://mods.factorio.com/mod/LoaderRedux) mod by Optera ([Github](https://github.com/Yousei9/Loader-Redux)), which adds these devices to load/unload containers, machines and trains as fast as a belt will allow. They're tiered the same way as belts are, so there will always be a loader available to keep a belt saturated or consume a saturated belt.

![Loaders, love 'em.](/assets/2020/07/loaders.png)

They're like super fucking fast inserters, ok? How many inserters do you need to saturate a belt? Don't even care, I got loaders.

These things from _Loaders Redux_ are 2x1 entities that consume no power; IMO a bit overpowered but if a power requirement was added it'd be fine. Tradeoffs, tradeoffs. But I digress. Here's how I use 'em.

## Loading and unloading machines

Loaders, (underground) belts and splitters can be woven together quite neatly to create easily stacked designs to load and unload different machines depending on the size of the machine and the amount of inputs and outputs required. These are the designs I commonly use.

A 3x3 machine, most common one probably being the assembling machine. Other sizes of machine are available. The principles behind the designs don't change with smaller or larger machines (you'll just have an easier time weaving together larger machines since there's so much more surface area to stick belts into). Anyways here's two variations for a simple one-input one-output assembling machine.

This design has the machines stacked, separated by one tile. The entire design is 9 tiles wide. Input on the left, output on the right. One belt - or a pair of belts - feeding into the input splitter will feed as many machines as the belt will allow. The same thing happens on the output side as well, except to the other direction of course.

![One input, one output. Taller variation.](/assets/2020/07/1-1.png)

Alternatively, trading horizontal space for vertical space savings, the machines can be stacked touching each other and instead use 11 tiles horizontally. Same principles still apply.

![w i d e](/assets/2020/07/2.png)

* * *

Okay so how about more than one belt as input? Two belts going in is still simple, requiring only belts and splitters (and the loaders of course). Again, two different designs, although this time with a different tradeoff.

Both of these designs build upon the 9-wide one above with just the one input. They both start off by nudging the machines one more tile apart to make space for the new loader. The difference between the two is where the new input comes in from. First design; both inputs from the same direction.

![These two just can't ever get together](/assets/2020/07/3.png)

The newly added belt is just like in the 11-wide design seen earlier, now essentially combining both designs. The design is taller and wider, but it has two belts feeding each machine. As usual, the output can be anything since the input side could just be mirrored to the output to have as many outputs as needed.

_But wait_, we could nudge the machines even further apart to save a tile of horizontal space. Now there's an entire machine's worth of empty space between the two, but the input column is still just four tiles wide.

![Even further apart, even more compact](/assets/2020/07/7.png)

_And that's not all_, the new input belt can be turned around to instead come in from the opposite side to the already existing input, which also results in the input column being four tiles wide, and it doesn't require the machines to be so far apart. It also entirely fills the empty gaps left behind which is _so neat_.

![just look at how tightly packet that input side is](/assets/2020/07/4.png)

This of course means the inputs are on the opposite sides to the machines, which may cause issues with keeping the machines fed. Note that it's possible to run an underground belt through the rightmost tile column of the input side to get the second input to the other side from the same side as the first input, which would fill the input column to the _max_.

* * *

How about three input belts? At this point - and honestly even earlier - you should consider combining two recipe inputs into a single belt (you got two halves after all). It's likely if the recipe calls for three or more input materials, it's gonna take enough time to complete it doesn't matter if the inputs are only half of a full belt. But I'm sure you can weave in three inputs (why not put one input on the same side as the output?)

* * *

Okay, what about liquids? Luckily pipes are way more flexible than belts (even if they're made of solid metal), so their placement can be dictated more easily. The pipe input is simply a line that runs alongside the machines, and the machines tap into it however. And since the one-input-belt design from earlier leaves a nice little gap to fit a pipe into, it all comes together _so neat_.

![The pipe fits right in](/assets/2020/07/5.png)

Could even save a couple underground pipes and have two adjacent machines use the same pipe as input.

* * *

The ideas behind these designs, once figured out, can be exploited to the extreme to fit machines together as close as possible while still weaving in multiple inputs and outputs, both belt and liquid. This is an example of a design for the 5x5 Angel's Floatation cell that has an input of one belt and liquid, and an output of two belts and a liquid. The design actually has two columns of the machines facing "away" from each other with both inputs and half of the output on the inside of the machine columns, and the rest on the outside.

![It took me a long time to figure this one out](/assets/2020/07/6.png)

So that's machines, what about containers?

## Big-ass containers

Crates are kinda boring. They're small, both in terms of physical and storage, and I'm a big boy so I want something bigger. Angel's got my back again with [warehouses](https://mods.factorio.com/mod/angelsaddons-warehouses) and [silos](https://mods.factorio.com/mod/angelsaddons-oresilos), these behemoths of containers that fit tens upon tens of thousands of items, and take a considerable amount of space (_and resources to craft_). They're great for storing a **lot** of something, or a decent amount of many things. They're the perfect match for loaders.

I'm sure you understand how loaders and containers interact, but the earlier point about storing multiple things has one caveat: one item might fill up the warehouse/silo so much other things can't fit anymore, and we don't want that. If your instinct tells you the circuit network can help with that, you're right!

Except there's another caveat. Loaders can't interact with the circuit network directly, so in order to control them letting stuff in or out, we have to control the belt directly next to them.

![yain't gettin' in](/assets/2020/07/10.png)

Problem solved!

Except there's _one more caveat_: the loaders have an "internal buffer" that might mess things up. Since the loader is like a two-tile long piece of belt, cutting off its input can still leave some items inside it and they'll end up in the container even after the cut. It's not in any way consistent how many items there might be inside the loader at any time, but we can be sure it's never more than four. Just something to keep in mind if you ever find yourself caring about the _exact_ amount of things inside something.

## Trains!

I know I love trains, and probably so do you. Unfortunately I'm not gonna talk about trains here, but instead in another post I haven't actually written yet all about station designs ... so stay tuned?

