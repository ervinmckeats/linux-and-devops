# Lesson 3. Anatomy of a command and getting help

## Goal
Read any command line and find help for any command without the internet.

## Theory

```
ls  -l -a  --human-readable  /var/log
│   │      │                 └─ argument (what to work on)
│   │      └─ long option (two dashes, a word)
│   └─ short options (one dash, one letter; can be joined: -la)
└─ command (a program, a builtin, an alias or a function)
```

- **Options** change *how* a command works. **Arguments** say *what* it works on.
- Short options combine: `ls -l -a -h` = `ls -lah`.
- Some options take a value: `head -n 5 file` or `head --lines=5 file`.
- `--` ends options: `rm -- -strange-file` removes a file whose name starts with a dash.

### What runs when you type a name

The shell checks, in order:
1. **Alias** (`alias ll='ls -la'`)
2. **Function**
3. **Builtin**: part of the shell itself (`cd`, `echo`, `export`)
4. **Executable** found in a directory listed in `$PATH`

```bash
type ls
type cd
type -a echo
which python3
echo $PATH
```

- `type ls`: `ls` is aliased to `ls --color=auto`
- `type cd`: cd is a shell builtin
- `type -a echo`: all matches, in order
- `which python3`: path of the executable
- `echo $PATH`: where the shell looks, left to right

`cd` must be a builtin: a child process cannot change the directory of its parent shell.

## Getting help

| Tool | When |
|---|---|
| `cmd --help` | quick list of options |
| `man cmd` | full manual |
| `tldr cmd` | practical examples |
| `help cd` | help for bash builtins |
| `apropos keyword` / `man -k` | "which command does X?" |

Inside `man`: `/word` search, `n` next match, `g`/`G` top/bottom, `q` quit.

Man sections you will use: `1` user commands, `5` file formats, `8` admin commands. `man 5 passwd` is the file format, `man 1 passwd` is the command.

## Commands

```bash
whoami; hostname; date; uptime
pwd
history | tail
!!
sudo !!
clear
```

- `whoami; hostname; date; uptime`: ; runs commands one after another
- `pwd`: where am I
- `history | tail`: last commands
- `!!`: repeat last command
- `sudo !!`: repeat last command as root
- `clear`: or Ctrl+L

### Shortcuts that save hours

| Key | Action |
|---|---|
| `Tab` / `Tab Tab` | complete / show options |
| `Ctrl+R` | search history |
| `Ctrl+A` / `Ctrl+E` | start / end of line |
| `Ctrl+W` | delete word before cursor |
| `Ctrl+U` / `Ctrl+K` | delete to start / to end |
| `Ctrl+C` | stop the running command |
| `Ctrl+D` | end of input / log out |
| `Alt+.` | paste last argument of previous command |

## Practice

1. Find with `apropos` a command that shows disk free space.
2. Using only `man ls`, find how to sort by modification time, newest last.
3. Run `type` on `ls`, `cd`, `ll`, `type`. Explain each answer.
4. Use `Ctrl+R` to find a command you ran in lesson 1.

## Check yourself

- Why is `cd` a builtin and not a program in `/usr/bin`?
- What does `--` do?
- How do you read the format of `/etc/crontab` in the manual?

## Next
[Lesson 4. Navigating the filesystem](04-navigation-fhs.md)
