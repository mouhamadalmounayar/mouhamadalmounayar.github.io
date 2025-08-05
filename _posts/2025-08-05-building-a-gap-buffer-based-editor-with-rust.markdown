For the last couple of weeks, I have been working on a tui snippet manager which allows users to save code snippets, display them and edit them without leaving the terminal. I decided to go with rust along with ratatui for this project for obvious reasons.

Thus, I needed to build a minimal editor widget with these basic functionalities :

- Basic editing: adding/deleting chars.
- Basic navigation: move the cursor around wherever I want in the editor.
- Syntax highlighting: because let's face it, no one likes to edit code as plain text.

I was tempted to use tui-textarea at first, since the editor worked out of the box, and I didn't have the courage to go into implementing an editor by myself.

I was disappointed to discover that the rendered text didn’t support ANSI escape codes, and that there was no way to convert ANSI formatting into Ratatui widget formatting while continuing to use tui-textarea, since it operates on plain text rather than line- or paragraph-based widgets. I quickly accepted the fact that I needed to implement my own minimal editor for this to work.

In my ratatui application, I was using a component based architecture, where each component keeps track of its local state, and when the component needs to communicate with other components in the application, it syncs the local state with the global state, which other components can access.

Here is what the `Component` trait looks like

```rust
pub trait Component {
    pub render(&self, frame: &Frame, area: Rect, state: &AppState);

    pub handle_event(&mut self, event: &Event, state: &mut AppState);
}
```

The `handleEvent` is called on one component per render iteration, (depending on which component is currently focused), so it is fine to give it a mutable reference to the state.

### Naive approach

The naive approach is to keep track of a `Vec<String>`or even a `String` in the local editor state, and handle the key events by updating the buffer.

For inserting characters at the end of the buffer this is fine, but it quickly becomes ineffiscient whent we want to move the cursor around.

For example, if we want to insert a character in the middle of our buffer, we would need to shift all the characters to make space for that character, same issue if we want to delete a character in the middle of our buffer. Insert and Delete operations become pretty costly, especially if we are dealing with large texts.

### Gap Buffer

A gap buffer is a data structure which keeps a part of the array (or string) empty, allowing for `O(1)` insert and delete operations.

```rust
['H', 'e', 'l', 'l', 'o', '\0', '\0', '\0', '\0']
// ------actual text----  --------gap-----------
```

Here is how the data structure would look like :

```rust
struct GapBuffer {
    gap_start: usize,
    gap_end: usize,
    buffer: Vec<Char>
}
```

Inserting at the end of the buffer would simply look like :

```rust
self.buffer[gap_start] = c;
```

Inserting in the middle of the buffer would involve moving the gap, and then inserting. **While moving the gap may cost more than `O(1)`, the amortized cost over the number of insertions is still `O(1)`**.

```rust
// insert characters between index 1 and 2
['H', 'e', 'l', 'l', 'o', '\0', '\0', '\0', '\0']

// move the gap to the desired position
['H', 'e', '\0', '\0', '\0', '\0', 'l', 'l', 'o']

// insert characters
['H', 'e', 'l', '\0', '\0', '\0', 'l', 'l', 'o']
```

It is clear now what are the different methods which need to be implemented :

- A method to handle inserting characters.
- A method to handle shifting the gap.
- A method to handle growing the gap when the gap is full.

Here is the full code for the `GapBuffer` struct and its implementation :

