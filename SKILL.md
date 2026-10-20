---
name: headless-gui
description: "Run, see and drive a GUI app in a private headless Wayland compositor, so that no window, focus change, cursor move or keystroke reaches the desktop the user works in. Use when a GUI or desktop app is started to test it, debug it, take a screenshot of it or click through it, Wayland and X11 apps alike. Triggers: run the app and show me, take a screenshot of the window, test the GUI, click the button, check how it looks, e2e test of the desktop app, запусти приложение и посмотри, сделай скриншот окна, протестируй GUI, нажми на кнопку, проверь как выглядит, не трогай мой экран"
license: MIT
---

# headless-gui

The user's desktop is a shared input device. A window that opens there takes focus, a synthetic key goes to whatever has focus, and a moved pointer moves under their hand. A GUI is run and driven only inside a nested sway with the headless backend: it renders into memory, it has its own Wayland and IPC sockets, and nothing in it reaches the user's screen

## What never touches the user's session

- **No window in it.** A client connects to whatever `WAYLAND_DISPLAY` and `DISPLAY` say, and the agent's shell inherits both from the user's session. Every command below runs with them pointed at the nested compositor or unset
- **No D-Bus call into it.** The shell also inherits `DBUS_SESSION_BUS_ADDRESS`. On the user's bus, an app with a unique id, such as a `GApplication`, finds the copy the user already runs and hands it the activation, so the window opens on the user's desktop; file dialogs through the portal and notifications appear there too. The nested sway runs under `dbus-run-session`, and every app gets that private bus
- **No shared profile.** An app that keeps one instance through a lock or a socket in its profile, such as Firefox or an Electron app, finds the user's copy without D-Bus. It starts with a profile or data directory of its own under `$R`, through the app's own flag
- **No IPC to the parent compositor.** `hyprctl dispatch`, a bare `swaymsg` that finds the user's `SWAYSOCK`, `kwin` or GNOME D-Bus calls all act on the user's desktop
- **No uinput tools.** `ydotool`, `dotool` and `evemu` create a kernel input device, and every compositor on the seat reads it, whatever `WAYLAND_DISPLAY` says. Input goes through the nested sway instead
- **A browser needs none of this.** A headless browser (Playwright, `--headless`) already renders off screen

## Start

The tools are `sway`, `grim`, `wtype`, `wayvnc`, `vncdo` from vncdotool and `dbus-run-session` from dbus; on Nix, `nix shell nixpkgs#sway nixpkgs#grim nixpkgs#wtype nixpkgs#wayvnc nixpkgs#python3Packages.vncdotool nixpkgs#dbus` provides them for the commands that follow

```sh
R=$(mktemp -d /tmp/hgui.XXXX)   # mode 700; keep the path short, see below
env -u WAYLAND_DISPLAY -u DISPLAY -u SWAYSOCK -u HYPRLAND_INSTANCE_SIGNATURE -u DBUS_SESSION_BUS_ADDRESS \
  XDG_RUNTIME_DIR=$R WLR_BACKENDS=headless WLR_LIBINPUT_NO_DEVICES=1 WLR_RENDERER=pixman \
  setsid dbus-run-session -- sway -c /dev/null >$R/sway.log 2>&1 </dev/null &
for i in $(seq 50); do ls $R | grep -q '^sway-ipc' && break; sleep 0.1; done
SWAYSOCK=$(echo $R/sway-ipc.*.sock) swaymsg exec \
  "echo DISPLAY=\$DISPLAY DBUS_SESSION_BUS_ADDRESS=\$DBUS_SESSION_BUS_ADDRESS > $R/env" >/dev/null
for i in $(seq 50); do [ -s $R/env ] && break; sleep 0.1; done
WAYLAND_DISPLAY=$(echo $R/wayland-[0-9]) setsid wayvnc -u -S $R/wayvnc.ctl $R/vnc.sock >$R/wayvnc.log 2>&1 </dev/null &
for i in $(seq 50); do [ -S $R/vnc.sock ] && break; sleep 0.1; done
setsid vncdo -s $R/vnc.sock pause 86400 >/dev/null 2>&1 </dev/null &
```

