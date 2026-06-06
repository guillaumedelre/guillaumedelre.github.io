---
title: "Controlling a USB missile launcher over HTTP with FastAPI and Docker"
date: 2017-02-21
categories: [iot]
tags: [fastapi, usb, docker, wsl2, pyusb]
description: "How we wired a USB foam missile launcher to the CI pipeline — and what Docker, udev, and WSL2 had to say about it."
---

The rule was simple: whoever breaks the CI build owes the team a coffee. It worked fine for a while. Then someone suggested we needed something with more immediate feedback. Something physical. Something that fires.

A <a href="http://www.dreamcheeky.com/thunder-missile-launcher" target="_blank" rel="noopener noreferrer">Dream Cheeky Thunder</a> appeared on a desk shortly after. Four foam missiles, a USB cable, and a very clear team consensus: hook it to the cluster, wire it to the build pipeline, and let the CI decide who deserves a volley.

The launcher needed to respond to HTTP calls from anywhere on the network. No driver, no GUI, no manual aiming. Just an endpoint that makes it shoot in the direction of the guilty party's desk.

This is the story of <a href="https://github.com/guillaumedelre/dream-cheeky-thunder" target="_blank" rel="noopener noreferrer">dream-cheeky-thunder</a>.

![Dream Cheeky Thunder](https://raw.githubusercontent.com/guillaumedelre/dream-cheeky-thunder/develop/docs/Dream-Cheeky-Thunder.jpg)

## No SDK, no docs, no problem

Dream Cheeky never published a protocol spec. The launcher speaks raw USB HID, and the only starting point was a vendored Python script from 2012 floating around in forum threads. Vendor ID `0x2123`, product ID `0x1010`, and a handful of control bytes that someone had reverse engineered years before.

That was enough. The protocol is simple: send a byte sequence to move the motors, send another to fire. The tricky part is that the launcher has no position feedback. No encoders, no limit switches beyond the physical hard stops at the extremes. You drive it blind.

## From USB to HTTP

The CI pipeline needed to trigger the launcher over the network. A local script wasn't going to cut it — the launcher had to be reachable from any machine on the cluster, including the build server. So: a REST API.

FastAPI was the obvious choice. The targeting flow from the CI side ends up being three HTTP calls:

```bash
curl -X POST http://localhost:8000/park      # reset to known position
curl -X POST http://localhost:8000/yaw/20    # rotate toward guilty desk
curl -X POST "http://localhost:8000/fire?shots=2"
```

The `/park` call matters more than it looks. Since the launcher has no position feedback, the server estimates the current angle by tracking how long the motors have been running. That estimate drifts. Bumping the hardware, interrupting a command, or just the imprecision of time-based tracking — they all accumulate. Parking drives both motors against the physical hard stops at full sweep, which guarantees alignment regardless of what the server thinks it knows. Skip it, and your aim is a guess.

The full API reference is <a href="https://github.com/guillaumedelre/dream-cheeky-thunder/blob/develop/docs/api.md" target="_blank" rel="noopener noreferrer">in the repo</a>. There's also a web UI if you prefer clicking over `curl`.

## Docker knows nothing about USB

Running this in a Docker container on the cluster was where the fun really started: containers don't see USB devices by default.

The `devices` mount in `compose.yaml` exposes the USB bus to the container:

```yaml
devices:
  - /dev/bus/usb:/dev/bus/usb
```

Not enough. First run came back with `USBError: [Errno 13] Access denied`. The device node is there inside the container, but it inherits permissions from the host, and on the host only root can open it by default.

The fix is a udev rule. Drop one file into `/etc/udev/rules.d/`, and the kernel sets the right group and permissions when the device plugs in. After that, the container user can open it without needing elevated privileges. The rule ships with the project, setup instructions are <a href="https://github.com/guillaumedelre/dream-cheeky-thunder/blob/develop/docs/setup-linux.md" target="_blank" rel="noopener noreferrer">in the docs</a>.

## WSL2 made it interesting

Half the team runs Windows with Docker Desktop on WSL2. That's where things got creative.

WSL2 has no access to USB devices by default: the Windows kernel holds them, and the `devices` mount alone does nothing because WSL2 simply doesn't see the hardware. The fix is <a href="https://github.com/dorssel/usbipd-win" target="_blank" rel="noopener noreferrer">usbipd-win</a>, which forwards the USB device from Windows into the WSL2 kernel over IP. Once that's done, the Linux path works exactly the same: udev rule, `devices` mount, done.

The attachment doesn't survive reboots, though. usbipd v4+ added a policy mechanism that automates reconnection, which killed the "it worked yesterday" mystery that had been annoying us for days.

## What actually surprised us

**Time-based positioning works well enough.** No encoders meant we went in expecting the angle tracking to be basically useless. Turns out, parking before every sequence kept it accurate enough to reliably aim at a specific desk. Not millimeter precision, but foam missile precision is fine.

**The `devices` mount is necessary but not sufficient.** The permission error was confusing precisely because the device was clearly visible inside the container. The udev rule is the bit most tutorials quietly skip.

**The coffee rule was never the same after this.** Once the launcher was wired to the pipeline, broken builds suddenly became a lot more motivating to fix.

<style>
.gh-card {
  display: block;
  border: 1px solid #d0d7de;
  padding: 16px;
  margin-top: 2em;
  border-radius: 6px;
  text-decoration: none !important;
}
.gh-card:hover { border-color: #8c959f; }
.gh-card,
.gh-card *,
.md-content .gh-card,
.md-content .gh-card * { text-decoration: none !important; }
.gh-card__head { display: flex; align-items: center; gap: 8px; }
.gh-card__head svg { flex-shrink: 0; fill: #1f2328 !important; }
.gh-card__repo { font-weight: 600; color: #1f2328 !important; }
.gh-card__desc { margin: 8px 0 0; color: #59636e !important; font-size: 14px; }
[data-theme=dark] .gh-card { border-color: #30363d; }
[data-theme=dark] .gh-card:hover { border-color: #6e7681; }
[data-theme=dark] .gh-card__head svg { fill: #e6edf3 !important; }
[data-theme=dark] .gh-card__repo { color: #e6edf3 !important; }
[data-theme=dark] .gh-card__desc { color: #8b949e !important; }
</style>
<a class="gh-card" href="https://github.com/guillaumedelre/dream-cheeky-thunder" target="_blank" rel="noopener noreferrer">
  <span class="gh-card__head">
    <svg width="20" height="20" viewBox="0 0 24 24" aria-hidden="true"><path d="M12 .297c-6.63 0-12 5.373-12 12 0 5.303 3.438 9.8 8.205 11.385.6.113.82-.258.82-.577 0-.285-.01-1.04-.015-2.04-3.338.724-4.042-1.61-4.042-1.61C4.422 18.07 3.633 17.7 3.633 17.7c-1.087-.744.084-.729.084-.729 1.205.084 1.838 1.236 1.838 1.236 1.07 1.835 2.809 1.305 3.495.998.108-.776.417-1.305.76-1.605-2.665-.3-5.466-1.332-5.466-5.93 0-1.31.465-2.38 1.235-3.22-.135-.303-.54-1.523.105-3.176 0 0 1.005-.322 3.3 1.23.96-.267 1.98-.399 3-.405 1.02.006 2.04.138 3 .405 2.28-1.552 3.285-1.23 3.285-1.23.645 1.653.24 2.873.12 3.176.765.84 1.23 1.91 1.23 3.22 0 4.61-2.805 5.625-5.475 5.92.42.36.81 1.096.81 2.22 0 1.606-.015 2.896-.015 3.286 0 .315.21.69.825.57C20.565 22.092 24 17.592 24 12.297c0-6.627-5.373-12-12-12"/></svg>
    <span class="gh-card__repo">guillaumedelre/dream-cheeky-thunder</span>
  </span>
  <span class="gh-card__desc">FastAPI + Docker + PyUSB — HTTP control for the Dream Cheeky Thunder USB missile launcher. Pull requests welcome, especially if you have a better angle calibration approach.</span>
</a>
