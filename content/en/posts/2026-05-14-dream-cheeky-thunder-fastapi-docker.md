---
title: "Controlling a USB missile launcher over HTTP with FastAPI and Docker"
aliases: ["/2017/02/21/controlling-a-usb-missile-launcher-over-http-with-fastapi-and-docker/"]
date: 2026-05-14
lastmod: 2026-11-18
tags: [python, docker]
format: experiment
featureimage: "https://raw.githubusercontent.com/guillaumedelre/dream-cheeky-thunder/develop/docs/Dream-Cheeky-Thunder.jpg"
featureimagecaption: "The Dream Cheeky Thunder, a small USB turret that fires foam missiles"
images: ["https://raw.githubusercontent.com/guillaumedelre/dream-cheeky-thunder/develop/docs/Dream-Cheeky-Thunder.jpg"]
description: "How we wired a USB foam missile launcher to the CI pipeline — and what Docker, udev, and WSL2 had to say about it."
---

{{< lead >}}
The rule was simple: whoever breaks the CI build owes the team a coffee. It worked fine for a while. Then someone suggested we needed something with more immediate feedback, something physical that fires.
{{< /lead >}}

A [Dream Cheeky Thunder][dct] appeared on a desk shortly after. Four foam missiles, a USB cable, and a very clear team consensus: hook it to the cluster, wire it to the build pipeline, and let the CI decide who deserves a volley.

The launcher needed to respond to HTTP calls from anywhere on the network, with no driver or GUI to install and no manual aiming: an endpoint that makes it shoot in the direction of the guilty party's desk.

This is the story of [dream-cheeky-thunder][repo].

## A forum script and a handful of control bytes

Dream Cheeky never published a protocol spec. The launcher speaks raw USB HID, and the only starting point was an old Python script floating around in forum threads. Vendor ID `0x2123`, product ID `0x1010`, and a handful of control bytes that someone had reverse engineered years before.

That was enough. The protocol is simple: send a byte sequence to move the motors, send another to fire. The tricky part is that the launcher has no position feedback: no encoders, no limit switches beyond the physical hard stops at the extremes, so you drive it blind.

## From USB to HTTP

The CI pipeline needed to trigger the launcher over the network. A local script wasn't going to cut it — the launcher had to be reachable from any machine on the cluster, including the build server. So: a REST API.

FastAPI was the obvious choice. The targeting flow from the CI side ends up being three HTTP calls:

```bash
curl -X POST http://localhost:8000/park      # reset to known position
curl -X POST http://localhost:8000/yaw/20    # rotate toward guilty desk
curl -X POST "http://localhost:8000/fire?shots=2"
```

The `/park` call matters more than it looks. Since the launcher has no position feedback, the server estimates the current angle by tracking how long the motors have been running. That estimate drifts. Bumping the hardware, interrupting a command, or the imprecision of time-based tracking itself — they all accumulate. Parking drives the launcher left, then down, against the physical hard stops for a full sweep each, which guarantees alignment regardless of what the server thinks it knows; without it, the aim is a guess.

The full API reference is [in the repo][api]. There's also a web UI if you prefer clicking over `curl`.

## Docker knows nothing about USB

Running this in a Docker container on the cluster was where the fun started: containers don't see USB devices by default.

The `devices` mount in `compose.yaml` exposes the USB bus to the container:

```yaml
devices:
  - /dev/bus/usb:/dev/bus/usb
```

Not enough. First run came back with `USBError: [Errno 13] Access denied`. The device node is there inside the container, but it inherits permissions from the host, and on the host only root can open it by default.

The fix is a udev rule. Drop one file into `/etc/udev/rules.d/`, and udev applies it when the device plugs in: mode `0666`, group `plugdev`. The device node is no longer reserved to root. The rule ships with the project, setup instructions are [in the docs][setup-linux].

## WSL2 made it interesting

Half the team runs Windows with Docker Desktop on WSL2, which puts one more layer between the launcher and the container.

WSL2 has no access to USB devices by default: the Windows kernel holds them, and the `devices` mount alone does nothing because WSL2 doesn't see the hardware. The fix is [usbipd-win][usbipd-win], which forwards the USB device from Windows into the WSL2 kernel over IP. Once that's done, the Linux path works exactly the same: udev rule, `devices` mount, done.

The attachment doesn't survive reboots, though. usbipd v4.2+ can share the device automatically with a policy rule, and `usbipd attach --wsl --auto-attach` re-attaches it whenever it reconnects. Together, they killed the "it worked yesterday" mystery that had been annoying us for days.

## What surprised us

Time-based positioning works well enough. With no encoders, we went in expecting the angle tracking to be nearly useless; in practice, parking before every sequence kept it accurate enough to reliably aim at a specific desk. Not millimeter precision, but foam missile precision is fine.

The other surprise was the permission error. The `devices` mount is necessary but not sufficient, and the error was confusing precisely because the device was clearly visible inside the container. The udev rule is the bit most tutorials quietly skip. As for the coffee rule, it was never the same: once the launcher was wired to the pipeline, broken builds became a lot more motivating to fix.

{{< github repo="guillaumedelre/dream-cheeky-thunder" showThumbnail=true >}}

[dct]: https://web.archive.org/web/20171216061329/http://dreamcheeky.com/thunder-missile-launcher
[repo]: https://github.com/guillaumedelre/dream-cheeky-thunder
[api]: https://github.com/guillaumedelre/dream-cheeky-thunder/blob/develop/docs/api.md
[setup-linux]: https://github.com/guillaumedelre/dream-cheeky-thunder/blob/develop/docs/setup-linux.md
[usbipd-win]: https://github.com/dorssel/usbipd-win
