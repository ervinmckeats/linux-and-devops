# Lesson 7. Editors: nano and vim

## Goal
Edit configs on any server. `vim` (or at least `vi`) is installed almost everywhere; `nano` is the easy fallback.

## nano

```bash
nano file
```
Commands are shown at the bottom (`^` = Ctrl): `Ctrl+O` save, `Ctrl+X` exit, `Ctrl+W` search, `Ctrl+K` cut line, `Ctrl+U` paste, `Alt+U` undo.

## vim

vim has **modes**. This is the only hard part.

| Mode | Enter with | For |
|---|---|---|
| Normal | `Esc` | moving, deleting, commands |
| Insert | `i`, `a`, `o` | typing text |
| Visual | `v`, `V` | selecting |
| Command | `:` | save, quit, replace |

### Survival kit

```
Esc :q!        quit without saving (when lost, press Esc twice and type this)
Esc :wq        save and quit (also :x or ZZ)
i              insert before cursor
o              new line below and insert
u / Ctrl+r     undo / redo
```

### Moving and editing (Normal mode)

```
h j k l        left down up right
w b            next / previous word
0 $            start / end of line
gg G           first / last line
:42            go to line 42
dd             delete (cut) line        5dd  five lines
yy             copy line                p    paste below
x              delete character
ciw            change the word under cursor
.              repeat last change
/text  n  N    search, next, previous
:%s/old/new/g  replace in whole file
:set number    line numbers
:set paste     paste without auto-indent mess
```

### `sudo` and vim

Opened a system file without sudo? Do not lose your edits:
```
:w !sudo tee % > /dev/null
```
Better habit: `sudoedit /etc/ssh/sshd_config`: edits a temp copy with your own editor settings and writes it back as root.

Set your default editor (used by `crontab -e`, `git commit`, `visudo`):
```bash
echo 'export EDITOR=vim' >> ~/.bashrc
```

## Practice

1. Run `vimtutor` and finish it (about 30 minutes). This is the best vim lesson there is.
2. Copy `/etc/services` to `~`, open in vim, delete lines 1–10, replace `tcp` with `TCP` everywhere, save.
3. Open a file in `/etc` without `sudo`, make a change and save it with the `tee` trick (in your lab).

## Check yourself

- You are stuck in vim. What do you type?
- What does `.` do in Normal mode?
- Why is `sudoedit` safer than `sudo vim`?

## Next
[Lesson 8. Inodes and links](08-inodes-and-links.md)
