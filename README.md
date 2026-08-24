# steamvr-fbt-fix

Make lighthouse trackers (Vive / Tundra) show up in SteamVR on Linux when the
headset comes from a **different driver**, such as Steam Link, ALVR, or WiVRn.

If your trackers work fine under WiVRn but do not appear in SteamVR at all, this is
almost certainly your problem, and it is a one key fix.

Confirmed on CachyOS, SteamVR 2.17.7, Quest 2 over Steam Link, Vive Tracker 3.0 and
2 base stations.

## The cause

SteamVR only finishes activating devices belonging to the driver that provides the
HMD. With Steam Link the HMD comes from `driver_vrlink`, so `driver_lighthouse`
never completes and your trackers stay invisible.

The driver is **not** failing to load, and this is **not** a udev, pairing, or
firmware problem. The lighthouse driver loads, opens every dongle, negotiates with
the tracker and reads its firmware config. Only the final activation handshake is
withheld. Chasing permissions here wastes hours.

## Quick fix

With SteamVR **fully closed**, because it rewrites this file on exit, merge into
`~/.local/share/Steam/config/steamvr.vrsettings`:

```json
"steamvr" : {
   "activateMultipleDrivers" : true
},
"driver_lighthouse" : {
   "enable" : true
}
```

Then start SteamVR and **connect your headset**. Trackers only activate once an HMD
is present.

## Or use the script

```sh
git clone https://github.com/wundervrc/steamvr-fbt-fix
cd steamvr-fbt-fix
install -Dm755 steamvr-fbt-fix ~/.local/bin/steamvr-fbt-fix
```

```
steamvr-fbt-fix --check    # report state, change nothing
steamvr-fbt-fix            # patch settings
steamvr-fbt-fix --launch   # patch, launch SteamVR, watch the log for activation
```

It backs up `steamvr.vrsettings` before editing, stops SteamVR first (otherwise
SteamVR overwrites the edit on exit), checks the dongles are present, warns if WiVRn
or monado is holding them, and in `--check` mode detects the failure signature
directly.

Requires `bash`, `python3`, and `lsusb`.

## Verifying

Every `started activation` must have a matching `finished adding`:

```sh
grep -E "Driver 'lighthouse' (started activation|finished adding)" \
  ~/.local/share/Steam/logs/vrserver.txt
```

**Broken**, starts but never finishes:

```
Driver 'lighthouse' started activation of tracked device with serial number 'LHR-4177671B'
                          (no matching "finished adding" line)
```

**Working:**

```
Driver 'lighthouse' started activation of tracked device with serial number 'LHR-4177671B'
Driver 'lighthouse' finished adding tracked device with serial number 'LHR-4177671B'
lighthouse: LHR-4177671B C: ----- BOOTSTRAPPED base 3069030F (immediate) distance 2.95m -----
lighthouse: LHR-4177671B C: ----- CALIBRATED base 3069030F at pitch 40.94 deg roll 4.59 deg -----
```

`BOOTSTRAPPED` and `CALIBRATED` mean real optical lock, not just enumeration.

> **Ignore `Driver lighthouse has no suitable devices.`** It appears at startup even
> on a fully working system. It only means that driver supplies no headset, and it
> sends a lot of people down the wrong path.

## Then: space calibration

Streaming headsets track inside-out, so trackers arrive in a different coordinate
space and will float or drift even while tracking perfectly. Run
[OpenVR-SpaceCalibrator](https://github.com/hyblocker/OpenVR-SpaceCalibrator)
(`openvr-space-calibrator-linux` on the AUR) once per room setup.

It worked out of the box here with no configuration needed. One wrinkle: installing
it **registers a new OpenVR driver with SteamVR**, so SteamVR must be restarted
before it shows up. If SteamVR was already running, restart it and reconnect your
headset. With Steam Link that means reconnecting from inside the headset too.
Nothing is broken if it does not appear immediately, it just has not been picked up
yet.

`motoc` does **not** apply here. It calibrates against WiVRn and monado, not
SteamVR.

## Playspace mover (optional)

Not required for tracking, but part of a complete FBT setup. OVR Advanced Settings
runs fine under Proton, and applying a playspace mover binding to it works.

Configure it in **desktop mode** rather than from inside VR. In-headset interaction
with its panel was unreliable, while the desktop window worked perfectly. That is
the part worth knowing.

On why OVRAS: playspace mover did not show up in the list of binding options at a
glance, so OVRAS was the familiar fallback from Windows. Adding it to Steam as an
external application might have surfaced it, but that was never tested, and since it
does appear in the Steam overlay in VR that may not be the explanation either. Read
this as one route that works, not as a recommendation over the native options.

## Worth knowing

- **Do not run WiVRn and SteamVR at once.** Two processes cannot own the same
  lighthouse dongles. Run `systemctl --user stop wivrn` first.
- **Plug dongles in before SteamVR starts.** It enumerates receivers once at init
  and never picks up hotplugged ones.
- **Tracker roles are optional.** VRChat calibrates unassigned trackers itself, so
  `Not autobinding role for device path: ...` is harmless.
- **A one second `Transition to protocol version 0/1` loop** is the secondary tell.
  That is the radio link resetting because the device never attached.

## More

[`docs/full-writeup.md`](docs/full-writeup.md) has the full diagnosis with raw log
excerpts, plus the theories that turned out to be wrong and why.

## Provenance

Diagnosed and written by AI (Claude) :3 working from live logs on the machine
described above, then verified end to end on that hardware by its owner. The log
excerpts are real output, not illustrative examples. Everything stated as confirmed
was actually observed working, and anything untested is labelled as such.

## License

MIT, see [LICENSE](LICENSE). Use it, change it, ship it. Issues and PRs welcome.
