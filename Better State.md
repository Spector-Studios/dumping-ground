Perfect, this is where the **pushdown automaton + stateful UI** really shines.
We’ll add two things:

1. **Nested Menus**: e.g. `PlayerAction → SkillMenuState`

   * Push `SkillMenuState` when the player chooses “Skill.”
   * Pop back to `PlayerAction` after selecting a skill.

2. **Stateful UI Widgets**:

   * Instead of only mouse-driven buttons, menus maintain a `selected_index`.
   * Controller/keyboard input (`Up`/`Down`/`Enter`) changes selection and confirms.
   * Works for gamepads too (macroquad has `is_key_pressed`, and you can layer gamepad support later).

---

## 🕹️ Skeleton: Stateful UI + Submenus

```rust
use macroquad::prelude::*;
use std::collections::VecDeque;

// ---------------- Messages ----------------
#[derive(Debug)]
pub enum GameMsg {
    UnitSelected(u32),
    TileChosen((i32, i32)),
    ActionChosen(ActionType),
    SkillChosen(String),
}

#[derive(Debug)]
pub enum ActionType {
    Attack,
    Skill,
    Item,
}

// ---------------- State Machine ----------------
pub enum Transition {
    None,
    Push(Box<dyn GameState>),
    Pop,
    Switch(Box<dyn GameState>),
}

pub trait GameState {
    fn update(&mut self, msg_queue: &mut VecDeque<GameMsg>) -> Transition;
    fn draw(&self, msg_queue: &mut VecDeque<GameMsg>);
    fn name(&self) -> &str;
}

pub struct StateMachine {
    stack: Vec<Box<dyn GameState>>,
    pub msg_queue: VecDeque<GameMsg>,
}

impl StateMachine {
    pub fn new(initial: Box<dyn GameState>) -> Self {
        Self {
            stack: vec![initial],
            msg_queue: VecDeque::new(),
        }
    }

    pub fn update(&mut self) {
        if let Some(state) = self.stack.last_mut() {
            match state.update(&mut self.msg_queue) {
                Transition::None => {}
                Transition::Pop => { self.stack.pop(); }
                Transition::Push(s) => { self.stack.push(s); }
                Transition::Switch(s) => {
                    self.stack.pop();
                    self.stack.push(s);
                }
            }
        }
    }

    pub fn draw(&mut self) {
        if let Some(state) = self.stack.last() {
            state.draw(&mut self.msg_queue);
        }
    }

    pub fn current_state_name(&self) -> &str {
        self.stack.last().map(|s| s.name()).unwrap_or("None")
    }
}

// ---------------- Stateful Menu Widget ----------------
struct Menu {
    items: Vec<String>,
    selected: usize,
}

impl Menu {
    fn new(items: Vec<&str>) -> Self {
        Self {
            items: items.into_iter().map(|s| s.to_string()).collect(),
            selected: 0,
        }
    }

    fn update(&mut self) -> Option<usize> {
        if is_key_pressed(KeyCode::Down) {
            self.selected = (self.selected + 1) % self.items.len();
        }
        if is_key_pressed(KeyCode::Up) {
            if self.selected == 0 {
                self.selected = self.items.len() - 1;
            } else {
                self.selected -= 1;
            }
        }
        if is_key_pressed(KeyCode::Enter) {
            return Some(self.selected);
        }
        None
    }

    fn draw(&self, x: f32, y: f32, msg_queue: &mut VecDeque<GameMsg>, f: impl Fn(&str) -> GameMsg) {
        for (i, item) in self.items.iter().enumerate() {
            let color = if i == self.selected { YELLOW } else { WHITE };
            draw_text(item, x, y + i as f32 * 30.0, 30.0, color);

            // Mouse support too
            let (mx, my) = mouse_position();
            let bx = x;
            let by = y + i as f32 * 30.0 - 24.0;
            let bw = 200.0;
            let bh = 30.0;

            let hovering = mx >= bx && mx <= bx + bw && my >= by && my <= by + bh;
            if hovering {
                draw_rectangle_lines(bx, by, bw, bh, 2.0, GRAY);
                if is_mouse_button_pressed(MouseButton::Left) {
                    msg_queue.push_back(f(item));
                }
            }
        }
    }
}

// ---------------- States ----------------
struct PlayerSelect;
impl GameState for PlayerSelect {
    fn update(&mut self, msg_queue: &mut VecDeque<GameMsg>) -> Transition {
        if is_key_pressed(KeyCode::Enter) {
            msg_queue.push_back(GameMsg::UnitSelected(1)); // demo
        }

        if let Some(msg) = msg_queue.pop_front() {
            match msg {
                GameMsg::UnitSelected(id) => {
                    println!("Unit {} selected", id);
                    return Transition::Push(Box::new(PlayerMove));
                }
                _ => {}
            }
        }
        Transition::None
    }

    fn draw(&self, _msg_queue: &mut VecDeque<GameMsg>) {
        draw_text("Press Enter to select a unit", 20.0, 40.0, 30.0, WHITE);
    }

    fn name(&self) -> &str { "PlayerSelect" }
}

struct PlayerMove;
impl GameState for PlayerMove {
    fn update(&mut self, msg_queue: &mut VecDeque<GameMsg>) -> Transition {
        if is_key_pressed(KeyCode::Enter) {
            msg_queue.push_back(GameMsg::TileChosen((2,3)));
        }

        if let Some(msg) = msg_queue.pop_front() {
            match msg {
                GameMsg::TileChosen(pos) => {
                    println!("Moving to {:?}", pos);
                    return Transition::Switch(Box::new(PlayerAction::new()));
                }
                _ => {}
            }
        }
        Transition::None
    }

    fn draw(&self, _msg_queue: &mut VecDeque<GameMsg>) {
        draw_text("Press Enter to pick destination (2,3)", 20.0, 40.0, 30.0, YELLOW);
    }

    fn name(&self) -> &str { "PlayerMove" }
}

struct PlayerAction {
    menu: Menu,
}

impl PlayerAction {
    fn new() -> Self {
        Self {
            menu: Menu::new(vec!["Attack", "Skill", "Item"]),
        }
    }
}

impl GameState for PlayerAction {
    fn update(&mut self, msg_queue: &mut VecDeque<GameMsg>) -> Transition {
        if let Some(index) = self.menu.update() {
            match index {
                0 => msg_queue.push_back(GameMsg::ActionChosen(ActionType::Attack)),
                1 => return Transition::Push(Box::new(SkillMenu::new())),
                2 => msg_queue.push_back(GameMsg::ActionChosen(ActionType::Item)),
                _ => {}
            }
        }

        if let Some(msg) = msg_queue.pop_front() {
            match msg {
                GameMsg::ActionChosen(a) => {
                    println!("Performing action: {:?}", a);
                    return Transition::Pop;
                }
                _ => {}
            }
        }
        Transition::None
    }

    fn draw(&self, msg_queue: &mut VecDeque<GameMsg>) {
        draw_text("Choose action (arrows + Enter):", 20.0, 40.0, 30.0, WHITE);
        self.menu.draw(20.0, 80.0, msg_queue, |s| match s {
            "Attack" => GameMsg::ActionChosen(ActionType::Attack),
            "Skill" => GameMsg::ActionChosen(ActionType::Skill),
            "Item" => GameMsg::ActionChosen(ActionType::Item),
            _ => unreachable!(),
        });
    }

    fn name(&self) -> &str { "PlayerAction" }
}

struct SkillMenu {
    menu: Menu,
}

impl SkillMenu {
    fn new() -> Self {
        Self {
            menu: Menu::new(vec!["Fireball", "Heal", "Lightning"]),
        }
    }
}

impl GameState for SkillMenu {
    fn update(&mut self, msg_queue: &mut VecDeque<GameMsg>) -> Transition {
        if let Some(index) = self.menu.update() {
            let skill = self.menu.items[index].clone();
            msg_queue.push_back(GameMsg::SkillChosen(skill));
        }

        if let Some(msg) = msg_queue.pop_front() {
            match msg {
                GameMsg::SkillChosen(skill) => {
                    println!("Skill chosen: {}", skill);
                    return Transition::Pop; // go back to PlayerAction
                }
                _ => {}
            }
        }
        Transition::None
    }

    fn draw(&self, msg_queue: &mut VecDeque<GameMsg>) {
        draw_text("Choose skill:", 20.0, 40.0, 30.0, WHITE);
        self.menu.draw(20.0, 80.0, msg_queue, |s| GameMsg::SkillChosen(s.to_string()));
    }

    fn name(&self) -> &str { "SkillMenu" }
}

// ---------------- Macroquad Main ----------------
#[macroquad::main("RPG Menus")]
async fn main() {
    let mut sm = StateMachine::new(Box::new(PlayerSelect));

    loop {
        clear_background(BLACK);

        sm.update();
        sm.draw();

        draw_text(
            &format!("Current state: {}", sm.current_state_name()),
            20.0,
            280.0,
            20.0,
            GRAY,
        );

        next_frame().await;
    }
}
```

---

## 🔑 Highlights

* **Nested Menus**:

  * Selecting “Skill” in `PlayerAction` pushes `SkillMenu` onto the stack.
  * When a skill is chosen, `SkillMenu` pops back → you’re in `PlayerAction` again.

* **Stateful UI**:

  * `Menu` tracks `selected` index.
  * Input (Up/Down/Enter) moves through items.
  * Also supports mouse hover + click.

* **Pushdown Automaton fits naturally**:

  * You don’t hardcode transitions for every submenu.
  * Each submenu manages itself and returns `Pop` when done.

---

👉 Would you like me to **expand the SkillMenu** example to show *nested confirmation dialogs* (e.g. “Are you sure you want to cast Fireball?” → Yes/No submenu), so you see deeper nesting in action?
