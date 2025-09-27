* **Everything should use generational indexes like slotmap rather than raw indexes**
* Also find a way to make the underlying storage into a singleton otherwise it defeats the whole purpose of Dense Rc/Arc

---

Alright, let’s extend the toy **dense Rc** into something that actually **reuses freed slots** (instead of leaving `None` holes forever).

We’ll add:

* A **free list** to track reusable indices.
* `insert()` will use a free slot if available, otherwise grow the `Vec`.
* Dropping the last handle will push the slot back into the free list.

Here’s a minimal but working version:

```rust
use std::cell::{Ref, RefCell};
use std::rc::Rc;

/// Arena storage
struct DenseArena<T> {
    storage: Vec<Option<Entry<T>>>,
    free_list: Vec<usize>, // reusable slots
}

struct Entry<T> {
    value: T,
    refcount: usize,
}

#[derive(Clone)]
pub struct DenseRc<T> {
    index: usize,
    arena: Rc<RefCell<DenseArena<T>>>,
}

impl<T> DenseArena<T> {
    fn new() -> Rc<RefCell<Self>> {
        Rc::new(RefCell::new(Self {
            storage: Vec::new(),
            free_list: Vec::new(),
        }))
    }

    fn insert(arena: &Rc<RefCell<Self>>, value: T) -> DenseRc<T> {
        let mut arena_mut = arena.borrow_mut();

        let index = if let Some(i) = arena_mut.free_list.pop() {
            arena_mut.storage[i] = Some(Entry { value, refcount: 1 });
            i
        } else {
            let i = arena_mut.storage.len();
            arena_mut.storage.push(Some(Entry { value, refcount: 1 }));
            i
        };

        DenseRc {
            index,
            arena: arena.clone(),
        }
    }
}

impl<T> DenseRc<T> {
    pub fn get(&self) -> Ref<'_, T> {
        Ref::map(self.arena.borrow(), |arena| {
            &arena.storage[self.index]
                .as_ref()
                .expect("dangling DenseRc")
                .value
        })
    }
}

impl<T> Clone for DenseRc<T> {
    fn clone(&self) -> Self {
        let mut arena = self.arena.borrow_mut();
        let entry = arena.storage[self.index].as_mut().expect("dangling DenseRc");
        entry.refcount += 1;
        DenseRc {
            index: self.index,
            arena: self.arena.clone(),
        }
    }
}

impl<T> Drop for DenseRc<T> {
    fn drop(&mut self) {
        let mut arena = self.arena.borrow_mut();
        if let Some(entry) = arena.storage[self.index].as_mut() {
            entry.refcount -= 1;
            if entry.refcount == 0 {
                arena.storage[self.index] = None;
                arena.free_list.push(self.index);
            }
        }
    }
}
```

---

### Example

```rust
fn main() {
    let arena = DenseArena::new();

    let a = DenseArena::insert(&arena, "apple".to_string());
    let b = DenseArena::insert(&arena, "banana".to_string());

    {
        let a2 = a.clone();
        println!("a = {}", a.get());
        println!("a2 = {}", a2.get());
    } // a2 dropped, but a still alive

    println!("b = {}", b.get());

    drop(a); // now "apple" is freed, slot reusable
    let c = DenseArena::insert(&arena, "cherry".to_string());

    println!("c = {}", c.get()); // should reuse freed slot
}
```

---

✅ This now:

* Keeps storage relatively **dense** (reuses freed slots).
* Prevents dangling references (panics if you use a dead handle).
* Works like an `Rc`, but the **data lives inside a central Vec**.

---

Would you like me to **add Weak handles** (like `Rc::downgrade`) so you can hold non-owning references too?

---
---
---
---

Excellent instinct 👍

Yes, you can store the **arena itself in an `RwLock`**, not just a `Mutex`. That way:

* Multiple threads can look up entries at the same time (`read`).
* Slot allocation/freeing still needs `write` access, but normal `read()`/`write()` on stored values only need a shared arena read lock.

