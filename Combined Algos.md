Excellent 👌 — you spotted something important: **Dijkstra and A\*** are really the *same algorithm* — the only difference is whether you use a heuristic (`h(n)`).

Let’s unify them into one function.

---

## 1. Shared Idea

* **Dijkstra**:
  `priority = g(n)` → cost so far.
* **A\***:
  `priority = g(n) + h(n)` → cost so far + heuristic to goal.
* If you set `h(n) = 0`, A\* reduces to Dijkstra.

So we can implement one generic **best-first search** function.

---

## 2. Unified Search Function

Here’s Rust-style pseudocode:

```rust
use std::collections::{HashMap, BinaryHeap};
use std::cmp::Reverse;

fn best_first_search<F>(
    map: &Map,
    unit: &Unit,
    start: Position,
    goal: Option<Position>,  // None = full range (Dijkstra), Some = target (A*)
    heuristic: F,            // h(n), usually manhattan distance
) -> (HashMap<Position, i32>, HashMap<Position, Position>)
where
    F: Fn(Position, Position) -> i32,
{
    let mut came_from: HashMap<Position, Position> = HashMap::new();
    let mut cost_so_far: HashMap<Position, i32> = HashMap::new();
    let mut pq = BinaryHeap::new();

    pq.push((Reverse(0), start));
    cost_so_far.insert(start, 0);

    while let Some((Reverse(_), pos)) = pq.pop() {
        // If we have a goal and reached it → stop early (A*)
        if let Some(goal_pos) = goal {
            if pos == goal_pos {
                break;
            }
        }

        for n in map.neighbors(pos) {
            if let Some(step_cost) = map.movement_cost(n, unit) {
                let new_cost = cost_so_far[&pos] + step_cost;
                if new_cost <= unit.movement &&
                   new_cost < *cost_so_far.get(&n).unwrap_or(&i32::MAX) {
                    cost_so_far.insert(n, new_cost);

                    let priority = if let Some(goal_pos) = goal {
                        new_cost + heuristic(n, goal_pos) // A*
                    } else {
                        new_cost // Dijkstra
                    };

                    pq.push((Reverse(priority), n));
                    came_from.insert(n, pos);
                }
            }
        }
    }

    (cost_so_far, came_from)
}
```

---

## 3. Usage Examples

### Movement Range (Dijkstra)

```rust
let (costs, _) = best_first_search(
    &map,
    &unit,
    unit.position,
    None,                       // no goal
    |_, _| 0                    // h(n) = 0
);

let reachable: Vec<_> = costs
    .iter()
    .filter(|(_, &c)| c <= unit.movement)
    .map(|(pos, _)| *pos)
    .collect();
```

### Path to a Tile (A\*)

```rust
let (costs, came_from) = best_first_search(
    &map,
    &unit,
    unit.position,
    Some(target_pos),
    manhattan                 // h(n)
);

if costs.contains_key(&target_pos) {
    // reconstruct path
    let mut path = vec![target_pos];
    let mut current = target_pos;
    while let Some(prev) = came_from.get(&current) {
        current = *prev;
        path.push(current);
    }
    path.reverse();
    println!("Path = {:?}", path);
}
```

---

## 4. Why This is Nice

* One implementation covers both Dijkstra and A\*.
* Heuristic is pluggable (Manhattan, Euclidean, or `0`).
* Works for:

  * **Movement range overlays** (explore all reachable tiles).
  * **Targeted pathfinding** (fastest path to one goal).
  * Even AI behaviors that need custom heuristics.

---

✅ With this, you now have:

* BFS (for simple cases).
* One unified **best-first search** for Dijkstra + A\*.

---

Would you like me to also show how to extend this unified function to **return multiple goals efficiently** (e.g., AI wants the nearest enemy out of many)? That’s another common pattern.
