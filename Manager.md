Great! Here's a full implementation of a **fast and idempotent `TextureManager`** in Rust, optimized for:

* **Fast `get(handle)`** using `Vec[usize]` access (zero overhead).
* **Idempotent `load(path)`** using a `HashMap<String, TextureHandle>`.
* **Cheaply clonable `TextureHandle`** (a simple `usize` index).
* **WASM-friendly**: avoids `Arc`/`Rc`, no atomics.

---

## ✅ Final Code: `TextureManager`

```rust
use std::collections::HashMap;
use std::path::Path;

/// Fake texture type for illustration. Replace with your real Texture2D.
#[derive(Debug)]
pub struct Texture2D {
    pub path: String,
    // Add your GPU data, dimensions, etc.
}

#[derive(Copy, Clone, Debug, PartialEq, Eq, Hash)]
pub struct TextureHandle(usize);

pub struct TextureManager {
    textures: Vec<Texture2D>,
    path_to_handle: HashMap<String, TextureHandle>,
}

impl TextureManager {
    pub fn new() -> Self {
        Self {
            textures: Vec::new(),
            path_to_handle: HashMap::new(),
        }
    }

    /// Idempotently loads a texture if not already loaded and returns a handle.
    pub fn load<P: AsRef<Path>>(&mut self, path: P) -> TextureHandle {
        let key = path.as_ref().to_string_lossy().to_string();

        if let Some(&handle) = self.path_to_handle.get(&key) {
            return handle;
        }

        let texture = Self::load_texture_from_disk(&key);
        let handle = TextureHandle(self.textures.len());

        self.textures.push(texture);
        self.path_to_handle.insert(key, handle);
        handle
    }

    /// Fast access to texture data using the handle (O(1) vec access).
    pub fn get(&self, handle: TextureHandle) -> &Texture2D {
        &self.textures[handle.0]
    }

    fn load_texture_from_disk(path: &str) -> Texture2D {
        println!("Loading texture: {}", path);
        Texture2D {
            path: path.to_string(),
        }
    }
}
```

---

## 🧪 Usage Example

```rust
fn main() {
    let mut manager = TextureManager::new();

    let handle1 = manager.load("assets/tex1.png");
    let handle2 = manager.load("assets/tex1.png"); // Won't reload

    assert_eq!(handle1, handle2); // Idempotent

    let tex = manager.get(handle1);
    println!("Using texture: {:?}", tex);
}
```

---

## ✅ Benefits of This Design

| Feature            | Implementation                                      |
| ------------------ | --------------------------------------------------- |
| Idempotent loading | `HashMap<String, TextureHandle>`                    |
| Fast lookup (get)  | `Vec[usize]` access                                 |
| Cheap handle clone | `Copy` `usize`                                      |
| WASM-friendly      | ✅ No `Arc`, `Rc`, or atomics                        |
| Thread-safety      | ❌ Not thread-safe (but simple to make with `Mutex`) |

---

## 🔧 Optional Extensions

* Add `unload(handle)` support by using a `SlotMap` or free list.
* Use `DashMap` if you need thread-safe access.
* Add support for reloading or hot-swapping assets.

---

Would you like a version of this that works with a specific graphics framework like `wgpu`, `macroquad`, or `bevy`?
