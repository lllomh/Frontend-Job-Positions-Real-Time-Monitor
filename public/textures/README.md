# Globe texture sources

The globe loads these files locally; no external texture requests are made at runtime.

New textures are copied without modification from the Three.js r160 examples:

- `earth-day.jpg`: https://github.com/mrdoob/three.js/blob/r160/examples/textures/planets/earth_atmos_2048.jpg
- `earth-clouds.png`: https://github.com/mrdoob/three.js/blob/r160/examples/textures/planets/earth_clouds_1024.png
- `earth-specular.jpg`: https://github.com/mrdoob/three.js/blob/r160/examples/textures/planets/earth_specular_2048.jpg

Upstream repository license: https://github.com/mrdoob/three.js/blob/r160/LICENSE

Existing assets retained: `earth-night.jpg`, `earth-topology.png`, and `moon_1024.jpg`.
The scene uses a fixed artistic sunlight direction and compressed lunar distance; it is not a live astronomical simulation.