```rust
pub struct GapBuffer {
    pub buffer: Vec<char>,
    pub capacity: usize,
    pub gap_start: usize,
    pub gap_end: usize,
}

impl GapBuffer {
    pub fn from_str(text: &str, capacity: usize) -> Self {
        let chars: Vec<char> = text.chars().collect();
        let length = chars.len();
        let mut buffer: Vec<char> = Vec::with_capacity(capacity + length);
        buffer.extend_from_slice(&chars);
        buffer.resize(capacity + length, '\0');
        GapBuffer {
            buffer,
            capacity,
            gap_start: length,
            gap_end: length + capacity - 1,
        }
    }
    fn move_gap_left(&mut self, index: usize) {
        while self.gap_start > index {
            self.gap_start -= 1;
            self.gap_end -= 1;
            self.buffer[self.gap_end + 1] = self.buffer[self.gap_start];
            self.buffer[self.gap_start] = '\0';
        }
    }

    fn move_gap_right(&mut self, index: usize) {
        while self.gap_start < index {
            self.gap_start += 1;
            self.gap_end += 1;
            self.buffer[self.gap_start - 1] = self.buffer[self.gap_end];
            self.buffer[self.gap_end] = '\0';
        }
    }

    fn grow(&mut self) {
        let new_capacity = self.capacity * 2;
        let new_size = new_capacity + self.buffer.len();
        let mut new_buffer = Vec::with_capacity(new_size);
        new_buffer.extend_from_slice(&self.buffer[..self.gap_start]);
        for _ in 0..new_capacity {
            new_buffer.push('\0');
        }
        let gap_end = new_buffer.len();
        new_buffer.extend_from_slice(&self.buffer[self.gap_end + 1..]);
        self.gap_end = gap_end;
        self.buffer = new_buffer;
    }

    pub fn insert_char(&mut self, c: char) {
        let gap_range = self.gap_end - self.gap_start;
        if gap_range == 1 {
            self.grow();
        }
        self.buffer[self.gap_start] = c;
        self.gap_start += 1;
    }

    pub fn delete_char(&mut self) {
        if self.gap_start == 0 {
            return;
        }
        self.gap_start -= 1;
        self.buffer[self.gap_start] = '\0';
    }

    pub fn move_gap(&mut self, index: usize) {
        let gap_size = self.gap_end - self.gap_start;
        if index + gap_size > self.buffer.len() {
            error!("Gap will overflow the buffer if moved to this index.");
            return;
        }
        if index == self.gap_start {
            info!("Gap is already positioned on this index.");
            return;
        }
        if index < self.gap_start {
            self.move_gap_left(index);
        }
        if index > self.gap_start {
            self.move_gap_right(index);
        }
    }
    pub fn to_string(&self) -> String {
        self.buffer.iter().filter(|&&c| c != '\0').collect()
    }
}
```

With the `GapBuffer` struct, handling editor's event is now easy. Here is my `handle_event()` method for the editor's component :

```rust
 fn handle_event(&mut self, event: &Event, state: &mut AppState) {
        let buffer = self
            .gap_buffer
            .as_mut()
            .expect("unexpected state buffer must not be null at this point");
        match event {
            Event::Key(key) => {
                if key.kind == KeyEventKind::Press {
                    match key.code {
                        KeyCode::Char(c) => {
                            buffer.insert_char(c);
                        }
                        KeyCode::Enter => {
                            buffer.insert_char('\n');
                        }
                        KeyCode::Backspace => {
                            buffer.delete_char();
                        }
                        KeyCode::Left => {
                            buffer.move_gap(buffer.gap_start.saturating_sub(1));
                        }
                        KeyCode::Right => {
                            buffer.move_gap(buffer.gap_start + 1);
                        }
                        KeyCode::Tab => {
                            for _ in 0..TAB_SIZE {
                                buffer.insert_char(' ');
                            }
                        }
                        _ => {}
                    }
                    let text_before_cursor = &buffer.buffer[..buffer.gap_start];
                    let line_count = text_before_cursor.iter().filter(|&&c| c == '\n').count() + 1;
                    let last_newline = text_before_cursor
                        .iter()
                        .rposition(|&c| c == '\n')
                        .map(|p| p + 1)
                        .unwrap_or(0);
                    let column = buffer.gap_start - last_newline;
                    self.cursor_coordinates = (
                        state.current_area.x + PADDING_SIZE + column as u16 + 1,
                        state.current_area.y + PADDING_SIZE + line_count as u16,
                    );
                    state.focus_editor();
                }
            }
            _ => {}
        }
    }
```

> Disclaimer: I am still new to rust, the code showed above might not 100% follow the best practices. But it works.

If you ignore the last part which serves for calculating and updating the new cursor position, the code for event handling is pretty straightforward using the `GapBuffer` struct.