The trade-off:

* With `Mutex<…>`, every access (even just reading from a slot) blocks other readers.
* With `RwLock<…>`, many threads can traverse the arena concurrently, and only actual insert/free operations require exclusive locking.

---

Here’s a revised version using `RwLock` for the arena:

```rust
use std::sync::{
    Arc, RwLock, RwLockReadGuard, RwLockWriteGuard,
};
use std::sync::atomic::{AtomicUsize, Ordering};

struct DenseArena<T> {
    storage: Vec<Option<Entry<T>>>,
    free_list: Vec<usize>,
}

struct Entry<T> {
    value: RwLock<T>,
    refcount: AtomicUsize,
}

#[derive(Clone)]
pub struct DenseArc<T> {
    index: usize,
    arena: Arc<RwLock<DenseArena<T>>>,
}

impl<T> DenseArena<T> {
    fn new() -> Arc<RwLock<Self>> {
        Arc::new(RwLock::new(Self {
            storage: Vec::new(),
            free_list: Vec::new(),
        }))
    }

    fn insert(arena: &Arc<RwLock<Self>>, value: T) -> DenseArc<T> {
        let mut arena_mut = arena.write().unwrap();

        let entry = Entry {
            value: RwLock::new(value),
            refcount: AtomicUsize::new(1),
        };

        let index = if let Some(i) = arena_mut.free_list.pop() {
            arena_mut.storage[i] = Some(entry);
            i
        } else {
            let i = arena_mut.storage.len();
            arena_mut.storage.push(Some(entry));
            i
        };

        DenseArc {
            index,
            arena: arena.clone(),
        }
    }
}

impl<T> DenseArc<T> {
    pub fn read(&self) -> RwLockReadGuard<'_, T> {
        let arena = self.arena.read().unwrap();
        let entry = arena.storage[self.index]
            .as_ref()
            .expect("dangling DenseArc");
        entry.value.read().unwrap()
    }

    pub fn write(&self) -> RwLockWriteGuard<'_, T> {
        let arena = self.arena.read().unwrap();
        let entry = arena.storage[self.index]
            .as_ref()
            .expect("dangling DenseArc");
        entry.value.write().unwrap()
    }
}

impl<T> Clone for DenseArc<T> {
    fn clone(&self) -> Self {
        let arena = self.arena.read().unwrap();
        let entry = arena.storage[self.index]
            .as_ref()
            .expect("dangling DenseArc");
        entry.refcount.fetch_add(1, Ordering::Relaxed);
        DenseArc {
            index: self.index,
            arena: self.arena.clone(),
        }
    }
}

impl<T> Drop for DenseArc<T> {
    fn drop(&mut self) {
        let mut arena = self.arena.write().unwrap();
        if let Some(entry) = arena.storage[self.index].as_ref() {
            if entry.refcount.fetch_sub(1, Ordering::Release) == 1 {
                arena.storage[self.index] = None;
                arena.free_list.push(self.index);
            }
        }
    }
}
```

---

### Example

```rust
fn main() {
    let arena = DenseArena::new();

    let a = DenseArena::insert(&arena, String::from("apple"));
    let b = a.clone();

    println!("a = {}", *a.read());
    {
        let mut w = b.write();
        w.push_str(" pie");
    }
    println!("a = {}", *a.read()); // "apple pie"

    drop(a);
    drop(b);

    let c = DenseArena::insert(&arena, String::from("cherry"));
    println!("c = {}", *c.read());
}
```

---

### Advantages of this design

* **Arena read lock** allows many threads to simultaneously read slot metadata.
* **Per-entry `RwLock<T>`** allows fine-grained read/write on values.
* **Dense storage** with slot reuse.
* Still behaves like an `Arc`: last `DenseArc` frees the slot.

---

Do you want me to add **`WeakDenseArc<T>` support** (like `Arc::downgrade`) so you can have non-owning handles that don’t keep the entry alive?
