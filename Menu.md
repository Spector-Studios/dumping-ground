Yes 👍 — you definitely don’t want to hard-code a completely separate menu for “actions,” “pause,” and “main menu.”

The trick is to **separate the menu system (UI plumbing)** from **what the menu entries do (game logic)**.
That way, you can feed the same menu system with different entries in different states.

---

### Step 1: Define a generic menu item

You want something that can represent any kind of option in a UI menu.

```rust
/// What the menu entry represents
#[derive(Clone)]
pub struct MenuItem<T> {
    pub label: String,
    pub value: T,
}
```

* `T` is generic:

  * In a **pause menu**, `T` could be an enum `PauseAction` (`Resume`, `Quit`, …).
  * In a **player action menu**, `T` could be your `Action` enum (`Attack`, `Defend`, …).
  * In a **main menu**, `T` could be `MainMenuAction` (`StartGame`, `Exit`, …).

---

### Step 2: A generic menu state

Store the menu items and the player’s choice:

```rust
pub struct MenuState<T> {
    pub items: Vec<MenuItem<T>>,
    pub chosen: Option<T>,
}

impl<T: Clone> MenuState<T> {
    pub fn new(items: Vec<MenuItem<T>>) -> Self {
        Self { items, chosen: None }
    }
}
```

---

### Step 3: Generic UI rendering

For example with **egui** (in Bevy), you can write one reusable function:

```rust
fn render_menu<T: Clone>(
    title: &str,
    menu: &mut MenuState<T>,
    ui: &mut egui::Ui,
) {
    for item in &menu.items {
        if ui.button(&item.label).clicked() {
            menu.chosen = Some(item.value.clone());
        }
    }
}
```

---

### Step 4: Using it in different game states

Now you can plug this into your state machine:

```rust
enum GameState {
    MainMenu(MenuState<MainMenuAction>),
    PauseMenu(MenuState<PauseAction>),
    PlayerAction { unit_id: usize, menu: MenuState<Action> },
    EnemyTurn,
}

#[derive(Clone)]
enum MainMenuAction { StartGame, Exit }

#[derive(Clone)]
enum PauseAction { Resume, Quit }

#[derive(Clone)]
enum Action { Attack, Defend, Wait }
```

Example system:

```rust
fn ui_system(mut game: ResMut<Game>, mut egui_ctx: EguiContexts) {
    match &mut game.state {
        GameState::MainMenu(menu) => {
            egui::Window::new("Main Menu").show(egui_ctx.ctx_mut(), |ui| {
                render_menu("Main Menu", menu, ui);
            });

            if let Some(choice) = menu.chosen.take() {
                match choice {
                    MainMenuAction::StartGame => {
                        // transition into the game
                        game.state = GameState::PlayerAction {
                            unit_id: 0,
                            menu: MenuState::new(vec![
                                MenuItem { label: "Attack".into(), value: Action::Attack },
                                MenuItem { label: "Defend".into(), value: Action::Defend },
                                MenuItem { label: "Wait".into(), value: Action::Wait },
                            ]),
                        };
                    }
                    MainMenuAction::Exit => std::process::exit(0),
                }
            }
        }
        GameState::PauseMenu(menu) => {
            egui::Window::new("Pause").show(egui_ctx.ctx_mut(), |ui| {
                render_menu("Pause Menu", menu, ui);
            });

            if let Some(choice) = menu.chosen.take() {
                match choice {
                    PauseAction::Resume => game.resume(),
                    PauseAction::Quit => game.quit(),
                }
            }
        }
        GameState::PlayerAction { unit_id, menu } => {
            egui::Window::new("Choose Action").show(egui_ctx.ctx_mut(), |ui| {
                render_menu("Actions", menu, ui);
            });

            if let Some(action) = menu.chosen.take() {
                game.execute_action(*unit_id, action);
                game.state = GameState::EnemyTurn;
            }
        }
        _ => {}
    }
}
```

---

### Why this works

* You only wrote **one `render_menu` function** that knows how to render buttons and capture clicks.
* The **menu items** (labels + typed values) define what’s inside.
* Different states (`MainMenu`, `PauseMenu`, `PlayerAction`) each carry their own `MenuState<T>` with the right `T`.

So you can reuse this everywhere without duplicating UI boilerplate.

---

👉 Do you want me to adapt this into a **fully runnable Bevy + bevy\_egui example** (so you can copy-paste and see menus working)?


---
---
---
---

Great question 🙂 — now we’re talking about **dynamic action menus** that depend on unit state (class, equipment, status, etc.). The good news is the reusable `MenuState<T>` approach I showed already scales to this — you just need to generate the `Vec<MenuItem<Action>>` dynamically per unit.

---

## 1. Make actions data-driven

Instead of hardcoding `Attack`, `Defend`, etc., let units generate their own set:

```rust
#[derive(Clone, Debug)]
pub enum Action {
    AttackWith(String),  // e.g. "Sword", "Bow"
    Defend,
    Wait,
    UseItem(String),     // e.g. "Potion"
}
```

This way, one action type (`Action`) can cover all variations, but you keep the *specifics* in data (like which weapon or which item).

---

## 2. Unit decides what actions are available

Each unit can have a method to produce a menu:

```rust
struct Unit {
    name: String,
    class: String,
    weapons: Vec<String>,
    items: Vec<String>,
}

impl Unit {
    fn available_actions(&self) -> Vec<MenuItem<Action>> {
        let mut actions = Vec::new();

        // All units can wait
        actions.push(MenuItem { label: "Wait".into(), value: Action::Wait });

        // Some units can defend
        actions.push(MenuItem { label: "Defend".into(), value: Action::Defend });

        // Each weapon becomes an attack option
        for weapon in &self.weapons {
            actions.push(MenuItem {
                label: format!("Attack with {}", weapon),
                value: Action::AttackWith(weapon.clone()),
            });
        }

        // Each item becomes a usable option
        for item in &self.items {
            actions.push(MenuItem {
                label: format!("Use {}", item),
                value: Action::UseItem(item.clone()),
            });
        }

        actions
    }
}
```

So a knight with a sword + shield gives you `[Attack with Sword, Defend, Wait]`,
while an archer with a bow + dagger gives `[Attack with Bow, Attack with Dagger, Wait]`.

---

## 3. Creating the action menu in the game state

When you enter `PlayerAction`, you just ask the unit:

```rust
fn start_player_action(unit: &Unit) -> GameState {
    GameState::PlayerAction {
        unit_id: 0,
        menu: MenuState::new(unit.available_actions()),
    }
}
```

---

## 4. Executing the chosen action

Since `Action` carries enough data (`AttackWith("Sword")`, `UseItem("Potion")`, etc.), your executor just matches:

```rust
fn execute_action(&mut self, unit_id: usize, action: Action) {
    match action {
        Action::AttackWith(weapon) => {
            println!("Unit {unit_id} attacks with {weapon}!");
        }
        Action::Defend => println!("Unit {unit_id} defends!"),
        Action::Wait => println!("Unit {unit_id} waits."),
        Action::UseItem(item) => println!("Unit {unit_id} uses {item}!"),
    }
}
```

---

## 5. Why this scales

* Adding new actions = extend the `Action` enum.
* Different unit classes automatically generate different menus by overriding `available_actions()`.
* You still use the same generic `MenuState<Action>` + `render_menu` code.

---

✅ With this, your **UI menu system is generic** (works for all menus), and your **gameplay logic decides what actions are available dynamically** per unit.

---

Would you like me to show you how this would look **plugged into the generic menu system from before** (so pause menu, main menu, and unit-specific action menus all use the same machinery)?
