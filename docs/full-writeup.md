# SteamVR full-body tracking on Linux with a non-lighthouse HMD

Solved 2026-08-23 on CachyOS. SteamVR 2.17.7, Quest 2 over Steam Link,
4x Watchman dongles, 4x Vive/HTC Tracker 3, 2x base stations.

## Symptom

Lighthouse trackers do not appear in SteamVR at all. The same trackers work
perfectly under the WiVRn stack.

## Root cause

**SteamVR only completes device activation for the driver that provides the
HMD.** With Steam Link the HMD comes from `driver_vrlink`, so `driver_lighthouse`
is left half-initialised.

This is *not* "the lighthouse driver fails to load" — that was the wrong theory.
The driver loads fine and does real work: it opens every dongle, negotiates with
the tracker, reads its firmware config, and parses `lighthousedb.json`. What
never happens is the final activation handshake.

### How to recognise it in the log

`~/.local/share/Steam/logs/vrserver.txt` — the signature is an activation that
starts and never finishes:

```
[Info] - Driver 'lighthouse' started activation of tracked device with serial number 'LHR-4177671B'
                      ... and no matching "finished adding tracked device" line
```

Compare a healthy driver, which always pairs the two:

```
[Info] - Driver 'vrlink' started activation of tracked device with serial number 'VRLINKHMDQUEST2'
[Info] - Driver 'vrlink' finished adding tracked device with serial number 'VRLINKHMDQUEST2'
```

The secondary tell is the tracker's radio link resetting once per second,
forever, because the device never attaches:

```
lighthouse: LHR-4177671B: Transition to protocol version 0
lighthouse: LHR-4177671B: Protocol versioning supported, requesting 1
lighthouse: LHR-4177671B: Transition to protocol version 1
   (repeats every ~1s indefinitely)
```

`Driver lighthouse has no suitable devices.` also appears at startup. That one is
a red herring on its own — it just means the driver provides no HMD, and it shows
up even in a working configuration.

## Fix

With **SteamVR fully closed** (it rewrites this file on exit), in
`~/.local/share/Steam/config/steamvr.vrsettings`:

```json
"steamvr" : {
   "activateMultipleDrivers" : true
},
"driver_lighthouse" : {
   "enable" : true
}
```

`activateMultipleDrivers` is the one that matters. `driver_lighthouse.enable` is
belt-and-braces — it was already defaulting on here.

## Verifying

Trackers activate **only once an HMD is actually connected**. Launching SteamVR
with the headset off shows the trackers still stuck in the pending state, which
looks identical to the original bug. Connect the headset before concluding
anything.

A healthy result looks like this:

```
Driver 'lighthouse' finished adding tracked device with serial number 'LHR-4177671B'
Driver 'lighthouse' finished adding tracked device with serial number 'LHB-3069030F'
Driver 'lighthouse' finished adding tracked device with serial number 'LHB-A1EB994F'
lighthouse: LHR-4177671B C: ----- BOOTSTRAPPED base 3069030F (immediate) distance 2.95m -----
lighthouse: LHR-4177671B C: ----- CALIBRATED base 3069030F at pitch 40.94 deg roll 4.59 deg -----
lighthouse: LHR-4177671B C: ----- RELATIONSHIP bases 3069030F <-> a1eb994f distance 4.30m, angle 169.61 deg -----
```

Base stations (`LHB-*`) attach as tracked devices too, and `BOOTSTRAPPED` /
`CALIBRATED` / `RELATIONSHIP` mean real optical lock, not just enumeration.

## Remaining step: space calibration

Steam Link tracks inside-out, so the trackers arrive in a completely different
coordinate universe — they will appear offset or drifting even when tracking
perfectly. Run **OpenVR-SpaceCalibrator** (`openvr-space-calibrator-linux`, AUR)
once per room setup.

It worked out of the box here — no configuration needed. One wrinkle: installing it
**registers a new OpenVR driver with SteamVR**, so SteamVR must be restarted before
it shows up. If SteamVR was already running, restart it and reconnect your headset
(with Steam Link that means reconnecting from inside the headset too). Nothing is
broken if it doesn't appear immediately — it just hasn't been picked up yet.

Note `motoc` does *not* help here — it calibrates against monado/WiVRn, not a
SteamVR session. (`~/.config/motoc/last.json` holds the WiVRn-side calibration:
`WiVRn HMD` -> `LHR-E189B77A`.)

## Things that turned out not to be the problem

- **udev / permissions / pairing.** Provably fine — WiVRn runs Valve's lighthouse
  binary in-process (`use-steamvr-lh: true` in `~/.config/wivrn/config.json`) and
  the trackers work there.
- **WiVRn holding the dongles.** Real hazard in general, since two processes
  cannot own the same lighthouse USB devices — but WiVRn was not running here.
- **Dongles hotplugged after SteamVR started.** SteamVR enumerates receivers once
  at init, so this *is* a real failure mode, but all four were present at startup.
- **Tracker role assignment.** Not required. VRChat's own FBT calibration handles
  unassigned trackers. SteamVR logs `Not autobinding role for device path:
  /devices/htc/vive_trackerLHR-4177671B`, which is harmless.

## Script

Installed as `steamvr-fbt-fix` (see the README).

```
steamvr-fbt-fix            # patch settings (stops SteamVR first)
steamvr-fbt-fix --launch   # patch, launch SteamVR, watch the log for activation
steamvr-fbt-fix --check    # report state, change nothing
```

It backs up `steamvr.vrsettings` before editing, refuses to edit while SteamVR is
running (since SteamVR would overwrite it), checks for the dongles, warns if
WiVRn/monado is holding them, and in `--check` mode detects the
started-but-never-finished signature directly.

## The other option

`xrizer` (already installed, at `/opt/xrizer`) reimplements OpenVR on OpenXR, so
OpenVR games run through WiVRn with no SteamVR at all — FBT support landed in
v0.4. The LVRA wiki currently marks SteamVR as "not recommended" for trackers on
Linux. Given the above, that guidance may be worth revisiting: the fix is a
single settings key.
