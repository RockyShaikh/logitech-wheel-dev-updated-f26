# Lab: Streaming Wheel Data to the Raspberry Pi

## Overview

The Logitech G920 plugs into your laptop. A **proxy** program on the laptop
reads the wheel and sends its state to the Pi as UDP packets. On the Pi,
upstream's `proxy_receiver` prints the steering, throttle and brake values as
they arrive, and sends a test "force" byte back so you can see the return path
working too.

```
G920 ──USB──> laptop (proxy) ──UDP :8000──> Raspberry Pi 4 (proxy_receiver)
                          <──UDP :8001── (force byte)
```

This is a refactor of the upstream's own test procedure for Windows (originally written exclusively for Windows, now adapted for MacOS), written out step by step with the setup
details filled in. The short version lives in the repo's
[README.md](README.md), under "Proxy Application Test"
([same section on GitHub](https://github.com/arjunr2/logitech-wheel-dev#proxy-application-test)).

If you are on Windows, you should follow the README. If you are on MacOS, follow the instructions below. Note, this was written with Apple Silicon in mind, x86 has not been tested. 

There are two proxies. They send byte-identical packets, so the Pi side is the
same either way:

| Laptop | Proxy | Part |
|---|---|---|
| macOS (or Linux) | `mac_proxy.py` | Part 2A |
| Windows | `proxy_gui.py` | Part 2B |

Each packet is 276 bytes: a 4-byte packet counter followed by a 272-byte
`DIJOYSTATE2` struct (defined in `proxy_receiver/state.h`).

Throughout this handout, text in `<angle-brackets>` is a placeholder. Replace
the whole thing, brackets included, with your own value.

## What goes where

| Machine | Files | Software |
|---|---|---|
| Raspberry Pi 4 | the `proxy_receiver/` folder (`receiver.c`, `state.h`) | `gcc` (preinstalled on Raspberry Pi OS) |
| Mac | `mac_proxy.py` | Python 3, `pygame-ce` |
| Windows | `proxy_gui.py`, `logitech_steering_wheel/` folder | Python 3, `PyQt5`, Logitech G HUB or Logitech Gaming Software |

Both the laptop and the Pi must be on the same network: both on CMU-SECURE,
both on the same home Wi-Fi, or joined directly by an Ethernet cable.

**Both programs have their addresses compiled or written in**, so collect two
addresses before you start:

- `<pi-ip>` — the Pi's address, from `hostname -I` on the Pi
- `<laptop-ip>` — the laptop's address, from `ipconfig` (Windows) or
  `ipconfig getifaddr en0` (macOS Wi-Fi)

Campus addresses change between sessions, so re-check them whenever something
that used to work stops working.

---

## Part 1: Raspberry Pi

**1.1** In a terminal on the Pi desktop, find the Pi's IP address:

```
hostname -I
```

The first address shown is `<pi-ip>`.

**1.2** Copy the receiver folder to the Pi. From the laptop, in the repo folder
(this works from macOS Terminal and from Windows PowerShell):

```
scp -r proxy_receiver <pi-user>@<pi-ip>:~/
```

A USB stick works too; `receiver.c` and `state.h` just need to end up in the
same folder.

**1.3** Set the addresses. On the Pi, open `~/proxy_receiver/receiver.c` and
edit the block marked `/** Configure this **/` near the top:

```c
#define LOCAL_HOST "<pi-ip>"      // this Pi's own address
#define R_PORT 8000               // leave as is

#define REMOTE_HOST "<laptop-ip>" // where the force byte is sent back
#define S_PORT 8001               // leave as is
```

`LOCAL_HOST` must be an address this Pi actually has, or the program exits at
startup with `bind failed`. These are `#define`s, so changing them means
recompiling (step 1.4).

**1.4** Build it, exactly as the README says:

```
cd ~/proxy_receiver
gcc -pthread receiver.c -o proxy_receiver
```

`-pthread` is required: the receiver runs its force-sending loop in a second
thread.

> **Checkpoint:** no output means it built.

**1.5** Run it:

```
./proxy_receiver
```

> **Checkpoint:** it prints `Send force 0`, `Send force 5`, `Send force 10` …
> about three times a second. That's the return path, and it runs whether or
> not a proxy is connected. Leave it running and go to Part 2A or 2B.

---

## Part 2A: Proxy on macOS

**2A.1** Set up Python (one time only). In Terminal, in the repo folder:

```
python3 -m venv .venv
source .venv/bin/activate
pip install pygame-ce
```

Install **`pygame-ce`**, not `pygame`. It's a maintained fork that is still
imported as `pygame`, and plain `pygame` has no build for recent Python
versions. In every new Terminal window, run `source .venv/bin/activate` before
continuing.

**2A.2** Plug in the wheel: its **power brick first**, then USB. The wheel
does a calibration sweep, turning full left and full right, so keep your hands
clear. macOS needs no driver, and G HUB won't list the wheel on macOS; both
are expected.

**2A.3** Find out which axis number is which control:

```
python3 mac_proxy.py --probe
```

The script takes a few seconds to start.

> **Checkpoint:** you see `Wheel: ...G920...`, then a live row of
> `a0 ... a1 ... a2 ...` values.

Move **one control at a time** and record:

| Control | Axis number (`aN`) | Value when released |
|---|---|---|
| Steering wheel | `<steer-axis>` | (about 0 when centered) |
| Gas pedal | `<throttle-axis>` | +1 or −1 |
| Brake pedal | `<brake-axis>` | +1 or −1 |

Press Ctrl-C when done.

**2A.4** Start sending, using the numbers from 2A.3. For example, if the probe
showed steering on `a0`, gas on `a1` and brake on `a2`, use `--steer 0
--throttle 1 --brake 2`.

```
python3 mac_proxy.py --remote <pi-ip> --steer <steer-axis> --throttle <throttle-axis> --brake <brake-axis> --interval-ms 50
```

- If a pedal read **−1** when released, add `--invert-throttle` and/or
  `--invert-brake`.
- If macOS asks whether Python may accept incoming network connections, click
  **Allow**. That's for the force byte coming back.

> **Checkpoint:** the Mac shows `steer ... thr ... brk ... sent N` with N
> counting up, and a `force` value that changes every 300 ms. The force value
> is the receiver's test counter, not real force feedback: the Mac proxy only
> displays it.

Go to Part 3.

---

## Part 2B: Proxy on Windows

**2B.1** Install the Logitech software. Upstream was tested with **Logitech
Gaming Software 5.10**. The current **G HUB** is also expected to work with
the SDK. If the proxy can't connect with one, try the other.

If Windows installed its own driver for the wheel before the Logitech
software, the SDK won't see the wheel. Upstream's fix:

1. Plug in and power the wheel.
2. In Device Manager, uninstall the driver Windows installed, and leave the
   wheel plugged in and powered.
3. Reinstall the Logitech software.

**2B.2** Install Python 3 from python.org, and tick "Add python.exe to PATH"
in the installer. Upstream was tested on Python 3.8; any current 3.x should
work. Then, in PowerShell, in the repo folder:

```
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install PyQt5
```

If PowerShell refuses to run `Activate.ps1`, run
`Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once, then try again.

Always run the proxy from the repo folder. `proxy_gui.py` imports the
`logitech_steering_wheel` folder that sits next to it, and that folder
contains the Logitech DLLs.

**2B.3** Configure the proxy. Open `proxy_gui.py` and edit the block marked
`## Configure this ##` near the top:

```python
REMOTE_HOST = "<pi-ip>"      # where wheel data is sent
S_PORT = 8000                # leave as is

LOCAL_HOST = "<laptop-ip>"   # must be this laptop's own IP
R_PORT = 8001                # leave as is
```

`LOCAL_HOST` must be an address this laptop actually has. If it isn't, the
proxy crashes on startup with an `OSError` about the requested address. Don't
commit these edits; the addresses are specific to your setup.

**2B.4** Plug in the wheel (power brick first, then USB), then start the proxy:

```
python proxy_gui.py
```

If Windows Defender Firewall asks about Python, allow it on the network type
you're using.

**2B.5** In the window that opens, click **connect**.

> **Checkpoint:** the console prints `initialized successfully` and
> `connected to a steering wheel at index 0`, and the wheel gives a short
> bump. The three number boxes in the window show steering, throttle and
> brake, and they change as you move the controls. Once the Pi's receiver is
> running you also get `Received | (Pkt N) : <force>` lines, and the wheel
> applies that value as a constant force.

Keep the proxy window open. **stop** disconnects from the wheel.

---

## Part 3: Verify on the Pi

With a proxy sending, `proxy_receiver` prints one line per packet:

```
Receive state (Pkt:      F1) :  Wheel: -8123 | Throttle: 31002 | Brake: 0
```

| Field | Meaning |
|---|---|
| `Pkt` | the proxy's packet counter, in hex |
| `Wheel` | `lX` from the struct: steering |
| `Throttle` | `lY` |
| `Brake` | `lRz` |

Work through this checklist and **record what you actually see**:

| Action | Mac proxy expected | Record |
|---|---|---|
| Wheel full left, then full right | `Wheel` about −32767 → +32767 | |
| Gas released, then fully pressed | `Throttle` 0 → about +32767 | |
| Brake released, then fully pressed | `Brake` 0 → about +32767 | |
| Both proxies running in turn, same controls | the two should agree | |

**Windows is the reference.** On Windows, record `Wheel`, `Throttle` and
`Brake` at rest and at full travel. The pedal range there may differ from the
Mac proxy's 0 → +32767, for example running from positive when released to
negative when pressed. If they differ, the Mac proxy's scaling is what changes,
so that the Pi never has to know which proxy is upstream.

The counter is the cheapest diagnostic in the system: steadily increasing means
the link is healthy, jumps mean dropped UDP packets, a frozen counter means the
proxy stalled, and a counter that restarts at 0 means the proxy was restarted.

### Done when

- Steering, throttle and brake all respond to the controls.
- The packet counter increases steadily.
- The force values appear on the laptop side, showing the return path works.
- You have recorded the steer, throttle and brake values at rest and at full
  travel, for whichever proxy you used.

---

## Troubleshooting

### Pi

| Symptom | Likely cause |
|---|---|
| `bind failed: Cannot assign requested address` | `LOCAL_HOST` in `receiver.c` isn't this Pi's address. Re-check `hostname -I`, edit, and recompile. |
| `bind failed: Address already in use` | Another copy of `proxy_receiver` is still running. Stop it. |
| `undefined reference to 'pthread_create'` | `-pthread` was left off the `gcc` command. |
| `Send force` lines but no `Receive state` lines | Wrong `REMOTE_HOST` in the proxy, laptop and Pi on different networks, or campus Wi-Fi blocking device-to-device traffic (use an Ethernet cable). Test with `ping <pi-ip>` from the laptop. |
| Packet counter jumps around | Dropped UDP packets. Normal on Wi-Fi now and then; constant drops mean switch to Ethernet. |
| Pi reboots, freezes, or SSH drops | Undervoltage. Run `vcgencmd get_throttled`; anything other than `0x0` means the power supply is inadequate. Use the official 5 V 3 A USB-C supply. |
| `fatal error: state.h: No such file or directory` | `state.h` isn't in the same folder as `receiver.c`. |

### Mac proxy

| Symptom | Likely cause |
|---|---|
| `No joystick found` | Wheel not powered (the brick is required), a charge-only USB cable, or a USB hub. Plug straight into the laptop. |
| `invalid int value` for `--steer` etc. | A placeholder was left in. Use the numbers from `--probe`. |
| Probe runs but every axis stays at 0 | The wheel hasn't finished calibrating, or the script was modified to remove `SDL_VIDEODRIVER=dummy`. |
| `No matching distribution found for pygame` | You installed `pygame` instead of `pygame-ce`. |
| `externally-managed-environment` from pip | The virtual environment isn't active. Run `source .venv/bin/activate`. |
| `ModuleNotFoundError: No module named 'pygame'` | The virtual environment isn't active in this Terminal window. |
| `force --` never changes | The Pi's receiver isn't running, or its `REMOTE_HOST` isn't this laptop. |

### Windows proxy

| Symptom | Likely cause |
|---|---|
| `OSError` on startup mentioning the address | `LOCAL_HOST` in `proxy_gui.py` isn't this laptop's IP (step 2B.3). |
| `No module named 'logitech_steering_wheel'` | Not running from the repo folder. `cd` into it first. |
| `ModuleNotFoundError: No module named 'PyQt5'` | The virtual environment isn't active, or `pip install PyQt5` was skipped. |
| Clicking **connect** prints nothing, or not `connected ...` | The Logitech software isn't installed or running, or Windows's generic driver took over (see 2B.1). Try the other Logitech software. |
| The window's numbers don't move | The wheel hasn't finished calibrating, or the SDK lost the device. Click **stop**, unplug and replug the wheel, then click **connect**. |
| Values move in the window but the Pi sees nothing | Wrong `REMOTE_HOST`, laptop and Pi on different networks, or the firewall blocked Python. |
| Console prints `No data` constantly | The Pi's receiver isn't running or can't reach this laptop (check `REMOTE_HOST` in `receiver.c`). |
