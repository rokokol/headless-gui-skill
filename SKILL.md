---
name: headless-gui
description: "Run, see and drive a GUI app in a private headless Wayland compositor, so that no window, focus change, cursor move or keystroke reaches the desktop the user works in. Use when a GUI or desktop app is started to test it, debug it, take a screenshot of it or click through it, Wayland and X11 apps alike. Triggers: run the app and show me, take a screenshot of the window, test the GUI, click the button, check how it looks, e2e test of the desktop app, запусти приложение и посмотри, сделай скриншот окна, протестируй GUI, нажми на кнопку, проверь как выглядит, не трогай мой экран"
license: MIT
---

# headless-gui

The user's desktop is a shared input device. A window that opens there takes focus, a synthetic key goes to whatever has focus, and a moved pointer moves under their hand. A GUI is run and driven only inside a nested sway with the headless backend: it renders into memory, it has its own Wayland socket and IPC socket, and nothing in it reaches the user's screen

## What never touches the user's session

- **No window in it.** A client connects to whatever `WAYLAND_DISPLAY` and `DISPLAY` say, and the agent's shell inherits both from the user's session. Every command below runs with them pointed at the nested compositor or unset
- **No IPC to the parent compositor.** `hyprctl dispatch`, a bare `swaymsg` that finds the user's `SWAYSOCK`, `kwin` or GNOME D-Bus calls all act on the user's desktop
- **No uinput tools.** `ydotool`, `dotool` and `evemu` create a kernel input device, and every compositor on the seat reads it, whatever `WAYLAND_DISPLAY` says. Input goes through Wayland protocols instead: `wtype` for keys, `wlrctl` for the pointer
- **A browser needs none of this.** A headless browser (Playwright, `--headless`) already renders off screen

## Start

The tools are `sway`, `grim`, `wtype` and `wlrctl`; on Nix, `nix shell nixpkgs#sway nixpkgs#grim nixpkgs#wtype nixpkgs#wlrctl` provides them for the commands that follow

```sh
R=$(mktemp -d /tmp/hgui.XXXX)   # mode 700; keep the path short, see below
env -u WAYLAND_DISPLAY -u DISPLAY -u SWAYSOCK -u HYPRLAND_INSTANCE_SIGNATURE \
  XDG_RUNTIME_DIR=$R WLR_BACKENDS=headless WLR_LIBINPUT_NO_DEVICES=1 WLR_RENDERER=pixman \
  setsid sway -c /dev/null >$R/sway.log 2>&1 </dev/null &
for i in $(seq 50); do ls $R | grep -q '^sway-ipc' && break; sleep 0.1; done
```

- **The socket path has to fit 108 bytes.** A unix socket address holds at most 108 bytes in `sun_path`, and sway puts `sway-ipc.<uid>.<pid>.sock` inside `XDG_RUNTIME_DIR`. A deep temporary directory overflows it, and sway exits with `Socket path won't fit into ipc_sockaddr->sun_path`
- **`WLR_RENDERER=pixman` renders on the CPU**, so the nested sway needs no GPU and no render node
- **`-c /dev/null` gives sway no config**, so no keybinding and no bar. The virtual output starts at 1280×720; `swaymsg output HEADLESS-1 resolution <W>x<H>` changes it

## Every later command

The shell does not keep variables between commands, so each one sets the same environment from `R` again:

```sh
unset DISPLAY HYPRLAND_INSTANCE_SIGNATURE
export WAYLAND_DISPLAY=$(echo $R/wayland-[0-9]) SWAYSOCK=$(echo $R/sway-ipc.*.sock)
```

- **Start the app from the agent's shell**: `setsid <command> >$R/app.log 2>&1 </dev/null &`. It keeps the shell's `PATH`, working directory and project environment. `swaymsg exec` loses them: the child gets sway's environment from the moment sway started, and fails with `command not found` for anything that was not on that `PATH`
- **An X11 app needs the nested Xwayland's display**: `swaymsg exec "echo \$DISPLAY > $R/display"` once, then `DISPLAY=$(cat $R/display)` before the command. The user's own Xwayland has a different number
- **Wait for the window, not for a time**: poll `swaymsg -t get_tree` until it holds the app's `"app_id"` (Wayland) or `"class"` (X11). The same tree gives each window's `rect` for a crop or a click
- **Screenshot**: `grim <file>.png`, or `grim -g '<x>,<y> <w>x<h>' <file>.png` for one window, then read the image
- **Keyboard**: `swaymsg '[app_id=<id>] focus'`, then `wtype '<text>'` and `wtype -k Return`
- **Pointer**: `wlrctl pointer move <dx> <dy>` is relative, so a point on the output is `wlrctl pointer move -10000 -10000` to the corner and then `wlrctl pointer move <x> <y>`; `wlrctl pointer click left` clicks. With no config, focus follows the pointer
- **Stop**: `swaymsg exit`, then trash `$R`. `Unable to receive IPC response` after `exit` is expected: sway exits before it answers. The apps and the nested Xwayland exit with it. When sway does not answer, stop it by the PID in its socket name `sway-ipc.<uid>.<pid>.sock`, never by name: the user may run a sway of their own
