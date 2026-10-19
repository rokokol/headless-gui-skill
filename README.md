<div align="center">

# headless-gui skill

**Let the agent click through your app without touching your screen**

[![Agent Skill](https://img.shields.io/badge/Agent_Skill-6E56CF?style=flat)](https://agentskills.io)
[![license](https://img.shields.io/badge/MIT-3DA639?style=flat)](LICENSE)

</div>

An agent that tests a GUI app on your desktop shares your input with it. The window takes focus while you type, a synthetic keystroke goes to your editor, the pointer jumps under your hand. A tool that injects input through `uinput` reaches your session whatever display it was given. An app started on your D-Bus session can hand its window to the copy you already run, so even an app the agent started elsewhere can open on your screen

This skill gives the agent its own desktop instead: a nested sway with the headless backend, rendered on the CPU into memory, with its own Wayland and IPC sockets and its own D-Bus session. The agent starts the app there, waits for its window, clicks through sway's own seat commands, types with `wtype` and reads screenshots from `grim`. Wayland apps and X11 apps through Xwayland both work. Your compositor can be anything: the nested sway does not depend on it

A headless sway has no input devices, and a toolkit such as GTK ignores keys and clicks until the seat has a pointer and a keyboard. An idle VNC client connected to `wayvnc` over a private unix socket keeps both on the seat for as long as the session runs

## Requirements

Linux with `sway`, `grim`, `wtype`, `wayvnc`, vncdotool and dbus on hand, from the package manager or from `nix shell`. No GPU is needed

## Install

```bash
npx skills add -g rokokol/headless-gui-skill    # for you, everywhere
npx skills add rokokol/headless-gui-skill       # for the project you are standing in
```

Claude Code also takes it as a plugin:

```
/plugin marketplace add rokokol/skills
/plugin install headless-gui@rokokol-skills
```

or by hand — clone into whichever skills directory your agent reads:

```bash
git clone https://github.com/rokokol/headless-gui-skill ~/.claude/skills/headless-gui
```

> [!NOTE]
> A skill has no version to pin. It is read at whatever revision you have checked out, so `git pull` is the whole upgrade path

## Use

Ask the agent to run, test or screenshot a GUI app; the phrases it listens for are in the description of [SKILL.md](SKILL.md). The commands live in SKILL.md, where the agent reads them, and are not copied here
