# Rain simulator: walk or run?

A browser app that answers "do you get less wet walking or running in the rain?" by measuring how much rain a polygon mannequin intercepts at each speed.

## Layout

- `index.html` is the whole app: HTML, CSS and JavaScript in one file. No dependencies, no build step, no modules. Keep it that way; portability across OSes and CPU architectures is the point of the project.
- Rendering is plain WebGL2 (no three.js). The plot is hand-drawn on a 2D canvas.

## How the measurement works

- Axes: x forward (direction of travel), y up, z to the body's right. Units are metres, seconds.
- Rain: Marshall-Palmer drop size distribution, 29 size classes (0.2 to 6 mm), Atlas et al. (1973) terminal velocity. The distribution is rescaled so ground flux equals the chosen rain rate exactly. Wind moves all drops horizontally at wind speed.
- In the body's frame, every drop of one size class travels in one direction, so that class delivers `water content x relative speed x visible projected area`.
- Visible area comes from a precomputed table (`buildTable`): the mesh is rendered orthographically from 1,200 directions (25 zenith x 48 azimuth, upper hemisphere) with a triangle ID per pixel, and pixels are counted per triangle. The depth buffer handles self-shadowing.
- `evaluate()` bilinearly interpolates that table, so speed, wind, rain, distance and height changes are instant. Only pose or lean changes rebuild a table (about a second, with a progress bar). Tables are cached per `pose|lean`, at most 4.
- Height scales areas by `(H / 1.75)^2`; it does not rebuild the table.
- The rain streaks on screen are visual only and play no part in the numbers.

## Mannequin

- Built procedurally in `buildMesh` from tapered capsules (about 3,300 triangles). Poses are static (`POSES`: stand, walk, run); there is no gait animation by design.
- "Auto" pose uses the walk stride below 2.2 m/s and the run stride above, which puts a step in the curve at that speed.
- Regions for the breakdown: Head, Torso, Arms, Legs, Feet.
- Frontal area is about 0.5 m^2, slimmer than a clothed adult.

## Deliberately not modelled

Gait motion, ground splash, runoff and clothing absorption, airflow around the body, gusts.

## Results, sharing and errors

- The driest speed is the minimum of the sweep (0.05 m/s steps). It shows as "Fastest" when that minimum is less than 0.5% below the value at top speed.
- If the WebGL context is lost, the page shows a message (`showError`) and reloads when the context returns. The URL keeps the settings; the camera view is not kept.
- If a measurement throws, `ensureTables` clears the results (`clearResults`) and shows the same message overlay with a "Reload" button. A later measurement that succeeds hides the message.
- The plot marks a finite driest speed with a full-height line, a flag and a ring. The marker uses the text colour, not a palette colour.
- The address bar follows the settings (`syncAddress`): `shareURL()` writes each setting that differs from `DEFAULTS` as a URL parameter and keeps `selftest`. The "Copy link" button copies that URL. The camera view is not part of the URL.

## Running and testing

- Serve the folder: `python3 -m http.server 8765 --bind 127.0.0.1` from this directory, then open `http://localhost:8765/index.html` in a browser that supports WebGL2.
- `index.html?selftest` appends a `SELFTEST {...}` JSON line to the side panel: measured areas per pose, a sample of the sweep, and a sphere check. The sphere (radius 0.5 m) must measure close to pi/4 = 0.785 m^2 from every direction; it currently reads 0.782 to 0.784.
- Reference numbers for defaults (100 m, 10 mm/h, no wind): about 41.7 mL at 1.4 m/s, about 22.4 mL at 10 m/s. With a 3 m/s tailwind the driest speed is about 5.45 m/s.
- URL parameters set starting values: `v`, `dist`, `R`, `wx`, `wz`, `lean`, `H`, `pose` (for example `?v=4.5&wx=3&R=25&pose=run`).
- The address bar follows the settings about 250 ms after a change. In tests, wait for that delay before reading `location.href`.
- `window.rainSim` exposes the state object `S`, `sweep`, `ready`, `areas(pose)` and `sphereTest()` for tests.
- For browser automation (for example Playwright), use the same URL. If the automation tool runs in a container, `localhost` points at the container; use the host's address instead (for example `http://host.docker.internal:8765/index.html` with Docker). Wait for `window.rainSim.ready` before reading results. Use real mouse input (`page.mouse`) for the plot and the 3D canvas; they call `setPointerCapture`, which rejects synthetic pointer events.
- Without an automation tool, a headless Chromium-based browser (Chrome, Chromium, Edge) works for DOM dumps and screenshots: `<browser> --headless=new --virtual-time-budget=60000 --dump-dom <url>`. The name of the executable depends on the browser and the operating system.

## Conventions

- Chart colours follow the dataviz skill's dark palette (categorical slots 1 to 5 for the regions, one-hue blue ramp for the heat map). Re-run its palette validator if they change.
- The heat map scale is anchored to walking pace (1.4 m/s) so that changing speed visibly changes the body's colour.
- "Reset all" restores `DEFAULTS` and the camera, not URL-supplied values.
- Each setting has its own reset button (`.rst`, built in JavaScript from `SETTING_INPUTS`). The button shows only while the setting differs from its `DEFAULTS` value. A new setting must be added to `DEFAULTS`, `SETTING_INPUTS`, `syncInputs`, the `bind` calls and the URL parser at the end of the script.
