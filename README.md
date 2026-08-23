# steamvr-fbt-fix

Make lighthouse trackers (Vive / Tundra) show up in SteamVR on Linux when the
headset comes from a **different driver** — Steam Link, ALVR, or WiVRn.

If your trackers work fine under WiVRn but don't appear in SteamVR *at all*, this
is almost certainly your problem, and it's a one-key fix.

Confirmed on CachyOS, SteamVR 2.17.7, Quest 2 over Steam Link, Vive Tracker 3.0 +
2 base stations.

## The cause

SteamVR only finishes activating devices belonging to the driver that provides the
HMD. With Steam Link the HMD comes from `driver_vrlink`, so `driver_lighthouse`
never completes and your trackers stay invisible.

The driver is **not** failing to load, and this is **not** a udev, pairing, or
firmware problem. The lighthouse driver loads, opens every dongle, negotiates with
the tracker and reads its firmware config — only the final activation handshake is
withheld. Chasing permissions here wastes hours.

## Quick fix

With SteamVR **fully closed** (it rewrites this file on exit), merge into
`~/.local/share/Steam/config/steamvr.vrsettings`:

```json
"steamvr" : {
   "activateMultipleDrivers" : true
},
"driver_lighthouse" : {
   "enable" : true
}
```

Then start SteamVR and **connect your headset**. Trackers only activate once an
HMD is present.

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
SteamVR overwrites the edit on exit), checks the dongles are present, warns if
WiVRn or monado is holding them, and in `--check` mode detects the failure
signature directly.

Requires `bash`, `python3`, and `lsusb`.

## Verifying

Every `started activation` must have a matching `finished adding`:

```sh
grep -E "Driver 'lighthouse' (started activation|finished adding)" \
  ~/.local/share/Steam/logs/vrserver.txt
```

**Broken** — starts, never finishes:

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

`BOOTSTRAPPED` / `CALIBRATED` mean real optical lock, not just enumeration.

> **Ignore `Driver lighthouse has no suitable devices.`** It appears at startup even
> on a fully working system — it only means that driver supplies no headset. It
> sends a lot of people down the wrong path.

## Then: space calibration

Streaming headsets track inside-out, so trackers arrive in a different coordinate
space and will float or drift even while tracking perfectly. Run
[OpenVR-SpaceCalibrator](https://github.com/hyblocker/OpenVR-SpaceCalibrator)
(`openvr-space-calibrator-linux` on the AUR) once per room setup.

It worked out of the box here — no configuration needed. One wrinkle: installing it
**registers a new OpenVR driver with SteamVR**, so SteamVR must be restarted before
it shows up. If SteamVR was already running, restart it and reconnect your headset
(with Steam Link that means reconnecting from inside the headset too). Nothing is
broken if it doesn't appear immediately — it just hasn't been picked up yet.

`motoc` does **not** apply here — it calibrates against WiVRn/monado, not SteamVR.

## Worth knowing

- **Don't run WiVRn and SteamVR at once.** Two processes can't own the same
  lighthouse dongles. `systemctl --user stop wivrn` first.
- **Plug dongles in before SteamVR starts.** It enumerates receivers once at init
  and never picks up hotplugged ones.
- **Tracker roles are optional.** VRChat calibrates unassigned trackers itself, so
  `Not autobinding role for device path: …` is harmless.
- **A one-second `Transition to protocol version 0/1` loop** is the secondary tell —
  the radio link resetting because the device never attached.

## Playspace mover (optional)

Not required for tracking, but part of a complete FBT setup. **OVR Advanced
Settings** runs fine under Proton, and applying the playspace mover binding to it
works.

Configure it in **desktop mode**, not from inside VR — in-headset interaction with
its panel was unreliable, while the desktop window worked perfectly. That part is
the useful finding.

The choice of OVRAS itself is just familiarity carried over from Windows. Native
Linux options exist and weren't seriously attempted here, so read this as *a* route
that works rather than the recommended one. (The binding also didn't turn up among
those the SpaceCalibrator package registers, but that wasn't investigated far
enough to call it a limitation.)

## More

[`docs/full-writeup.md`](docs/full-writeup.md) has the full diagnosis with raw log
excerpts, plus the theories that turned out to be wrong and why.

## License

MIT — see [LICENSE](LICENSE). Use it, change it, ship it.
Issues and PRs welcome.
