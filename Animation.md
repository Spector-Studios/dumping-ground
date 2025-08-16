Ah, this is the classic **“separating game state from animations”** problem 🙂
You already have a good architecture:

* `Game` struct
* `update(&mut self, dt)` → runs logic per frame
* `render(&self)` → draws units based on their positions

That’s the right foundation. The trick is that your **unit’s position is discrete** (e.g. a tile grid), but for animations you want an **interpolated position** that smoothly moves from one tile to another over time.

---

## Step 1. Extend your `Unit` struct

Give each `Unit` both a *logical position* (the tile it’s on) and an *animation state* (optional):

```rust
#[derive(Debug)]
struct Unit {
    tile_pos: Point,        // the actual grid position (destination when settled)
    render_pos: (f32, f32), // smooth world position for rendering
    animation: Option<MoveAnimation>,
}

#[derive(Debug)]
struct MoveAnimation {
    start: (f32, f32),
    end: (f32, f32),
    progress: f32,   // 0.0 → 1.0
    duration: f32,   // seconds
}
```

---

## Step 2. Update the animation in `Game::update`

When in your `Animation` state, advance the progress based on `dt`:

```rust
impl Unit {
    fn update(&mut self, dt: f32) {
        if let Some(anim) = &mut self.animation {
            anim.progress += dt / anim.duration;

            if anim.progress >= 1.0 {
                // Snap to final position
                self.render_pos = anim.end;
                self.tile_pos = Point { 
                    x: anim.end.0 as i32, 
                    y: anim.end.1 as i32 
                };
                self.animation = None; // animation finished
            } else {
                // Interpolate (linear here, but you could use easing)
                let t = anim.progress;
                self.render_pos = (
                    anim.start.0 + (anim.end.0 - anim.start.0) * t,
                    anim.start.1 + (anim.end.1 - anim.start.1) * t,
                );
            }
        }
    }
}
```

---

## Step 3. Render from `render_pos`

In your `render` function:

```rust
impl Unit {
    fn render(&self, ctx: &mut RenderContext) {
        ctx.draw_sprite(self.render_pos.0, self.render_pos.1);
    }
}
```

---

## Step 4. Starting an animation

When you decide to move a unit, don’t immediately change its `tile_pos`. Instead, set up an animation:

```rust
unit.animation = Some(MoveAnimation {
    start: unit.render_pos,
    end: (dest.x as f32, dest.y as f32),
    progress: 0.0,
    duration: 0.25, // quarter-second move
});
```

---

## Step 5. Integrate with `Game` states

* `GameState::MoveUnit` → pick a destination
* switch to `GameState::Animation`
* while in animation, call `unit.update(dt)` each frame
* when all animations are finished, return to `GameState::SelectUnit`

---

✅ This way:

* **Logic (`tile_pos`)** always reflects the last *settled* position.
* **Render (`render_pos`)** is what gets drawn, interpolated during animations.
* When animation ends, `render_pos` = `tile_pos` again.

---

Would you like me to sketch a **full `GameState` enum + update loop** example showing how `Animation` integrates with your existing `SelectUnit` / `MoveUnit` states?
