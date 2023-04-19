+++
title = "fantasia - Drawing Lines"
date = "2023-04-18T17:39:42+02:00"
tags = ["fantasia"]
+++

Now, on to something more interesting: drawing stuff.
We need to figure out the simple things first, 
so we'll start with the simplest thing you can draw (except for a point, obviously): a line segment.

A line segment needs a starting point and an ending point. Let's first define a `Point` struct:
```rust
pub struct Point {
    pub x: f32,
    pub y: f32,
    pub z: f32,
}
```
> Note: If your point's `x` and `y` are actual "screen" coordinates (i.e. a specific position in the image buffer),
then if you change the width or height of the output, you'll also need to manually update the coordinates.
To avoid this, I'll be using "world" coordinates instead. This means I'll represent every component as an `f32`,
describing the point's position relative to the imaginary center of the "world".
For now, that also means the center of the resulting image.

Here are some basic implementations for the `Point` struct to make our life easier:
```rust
impl Point {
    pub fn new(x: f32, y: f32, z: f32) -> Self {
        Self { x, y, z }
    }

    pub fn magnitude(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2) + self.z.powi(2)).sqrt()
    }
}
impl Add for Point {
    type Output = Self;
    fn add(mut self, rhs: Self) -> Self::Output {
        self.x += rhs.x;
        self.y += rhs.y;
        self.z += rhs.z;
        self
    }
}
impl Sub for Point {
    type Output = Self;
    fn sub(mut self, rhs: Self) -> Self::Output {
        self.x -= rhs.x;
        self.y -= rhs.y;
        self.z -= rhs.z;
        self
    }
}
/// Scalar Multiplication
impl Mul<f32> for Point {
    type Output = Self;
    fn mul(mut self, rhs: f32) -> Self::Output {
        self.x *= rhs;
        self.y *= rhs;
        self.z *= rhs;
        self
    }
}
impl Mul<Point> for f32 {
    type Output = Point;
    fn mul(self, rhs: Point) -> Self::Output {
        rhs * self
    }
}
```

The actual algorithm
--------------------
Let's think about our line segment as a difference of 2 vectors: `A` and `B`.
{{< figure src="/fantasia-line-algorithm-vectors.svg" caption="representation of A, B, and their difference" >}}

You can get any point on a vector's trajectory by multiplying it with a scalar, let's call it `t`.
In the above figure you can clearly see exactly that.

If we limit the scalars to hundreths from 0 to 1, we can reasonably loop through every describable point in our line segment.
We won't get *every* point, sure, but it should be good enough.

Rust doesn't allow floating point ranges, so we have to do some redefining in our implementation:
```rust
impl Renderer {
    // ...
    pub fn line(&mut self, color: Rgba, from: Point, to: Point) {
        for t in 0..100 {
            let t = t as f32 / 100.;
            let current = t * (to - from);

            // TODO: put pixel
        }
    }
}
```
Now, while we could just write the code for actually putting the pixel on the screen/image,
it's better if we just turn that into its own method to avoid code duplication:
```rust
impl Renderer {
    // ...
    pub fn to_screen_coords(&self, point: Point) -> (usize, usize) {
        // the `+ 1.` is here to offset coordinates from the center to the top left corner
        (
            ((point.x + 1.) * self.width as f32 / 2.) as usize,
            ((point.y + 1.) * self.height as f32 / 2.) as usize,
        )
    }

    pub fn put_pixel(&mut self, coords: (usize, usize), color: Rgba) {
        let idx = coords.0 + coords.1 * self.width;
        self.buffer[idx] = color.r;
        self.buffer[idx+1] = color.g;
        self.buffer[idx+2] = color.b;
        self.buffer[idx+3] = color.a;
    }
}
```
And there we have it: a `to_screen_coords` method for converting our "world" coordinates into "screen" coordinates,
and a `put_pixel` method.

We can also make our `put_pixel` implementation smaller by using iterators again:
```rust
// ...
pub fn put_pixel(&mut self, coords: (usize, usize), color: Rgba) {
    let idx = coords.0 + coords.1 * self.width;
    self.buffer
        .splice(idx..idx + 4, color.bytes().iter().cloned());
}
// ...
```

Now, to finish `line`:
```rust
// ...
pub fn line(/* ... */) {
    for t in 0..100 {
        // ...
        self.put_pixel(self.to_screen_coords(current), color);
    }
}
// ...
```

Results
-------

Let's draw 3 lines in `main.rs` to form a triangle:
```rust
fn main() {
    // ...
    let a = Point::new(0., 1., 0.);
    let b = Point::new(-1., 0., 0.);
    let c = Point::new(1., 0., 0.);

    renderer.line(Rgba::new(255, 255, 255, 255), a, b);
    renderer.line(Rgba::new(255, 255, 255, 255), b, c);
    renderer.line(Rgba::new(255, 255, 255, 255), c, a);
    // ...
}
```

And now if we run it...

![](/fantasia-line-output-uh-oh.png)

*oh no...*

> Why did this happen? It's because we got something *slightly* wrong in our algorithm's implementation.
We assumed the point `t(B-A)` to always be on the desired line, but it's only guaranteed to be on the desired *vector*.
Why is that a problem? Well, the origin of a vector isn't set in stone. 
In our example, I set it to the end of `A`, but the renderer sets the left corner of the screen as the origin point.

But we can fix this!

By adding `t(B-A)` to `A` like in the figure below, we can find where the point actually needs to be.

{{< figure src="/fantasia-line-algorithm-vectors2.svg" caption="the vector C points to the correct point" >}}

Fixing it should be as simple as this:
```rust
// ...
pub fn line(/* ... */) {
    // ...
    let current = from + t * (to - from);
    // ...
}
```

And the result is...
![yet another failure](/fantasia-line-output-uh-oh2.png)

> **Author's note**: I initially thought of cutting the failures out of the guide. 
But I've now decided against doing that. Failing is an important part of understanding how something works,
by eliminating some misconceptions. I also want this to stay here as a reminder, for my readers, but also for myself,
that failing is normal and it's just part of the process. The way I figured the solution out was by just taking
breaks and coming back with a clearer mind.

Now, what's causing this? It turns out, this algorithm *really* doesn't like it when `t(B-A)` is negative.
It might be because of the conversion to screen coordinates, or because of the formula we're using to calculate `current`,
I'm not sure. But the way I fixed it was by just ensuring we're always drawing from the point closer to the top left corner,
towards the one farther away.

And here's an implementation:
```rust
pub fn line(/* ... */) {
    let mut from = self.to_screen_coords(from);
    let mut to = self.to_screen_coords(to);
    if to.magnitude() < from.magnitude() {
        std::mem::swap(&mut from, &mut to);
    }
    
    for t in t..100 {
        // ...
        self.put_pixel(current, color);
    }
}
```

Since we're already using screen coordinates to determine from where we should actually start drawing from,
I've decided to just redefine `to` and `from` accordingly, then swap them around if `to` turns out to be closer
to the top left corner.

And now, *finally*...
-------------------
![](/fantasia-line-output-success.png)

You did it! You've just a the traditional first triangle!