- **The socket path has to fit 108 bytes.** A unix socket address holds at most 108 bytes in `sun_path`, and sway puts `sway-ipc.<uid>.<pid>.sock` inside `XDG_RUNTIME_DIR`. A deep temporary directory overflows it, and sway exits with `Socket path won't fit into ipc_sockaddr->sun_path`
- **The idle VNC client keeps a pointer and a keyboard on the seat.** The headless backend has no input device, and a tool that creates one per call, such as `wtype`, removes it when it exits. A client binds its pointer and keyboard only after the seat announces them, so without the idle client a GTK app misses every key and click. `wayvnc` creates the devices for each connected client, and the idle `vncdo` stays connected. It starts before the app, so the app binds the devices at its own start. `wayvnc -u` listens on a unix socket inside `$R`, not on a TCP port, and `-S` keeps its control socket off the user's runtime directory
- **`WLR_RENDERER=pixman` renders on the CPU**, so the nested sway needs no GPU and no render node
- **`-c /dev/null` gives sway no config**, so no keybinding and no bar. The virtual output starts at 1280×720; `swaymsg output HEADLESS-1 resolution <W>x<H>` changes it

## Every later command

The shell does not keep variables between commands, so each one sets the same environment from `R` again:

```sh
unset HYPRLAND_INSTANCE_SIGNATURE
export XDG_RUNTIME_DIR=$R WAYLAND_DISPLAY=$(echo $R/wayland-[0-9]) SWAYSOCK=$(echo $R/sway-ipc.*.sock) $(cat $R/env)
```

`$R/env` holds what only sway knows: the `DISPLAY` of the Xwayland it starts on demand for X11 apps, and the address of the private bus. `XDG_RUNTIME_DIR` keeps the app's runtime files in `$R` as well: an app that keeps one instance through a lock file there, such as rofi's `rofi.pid`, otherwise takes the user's lock and stops the user's own copy from starting

- **Start the app from the agent's shell**: `setsid <command> >$R/app.log 2>&1 </dev/null &`. It keeps the shell's `PATH`, working directory and project environment. `swaymsg exec` loses them: the child gets sway's environment from the moment sway started, and fails with `command not found` for anything that was not on that `PATH`
- **An app the private bus starts needs the environment pushed to the bus first**: `dbus-update-activation-environment WAYLAND_DISPLAY DISPLAY XDG_RUNTIME_DIR PATH`. A desktop entry with `DBusActivatable=true`, launched through `gtk-launch` or `gio`, is started by the bus, and the bus has the environment from before sway started, with no `WAYLAND_DISPLAY`. Without the push the launch reports success and the app exits with no window
- **Wait for the window, not for a time**: poll `swaymsg -t get_tree` until it holds the app's `"app_id"` (Wayland) or `"class"` (X11). The same tree gives each window's `rect`, in the pixels a click and a crop use
- **Screenshot**: `grim <file>.png`, or `grim -g '<x>,<y> <w>x<h>' <file>.png` for one window, then read the image. `grim -c` draws the pointer too
- **Click**: `swaymsg seat seat0 cursor set <x> <y>`, then `swaymsg seat seat0 cursor press button1` and `swaymsg seat seat0 cursor release button1`. The position is absolute. With no config, focus follows the pointer
- **Keys and text**: `swaymsg '[app_id=<id>] focus'`, then `wtype '<text>'`, which types any Unicode text, `wtype -k Return` for a named key and `wtype -M ctrl a -m ctrl` for a chord. `vncdo` is only the idle client: its `type` drops every non-ASCII character
- **Stop**: `swaymsg exit`, then trash `$R`. `Unable to receive IPC response` after `exit` is expected: sway exits before it answers. The apps, the nested Xwayland, the private bus, `wayvnc` and the idle client exit with it. When sway does not answer, stop it by the PID in its socket name `sway-ipc.<uid>.<pid>.sock`, never by name: the user may run a sway of their own
