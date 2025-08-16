Perfect, let’s extend the movement logic to also cover **attack ranges** — this is a very common requirement in tactics games (think Fire Emblem, Advance Wars, etc.).

---

## 1. Movement + Attack = Two-Step Problem

When a unit takes its turn, you usually want to show:

1. **Movement range** → all tiles the unit can move to (what we already discussed).
2. **Attack range** → all tiles the unit could target with a weapon, after moving.

So the flow is:

* Step 1: Compute **reachable tiles** via movement traversal.
* Step 2: From each reachable tile, add the unit’s **weapon range** overlay.

---

## 2. Attack Range Rules

* Each weapon has a **minimum range** and **maximum range** (e.g., swords `1`, bows `2–3`).
* Ranges are typically measured in **Manhattan distance** (not diagonals unless your game allows it).
* Usually, units can **attack enemies only**, not empty tiles — but highlighting empty tiles helps the player see their threat zone.

---

## 3. Implementation Sketch

Let’s say each `Unit` has a weapon with:

```rust
struct Weapon {
    min_range: u32,
    max_range: u32,
}
```

Then:

```rust
fn calculate_attackable_tiles(
    map: &Map,
    unit: &Unit,
    units: &Vec<Unit>,
) -> HashSet<Position> {
    let mut attackable = HashSet::new();

    // Step 1: movement range
    let reachable = calculate_reachable_tiles(map, unit, units);

    // Step 2: expand each position by weapon range
    for pos in reachable {
        for dx in -(unit.weapon.max_range as i32)..=(unit.weapon.max_range as i32) {
            for dy in -(unit.weapon.max_range as i32)..=(unit.weapon.max_range as i32) {
                let dist = dx.abs() + dy.abs(); // Manhattan distance
                if dist >= unit.weapon.min_range as i32
                    && dist <= unit.weapon.max_range as i32
                {
                    let target_pos = Position { x: pos.x + dx, y: pos.y + dy };
                    if map.in_bounds(target_pos) {
                        attackable.insert(target_pos);
                    }
                }
            }
        }
    }

    attackable
}
```

---

## 4. Occupancy Rules for Attacks

Depending on your design:

* Units can **attack through allies** (common in Fire Emblem).
* Sometimes terrain blocks attacks (walls, mountains). If you want line-of-sight rules, you’ll need a **raycast check** between the attacker and target.
* Highlighting can include all attackable tiles, but when actually attacking, filter by:

  * Target must be an **enemy unit**.
  * Must be within attack range.
  * Must not violate line-of-sight (if you use that rule).

---

## 5. Visualization Example

Imagine a **5 move Infantry with a bow (range 2–3)**:

* Step 1: Movement range highlights all ground tiles reachable within 5 moves.
* Step 2: From each reachable tile, overlay all positions at Manhattan distance 2 or 3.
* Result: You now have an **attack grid** showing exactly where the unit threatens after moving.

---

✅ So now you have:

* `calculate_reachable_tiles()` for movement.
* `calculate_attackable_tiles()` for movement + weapon threat.

---

Do you want me to also show how to **merge all units’ attack ranges into a global “danger map”** (so the player can see which tiles are threatened by enemies)? That’s another common feature.
