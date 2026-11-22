# Weston with libinput key-forking (experimental)

This is an **experimental fork of Weston** whose only purpose is to make the
patched libinput "forking" plugin usable inside a Weston session. It is not a
general-purpose Weston fork and it is not meant for production.

The goal is simple: use my normal keyboard setup (my own XKB keymap plus the
forking plugin) in a Weston session, so that writing in `weston-terminal`
feels the same as in my usual Xorg session.

Upstream Weston documentation: <https://wayland.pages.freedesktop.org/weston/>

## Components

| Component | Repository | Role |
| --- | --- | --- |
| Patched libinput | <https://github.com/Michal-Maruska/libinput> (branch `mmc`) | adds `libinput_setup_fork()`, `libinput_register_fork()`, `libinput_fork_services` and the keyboard-event hook |
| Fork plugin | <https://github.com/michal-maruska/fork-plugin-core> (`libinput/`) | implements the forking state machine and builds `fork.so` |
| This Weston fork | this repository | calls `libinput_setup_fork()` and optionally loads a custom XKB keymap |

## What is different from upstream Weston

Only two changes are functional; the rest is comments, packaging metadata and
CI.

- `libweston/libinput-seat.c`: after creating the libinput context and setting
  up its log handler, Weston calls the new patched-libinput API:

  ```c
  libinput_setup_fork(input->libinput);
  ```

  This is the point where libinput `dlopen()`s `fork.so` and invokes its
  `fork_init()` entry point.

- `libweston/input.c`: when building the global keymap, Weston first tries
  `/usr/share/X11/xkb/keymap/xkb.mmc` with `xkb_keymap_new_from_file()`. If the
  file does not exist, it falls back to the normal RMLVO keymap
  (`xkb_keymap_new_from_names()`).

Everything else in the diff is exploratory comments, a `debian/changelog`
entry, the switch to `3.0 (native)` source format, and a GitHub workflow that
builds Debian packages.

## How the event flow works

```text
keyboard device
  -> libinput keyboard_notify_key()
     -> is a fork plugin registered?
        yes -> plugin.accept_event()
               -> forking machine decides
                  -> services->rewrite(event, new_keycode)
                  -> services->post_event(...)
        no  -> post_device_event() as usual
  -> Weston input handling
  -> focused client (e.g. weston-terminal)
```

The plugin is loaded from `LIBINPUT_MODULE_PATH` (default
`/usr/lib/libinput/modules/`). Its `fork_init(struct libinput_fork_services *)`
registers callbacks with `libinput_register_fork()`. The services provide
`rewrite()` for changing a keycode, `post_event()` for delivering the event,
and `log()` / `vlog()`.

## Minimum test

This is the shortest path to verify that libinput loads the plugin and that key
presses are rewritten.

### 1. Build and install the patched libinput

```sh
sudo apt install build-essential meson ninja-build pkg-config \
  libudev-dev libevdev-dev libmtdev-dev libwacom-dev

git clone https://github.com/Michal-Maruska/libinput.git
cd libinput
git checkout mmc
meson setup build --prefix=/usr -Dtests=false
ninja -C build
sudo ninja -C build install
sudo ldconfig
```

### 2. Build and install the forking plugin

```sh
sudo apt install cmake g++ libinput-dev

git clone https://github.com/michal-maruska/fork-plugin-core.git
cd fork-plugin-core

cmake -S libinput -B build-libinput -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DFETCHCONTENT_SOURCE_DIR_FORKING_MACHINE="$PWD"

cmake --build build-libinput
sudo cmake --install build-libinput
```

This installs `/usr/lib/libinput/modules/fork.so`.

The sample plugin in `libinput/fork-plugin.cpp` has hardcoded `configure_key()`
calls tailored to the author's custom keymap. For the smallest possible
rewrite test, replace that block with a single mapping, for example Space to
LeftCtrl:

```c
forking_machine->configure_key(fork_configure_key_fork,
                               57 /* KEY_SPACE */,
                               29 /* KEY_LEFTCTRL */,
                               1);
```

Then rebuild and reinstall `fork.so`.

### 3. Build and install this Weston

```sh
meson setup build --prefix=/usr
ninja -C build
sudo ninja -C build install
```

Alternatively, build the Debian packages from this tree with the usual Debian
tooling.

### 4. Run Weston on the DRM/udev backend

libinput is used by Weston's DRM (udev) backend. A nested Weston using the
Wayland or X11 backend does **not** exercise this code path.

From a free VT, with your display manager stopped:

```sh
WESTON_LIBINPUT_LOG_PRIORITY=debug weston --backend=drm --tty=2
```

The log should show the plugin being loaded:

```text
trying to open fork
Loading module '/usr/lib/libinput/modules/fork.so'
trying to invoke fork_init
call fork_init
hello from C++
```

If the module lives somewhere else, set
`LIBINPUT_MODULE_PATH=/path/to/modules`.

### 5. Verify key rewriting

Start `weston-terminal`. With the minimal Space-to-LeftCtrl mapping above:

- tap Space: a normal space is typed;
- hold Space and press `C`: the event is rewritten to Ctrl+C, for example it
  interrupts `sleep 100`.

## Custom XKB keymap

Weston normally only accepts RMLVO names. This fork additionally accepts a
complete keymap file at:

```text
/usr/share/X11/xkb/keymap/xkb.mmc
```

That is what makes the session feel like home: the forking plugin rewrites
keycodes, and the custom XKB keymap gives those keycodes the intended
modifier behaviour. Without the file, Weston falls back to the normal keymap.

## Notes

- Rebuild `fork.so` after changing the patched libinput, so both use compatible
  headers.
- The sample plugin enables verbose debugging with `set_debug(1)`. Set it to
  `0` for normal use.
- More background is in `porting-to-weston.md` in this tree.
