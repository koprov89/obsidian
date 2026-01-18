---
tags:
  - hotkeys
  - linux
  - dev-tools
date: 2026-01-14
---

# Terminal Hotkeys

Essential keyboard shortcuts for efficient terminal navigation and editing in bash/zsh.

## Line Navigation

- `Ctrl+A` - Move cursor to beginning of line
- `Ctrl+E` - Move cursor to end of line
- `Alt+B` - Move cursor backward one word
- `Alt+F` - Move cursor forward one word

## Editing

- `Ctrl+U` - Cut from cursor to beginning of line
- `Ctrl+K` - Cut from cursor to end of line
- `Ctrl+W` - Cut word before cursor
- `Alt+D` - Cut word after cursor
- `Ctrl+Y` - Paste (yank) last cut text
- `Ctrl+T` - Swap current character with previous
- `Alt+T` - Swap current word with previous
- `Alt+U` - Uppercase from cursor to end of word
- `Alt+L` - Lowercase from cursor to end of word
- `Alt+C` - Capitalize current word

## Deletion

- `Ctrl+H` - Delete character before cursor (same as Backspace)
- `Ctrl+D` - Delete character under cursor (or exit shell if line empty)
- `Alt+Backspace` - Delete word before cursor (alternative to Ctrl+W)

## History

- `Ctrl+R` - Reverse search through command history (press again for next match)
- `Ctrl+S` - Forward search (may need `stty -ixon` to enable)
- `Ctrl+G` - Escape from history search
- `Ctrl+P` - Previous command (same as Up arrow)
- `Ctrl+N` - Next command (same as Down arrow)
- `Alt+.` - Insert last argument from previous command (press repeatedly to cycle)
- `Alt+N Alt+.` - Insert Nth argument from previous command

## Screen Control

- `Ctrl+L` - Clear screen (same as `clear` command)
- `Ctrl+S` - Stop output to screen (freeze)
- `Ctrl+Q` - Resume output to screen
- `Ctrl+C` - Interrupt/kill current process
- `Ctrl+Z` - Suspend current process (resume with `fg` or `bg`)
- `Ctrl+D` - Exit current shell (EOF)

## Advanced

- `Alt+#` - Comment out current line and execute (adds # at start)
- `Ctrl+X Ctrl+E` - Open current command in default editor ($EDITOR)
- `!!` - Execute last command (not a hotkey but useful)
- `!$` - Last argument of previous command
- `!^` - First argument of previous command

## Tips

1. **Enable incremental search:** Add to `~/.bashrc` or `~/.zshrc`:
   ```bash
   # Disable XON/XOFF flow control to enable Ctrl+S
   stty -ixon
   ```

2. **Check your bindings:** Run `bind -P` (bash) or `bindkey` (zsh)

3. **Custom bindings:** Add to `~/.inputrc` for bash or use `bindkey` in zsh

## Related Notes
- [[hotkeys/Obsidian]]
