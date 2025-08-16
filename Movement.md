Great question — you’re essentially describing a **grid-based movement system with terrain costs, unit constraints, and faction rules** (very Fire Emblem / Advance Wars style). Let’s break it down.

---

## 1. Movement Calculation Model

The problem you’re describing is a **shortest-path with constraints** problem, very similar to Dijkstra’s algorithm (or A\* if you add targeting).

* Each **tile** has a terrain type (`Ground`, `Forest`, `Mountain`, etc.).
* Each **unit** has:

  * A movement allowance (integer).
  * A movement type (`Infantry`, `Cavalry`, `Flying`).
  * A faction (`Player`, `Enemy`, `Neutral`).

So the reachable area is basically “all tiles that can be reached with ≤ movement points remaining, given the rules above.”

---

## 2. Algorithm

For each unit **on its turn**, you’d run a **graph traversal (Dijkstra/BFS variant)**:

1. Start at the unit’s current tile with `movement_left = unit.movement`.

2. Use a priority queue (like Dijkstra) or a BFS if all costs are 1 (but Cavalry forests break that).

3. For each adjacent tile:

   * Check **terrain restrictions**:

     * Infantry blocked by mountains.
     * Cavalry blocked by mountains, forest costs +1.
     * Flying ignores restrictions.
   * Check **faction rules**:

     * If tile is occupied:

       * If it’s own faction → can pass but **not stop**.
       * If it’s neutral → can pass but **not stop**.
       * If it’s enemy and the moving unit is `Player` (or vice versa) → blocked completely.
   * If the move is valid and movement\_left ≥ cost → add to frontier with updated movement\_left.

4. Continue until no more tiles can be reached.

The result is a **set of reachable tiles** for that unit.

---

## 3. Should moves be precomputed or calculated on demand?

* **On demand (per turn)** is usually best:

  * Movement depends on **current unit positions** (blocking changes turn to turn).
  * Terrain doesn’t change often, but unit positions and factions do.
  * Precomputing for all units every turn is wasteful — most of the time, only the active unit matters.

So the map stores **static info** (terrain, occupancy), and the **movement system** computes reachable tiles per unit when needed.

---

## 4. Implementation Sketch (Rust-ish)

```rust
fn calculate_reachable_tiles(
    map: &Map,
    unit: &Unit,
    units: &Vec<Unit>,
) -> HashSet<Position> {
    let mut reachable = HashSet::new();
    let mut pq = BinaryHeap::new(); // (remaining_move, position)

    pq.push((unit.movement, unit.position));

    while let Some((moves_left, pos)) = pq.pop() {
        if moves_left < 0 { continue; }
        if !reachable.insert(pos) { continue; } // already visited

        for dir in DIRECTIONS {
            let new_pos = pos + dir;
            if let Some(tile) = map.get(new_pos) {
                let cost = movement_cost(tile, unit.movement_type);
                if cost <= moves_left && can_pass_through(new_pos, unit, units) {
                    pq.push((moves_left - cost, new_pos));
                }
            }
        }
    }

    reachable
}
```

Where:

* `movement_cost(tile, movement_type)` encodes terrain restrictions/penalties.
* `can_pass_through(pos, unit, units)` encodes faction/occupancy rules.

---

## 5. Key Design Takeaway

* **Map**: stores terrain + which unit (if any) is on each tile.
* **Unit**: stores movement + movement type + faction.
* **Movement system**: calculates reachable tiles *on demand* using BFS/Dijkstra.

---

👉 This keeps the **map lightweight and general**, and the **movement logic centralized and flexible**. You don’t “pre-store” moves in the map, you just compute when a unit is selected to move.

---

Would you like me to show you how to extend this so you can also calculate **attack ranges** (after movement, factoring weapon range)? That usually goes hand-in-hand with movement grids.
