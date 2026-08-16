# KittyV1 Attitude Viewer

Browser tool for testing and visualising the KittyV1 flight computer's serial
telemetry: connects over Web Serial, decodes the wire format, and renders the
vehicle's attitude in 3D.

**Live: https://iamraging123.github.io/kittyv1-attitude-viewer/**

Chrome or Edge on desktop — Web Serial is not implemented in Firefox or Safari.
Nothing is uploaded; the page talks to the serial port and nothing else.

## What it decodes

Rows are identified by field count, which is unambiguous across the three
`TELEMETRY_MODE` settings in `config.h`:

| Fields | Mode | Contents |
|---|---|---|
| 15 | 0 — flight | `ms tilt roll azi alt vv aup lat lon fix nsat loop_us ovr tmo flt` |
| 8 | 1 — raw IMU | `ms ax ay az gx gy gz temp` |
| 5 | 2 — raw mag | `ms mx my mz \|B\|` |

Lines starting with `#` are meta — headers, `STATS`, and event records
(`# <ms> <I/W/E> <NAME> v=… n=… first=…`). Event values are annotated with
their units, so `MAG_REJECT v=1111` is shown as `|B| = 111.1 uT`.

## Orientation source

- **Flight rows** — the firmware's quaternion is reconstructed from
  `tilt`/`roll`/`azi`. Those three angles are a complete parameterisation of
  SO(3), so this is exact, not an approximation: swing-twist about body X,
  matching `update_telemetry_angles()` in `attitude.cpp`.
- **Raw IMU rows** — orientation is derived from gravity alone (shortest arc
  from measured up onto world +Z). Roll about the nose is unobservable from
  accelerometer data, so it reads zero. Useful for checking `IMU_REMAP_SIGN`.

Body frame is +X out the nose; world Z is up.

## Buttons

`dump log (l)` and `header (h)` send the firmware's single-character debug
commands. `CSV` downloads the captured flight rows. `Demo` drives the render
with synthetic motion so you can check the page without hardware attached.
