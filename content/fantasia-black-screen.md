+++
title = "fantasia - Black Screen"
date = "2023-04-17T22:41:49+02:00"
tags = ["fantasia"]
+++

Time to actually start coding!
First, we need to decide how we're gonna structure it. I'll go for having a struct called `Renderer`,
so that we can save stuff as fields instead of passing the same repetitive arguments to our functions.
This struct will also contain every useful method that depends on the state.

As such, we'll start with the definition in `src/lib.rs`:
```rust
#[derive(Clone)]
pub struct Renderer {
    buffer: Vec<u8>,
    width: usize,
    height: usize,
}
```
Fields:
- `buffer`: this will hold all of our pixel data in a contiguous `Vec` of bytes.
- `width`: the width of the image we wish to write to.
- `height`: the height of the image we wish to write to.

Next, some simple methods:
```rust
impl Renderer {
    pub fn new(width: usize, height: usize) -> Self {
        Self {
            buffer: vec![0; width * height * 4],
            width,
            height,
        }
    }
}
```
#### Why implement it that way?
The buffer needs to already have space ready for every pixel in the image.

`width * height` is a well-known way to calculate the total length of a 2D list, 
but where does the `* 4` come from?

Well, for I've decided to use the RGBA format for this. RGBA stores its data across 4 bytes:
one for red, one for blue, one for green and one for *alpha*. Alpha simply represents opacity.


Clearing the screen
-------------------
The next thing we need to take care of is essentially "setting the background".
To do that, we just set the entire buffer to a given RGBA color.
```rust
impl Renderer {
    // ...
    pub fn clear(&mut self, color: Rgba) {
        todo!()
    }
}
```

And this is what our function should look like. But before we actually, implement it, let's actually
define and implement the `Rgba` type that we'll use to represent colors.
```rust
#[derive(Clone, Copy, PartialEq, Eq)]
pub struct Rgba {
    r: u8,
    g: u8,
    b: u8,
    a: u8,
}
impl Rgba {
    pub fn new(r: u8, g: u8, b: u8, a: u8) -> Self {
        Self { r, g, b, a}
    }

    pub fn bytes(&self) -> [u8; 4] {
        [self.a, self.b, self.c, self.a]
    }
}
```
Simple and easy~.

Now, we can return to `clear`. We can implement it in a simple, imperative way as such:
```rust
// ...
pub fn clear(&mut self, color: Rgba) {
    for i in 0..(self.buffer.len() / 4) {
        let i = i * 4;
        self.buffer[i] = color.r;
        self.buffer[i+1] = color.g;
        self.buffer[i+2] = color.b;
        self.buffer[i+3] = color.a;
    }
}
// ...
```
And while this works, it's quite ugly. I'll use the following implementation which uses one of my favorite
features of the Rust language: iterators.
```rust
// ...
pub fn clear(&mut self, color: Rgba) {
    self.buffer = std::iter::repeat(color)
        .flat_map(|color| color.bytes())
        .take(self.buffer.len())
        .collect();
}
// ...
```
If you understand this, that's good! But don't worry if you don't, just use the approach you understand better.
Debugging code you don't fully understand is a recipe for disaster, after all.

Time to see the results
-----------------------
For the purpose of testing our renderer out, let's create a `src/main.rs` file with the following contents:
```rust
use std::{fs::File, io::BufWriter};
use image::{codecs::png::PngEncoder, ImageEncoder};

use fantasia::*;

const WIDTH: usize = 400;
const HEIGHT: usize = 400;

fn main() {
    // Step 1
    let mut renderer = Renderer::new(WIDTH, HEIGHT);
    renderer.clear(Rgba::new(0, 0, 0, 255)); // the color black with 100% opacity

    // Step 2
    let buf = image::RgbaImage::from_raw(WIDTH as u32, HEIGHT as u32, renderer.buffer().to_vec())
        .unwrap();

    // Step 3
    let writer = BufWriter::new(File::create("output.png").unwrap());
    let encoder = PngEncoder::new(writer);

    encoder
        .write_image(&buf, WIDTH as u32, HEIGHT as u32, image::ColorType::Rgba8)
        .unwrap();
}
```

Steps:
1. Create a renderer and clear the screen.
2. Convert the raw pixel data into something the `image` crate understands.
3. Create a new `output.png` file and write the resulting image there.

And this is what we get:
{{< figure src="/output.png" caption="the output, 400x400 black PNG image" >}}
