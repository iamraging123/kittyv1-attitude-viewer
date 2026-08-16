# KittyV1 Ground Station

Browser ground station for the KittyV1 rocket flight computer. Connects to the
board over Web Serial, decodes the telemetry wire format, and presents it as a
3D attitude view, cockpit instruments, a ground track, and time-series plots.

**Live: https://iamraging123.github.io/kittyv1-attitude-viewer/**

Chrome or Edge on desktop — Web Serial is not implemented in Firefox or Safari,
and is unavailable inside sandboxed iframes. Nothing leaves the machine; the
page talks to the serial port and nothing else.

## Tabs

| Tab | Contents |
|---|---|
| FLIGHT | 3D orientation · attitude/altimeter/VSI gauges · satellite map, with a data strip beneath |
| KALMAN | altitude, vertical velocity, and world-up acceleration vs time |
| ATTITUDE | tilt / roll / azimuth, body angular rates, quaternion, raw IMU |
| EVENTS | event records with decoded values, sticky fault bits, loop health |

The serial console is a scrollable strip along the bottom, present on every
tab, collapsible.

## Map

Satellite imagery is Esri World Imagery, which needs no API key. The slippy-tile
renderer is hand-rolled rather than Leaflet, so the page stays a single
self-contained file. On HiDPI displays it fetches one zoom level deeper and
draws at half size, so imagery lands at native device resolution instead of
being upscaled.

Scroll or use `+`/`−` to zoom, drag to pan (which releases `follow`). The pad is
a white cross, the track a blue line, the vehicle a blue dot. Range and bearing
from the pad are in the data strip.

Tiles need a network connection. With no signal the map degrades to black with
the track, pad and scale bar still drawn — position data is unaffected, only
the imagery is missing.

## 3D view and STL

Renders with raw WebGL — no external library, so the page stays self-contained
and depth-buffers rather than sorting polygons, which is what makes arbitrary
STLs practical.

Drop in your own airframe with **STL**. Binary and ASCII are both handled;
binary is detected by exact byte-length match. Normals are recomputed from the
geometry because the ones stored in STL files are routinely wrong. The mesh is
centred on its bounding box and normalised to fit.

CAD exports rarely put the nose on the same axis the firmware uses, so pick the
model's nose axis from the dropdown — it defaults to **+Z**, which is what most
CAD and OpenRocket exports produce. The firmware's own convention is body +X
out the nose. The alignment is built as a proper rotation (determinant +1), not
an axis swap, so it can never silently mirror the model.

## Wire format

Rows are identified by field count, unambiguous across all three
`TELEMETRY_MODE` settings in `config.h`:

| Fields | Mode | Contents |
|---|---|---|
| 15 | 0 — flight | `ms tilt roll azi alt vv aup lat lon fix nsat loop_us ovr tmo flt` |
| 8 | 1 — raw IMU | `ms ax ay az gx gy gz temp` |
| 5 | 2 — raw mag | `ms mx my mz \|B\|` |

Lines starting with `#` are meta: headers, `STATS`, and event records
(`# <ms> <I/W/E> <NAME> v=… n=… first=…`). Event values are decoded into their
real units — `MAG_REJECT v=1111` is shown as `|B| = 111.1 µT`, `BOOT` expands
into named reset causes.

## Orientation

- **Flight rows** — the firmware quaternion is reconstructed by inverting the
  swing–twist decomposition in `attitude.cpp`. Tilt, roll and azimuth fully
  parameterise SO(3), so this is exact, not an approximation (verified to 1e-14
  over 200k random orientations).
- **Raw IMU rows** — orientation comes from gravity alone: the shortest arc
  from measured up onto world +Z. Roll about the nose is unobservable from an
  accelerometer, so it reads zero and the view says so.

Body frame is +X out the nose; world Z is up.

## Instruments

The attitude indicator shows elevation above the horizon (the complement of
tilt-from-vertical) and bank (roll about the nose).

The third gauge is a **vertical speed indicator**, not airspeed: there is no
pitot on this vehicle, so the only speed on the wire is the Kalman filter's
vertical velocity. Its scale is compressed (`|v|^0.55`) because a linear ±200
m/s dial would bury everything below 20 m/s.

## Buttons

`dump log · l` and `header · h` send the firmware's single-character debug
commands. `export CSV` downloads captured flight rows. `Demo` runs a synthetic
boost/coast/recovery profile so the whole panel can be evaluated with no
hardware attached — it deliberately does not touch the loop-health counters,
which must only ever reflect real serial traffic.
