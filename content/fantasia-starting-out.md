+++
title = "fantasia - Starting Out"
date = "2023-04-17T20:32:13+02:00"
tags = ["fantasia"]
keywords = ["fantasia", "renderer", "guide", "starting", "setup"]
+++

First, let's define some goals:
1. API-agnostic. The renderer should be a library-type crate that simply provides the necessary
tools for rendering into an array of bytes representing pixels.
Being API-agnostic would make it very flexible and versatile, allowing its users to adapt
to their own specific use cases.
2. Capable of rendering 3D models, of course. That's the main point, after all.
3. As few dependencies as possible. We don't want to complicate ourselves or pick up too much stuff that we don't need.
The renderer should be pretty simple in its design and not bring in too many dependencies to whoever might decide to use 
it in their project.
4. Something you hopefully understand pretty well by the end of this guide. This is more of a goal for me, if anything.
I hope my way of explaining concepts through examples will work for you as well.

And now that that's out of the way...

### Setup:

First, let's set our crate up.
```bash
cargo new --lib fantasia
cd fantasia
```

For now, I think we should only have 1 dependency: 
the [image](https://crates.io/crates/image) crate.
```bash
cargo add image
```
This crate isn't actually needed for the renderer itself, but it provides a nice and simple way to test it.
It will be removed later on.
