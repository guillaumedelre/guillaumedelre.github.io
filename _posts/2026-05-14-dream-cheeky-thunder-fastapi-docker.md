---
layout: post
title: "Controlling a USB missile launcher over HTTP with FastAPI and Docker"
date: 2026-05-14
categories: [python, docker, hardware]
tags: [fastapi, usb, docker, wsl2, pyusb]
---

A few years ago I rescued a [Dream Cheeky Thunder](http://www.dreamcheeky.com/thunder-missile-launcher) from a desk drawer. Four foam missiles, a USB cable, and absolutely no reason to leave it collecting dust. The obvious move: wrap it in a REST API, put it in a Docker container, and shoot things from `curl`.

This is the story of [dream-cheeky-thunder](https://github.com/guillaumedelre/dream-cheeky-thunder).

![Dream Cheeky Thunder](https://raw.githubusercontent.com/guillaumedelre/dream-cheeky-thunder/develop/docs/Dream-Cheeky-Thunder.jpg)

## :wrench: The hardware

The Dream Cheeky Thunder is a small USB missile launcher with two motorized axes:

- **Yaw (horizontal):** -135° to +135°
- **Pitch (vertical):** -5° to +45°
- **Capacity:** 4 foam missiles, ~4.5 seconds between shots

It speaks USB HID. The vendor ID is `0x2123`, the product ID is `0x1010`. You send raw USB control messages to move the motors and trigger the firing mechanism. No official SDK, no documentation — just a vendored Python script from 2012 floating around the internet, and a lot of reverse engineering notes in forum threads.

## :electric_plug: The API

Rather than a one-off script, I wanted something I could call from anywhere: a browser, a shell alias, a cron job, another service. FastAPI was the right tool.

The server exposes these endpoints:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/status` | :bar_chart: Current state (connected, missiles, yaw, pitch) |
| POST | `/park` | :house: Drive to home position (hard stop) |
| POST | `/move/{direction}` | :joystick: Raw move: `up`, `down`, `left`, `right` |
| POST | `/yaw/{angle}` | :left_right_arrow: Rotate to horizontal angle |
| POST | `/pitch/{angle}` | :arrow_up_down: Tilt to vertical angle |
| POST | `/fire` | :rocket: Fire N shots |
| POST | `/led` | :bulb: Toggle the LED ring |
| POST | `/reload` | :arrows_counterclockwise: Reset missile count after manual reload |

A typical targeting sequence looks like this:

```bash
curl -X POST http://localhost:8000/park      # reset to known position
curl -X POST http://localhost:8000/yaw/20    # aim right
curl -X POST http://localhost:8000/pitch/15  # tilt up
curl -X POST "http://localhost:8000/fire?shots=2"
```

The web UI at `http://localhost:8000` gives you buttons and sliders for the same operations, for the less terminal-inclined.

## :triangular_ruler: Angle tracking: time-based and honest about it

The launcher has no encoders. There is no way to read back the actual motor position. Angle tracking is entirely time-based: the server records how long each motor ran and estimates the current angle from that duration.

This works well enough for casual use. It degrades if the launcher is physically bumped or a command is interrupted mid-move. The `POST /park` endpoint solves this by driving both motors against their physical hard stops for the full sweep duration, guaranteeing a known position regardless of whatever the server thinks it knows.

Call `/park` before any precision targeting sequence.

## :whale: Docker and USB access

The tricky part of packaging this in Docker is USB access. By default, Docker containers do not see host USB devices.

The solution on Linux is a `devices` mount in `compose.yaml`:

```yaml
devices:
  - /dev/bus/usb:/dev/bus/usb
```

This exposes the entire USB bus to the container. Combined with a udev rule that sets the correct permissions on the device node, the container can open the launcher without running as root:

```bash
sudo cp udev/99-dream-cheeky.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules && sudo udevadm trigger
```

Without the udev rule, the container hits `USBError: [Errno 13] Access denied` even with the `devices` mount in place. The rule sets the device group and permissions so the container user can open it.

## :window: The WSL2 problem

On Windows with Docker Desktop using the WSL2 backend, the `devices` mount alone is not enough. WSL2 does not have access to USB devices by default — the Windows kernel holds them.

The fix is [usbipd-win](https://github.com/dorssel/usbipd-win), which forwards USB devices from Windows into the WSL2 kernel over IP:

```
Physical USB device
       |
  Windows kernel
       |
  usbipd-win        ← forwards to WSL2
       |
  WSL2 kernel       ← sees it as /dev/bus/usb/...
       |
  Docker container  ← reaches it via the devices mount
```

The setup:

```powershell
# Install usbipd-win
winget install usbipd

# Find the launcher's BUSID
usbipd list

# Attach to WSL2
usbipd attach --wsl --busid 20-2
```

With usbipd v4+, you can add a policy rule to automate this on reconnect:

```powershell
usbipd policy add --effect Allow --vid 2123 --pid 1010
```

After that, the Linux path (udev rule + Docker `devices` mount) works exactly the same in WSL2 as on a native Linux machine.

## :bulb: What I learned

A few things that surprised me:

:lock: **udev rules are not optional.** I initially thought the `devices` mount would be enough. It is not — the device node permissions are set at the OS level, and Docker inherits whatever the host has. The udev rule is the correct place to fix this, not running the container as root.

:dart: **Time-based positioning is surprisingly usable.** Given that there are no encoders, I expected the angle tracking to be useless. In practice, it is accurate enough for targeting if you `/park` first and don't bump the hardware.

:zap: **FastAPI + pyusb is a good combination for hardware APIs.** FastAPI gives you async, automatic OpenAPI docs, and a clean routing model. pyusb handles the USB layer. The two fit together cleanly: USB commands run in a thread pool executor so they don't block the event loop.

---

:octocat: The project is [on GitHub](https://github.com/guillaumedelre/dream-cheeky-thunder). Pull requests welcome, especially if you have a more precise angle calibration method.
