Right — good catch again ⚡.
If you just “let everyone draw,” you’ll get **multiple previews** when there are multiple unit-holding states (e.g. `PlayerMove` still hanging around under `PlayerAction`).

So we need something a bit stricter:

---

## 🔑 Rule

At any time, **only the *topmost state* that owns a scratch unit** should render that unit.
All states above it (menus, targeting overlays, etc.) may add UI, but they shouldn’t re-draw the unit.
All states below it are suppressed for that unit.

---

## 🛠 Implementation Idea

Add a method to your `GameState` trait:

```rust
trait GameState {
    // …
    fn scratch_unit(&self) -> Option<&Unit> { None }
}
```

Then in your renderer:

```rust
impl StateMachine {
    pub fn draw(&mut self, world: &World) {
        // 1. Find the topmost state with a scratch unit
        let mut scratch: Option<&Unit> = None;
        for state in self.stack.iter().rev() {
            if let Some(u) = state.scratch_unit() {
                scratch = Some(u);
                break;
            }
        }

        // 2. Draw authoritative units, except the overridden one
        for unit in world.units.values() {
            if scratch.map(|su| su.id == unit.id).unwrap_or(false) {
                continue;
            }
            draw_circle(unit.pos.0 as f32 * 32.0 + 16.0,
                        unit.pos.1 as f32 * 32.0 + 16.0,
                        12.0, WHITE);
        }

        // 3. Draw the scratch unit (if any)
        if let Some(unit) = scratch {
            draw_circle(unit.pos.0 as f32 * 32.0 + 16.0,
                        unit.pos.1 as f32 * 32.0 + 16.0,
                        12.0, YELLOW);
        }

        // 4. Let *all* states draw world overlays (like range highlights, cursors)
        for state in &self.stack {
            state.draw_world_overlay(world);
        }

        // 5. Let only the *top* state draw UI
        if let Some(top) = self.stack.last() {
            top.draw_ui(&mut self.msg_queue);
        }
    }
}
```

---

## 🔎 How this plays out

* If you’re in `PlayerMove`, that’s the top scratch owner → it’s drawn.
* If you push `PlayerAction`, its scratch copy **overrides** the one from `PlayerMove` → only that copy gets drawn.
* If you then push a `MenuState`, it has no scratch → the renderer still uses the `PlayerAction` copy.
* Once all scratch-owning states are popped, you fall back to the authoritative unit.

---

This way:
✔️ No duplicate previews.
✔️ Menus/targeting states don’t hide previews.
✔️ Exactly one unit copy is visible at a time — the one closest to the top.

---

Would you like me to also show how to make **cursors render only on player-controlled states** in this same layering system?
