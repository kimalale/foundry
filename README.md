# Foundry

A CAD bench where every part is a **signed distance field**: a function that tells any point in space how far it is from the surface. You type a part, and it is cast in machined metal, drawn on a cyanotype engineering sheet, weighed, machined on a simulated CNC mill, and exported as an STL.

**Live: https://kimalale.github.io/foundry/** · or open `index.html` in a browser (WebGL2). No build step, no dependencies.

```js
const d = param('diameter', 36, 20, 60);          // becomes a live slider
cyl(d, 18).round(1.5)
  .cut(cyl(4, 24).move(d / 2 + 0.8, 0, 0).polar(14), 1.2)   // fluted grip, blended
```

- **Code**: solids (`box cyl sphere cone torus hex gyroid`), booleans with optional blends (`.add .cut .and`), placement (`.move .rot .mirror .scale`), patterns (`.polar .array`) and forms (`.stretch .round .shell .twist`). The full list is under **Reference**. The last line is the part.
- **Sketches**: 2D profiles (`rect circle ngon polygon arc text`) with the same booleans, blends and patterns, plus `.offset()`. Finish a sketch with `.extrude(h, edgeRadius)` or `.revolve()` (x is the radius, y the height). `text()` uses a built-in single-stroke font, ready for engraving.
- **Assemblies**: `assembly({ block, shaft, bolts })` gives each named part its own colour and an **explode** slider. The page measures the clearance between every pair of parts within 5 mm, taken straight from their fields. Overlaps are flagged with depth and volume, and a section cut hatches the interference in red.
- **`param()`** makes a slider. Dragging it changes only numbers, never the shader: every number in the part lives in a uniform array, so the GPU program depends only on the shape of the tree.
- **Section** (`S`): cut along X, Y or Z. The cut face shows the field itself, coloured by how deep each point sits inside the material, with a contour line every step. It reads as a wall-thickness map.
- **Hover the floor** and a ring appears: the largest ball around that point that touches nothing. That radius is exactly how far a ray is allowed to leap, and it is the whole trick behind the renderer.
- **Measure** (`M`): click two points on the part.
- **Drawing**: third-angle top, front and right views plus an iso, auto-dimensioned, with a title block (material, mass, volume, scale). Export as PNG.
- **Export STL** (`E`): meshed with surface nets at Draft, Fine or Max resolution.
- **Machine** (`C`): plans a 3-axis CNC job for the part, a flat end mill roughing level by level and then a ball end mill finishing the surface, and cuts it out of a block of stock in real time. Play, pause, change speed or scrub. It reports machining time, material removed, and how much of the part is out of reach from above (side holes and overhangs that would need a second setup).
- **Commissions**: drawings to match. Your part is overlaid in red and graded by volume overlap, aligned by envelope centre. Progress is saved in the browser.
  - **Tier I**: seven dimensioned drawings at 98.5%, plus one more that stays locked until the other seven are stamped.
  - **Tier II**: six sketch parts (a nameplate, a cam, a gasket, a bottle, a curved slot, a ring spanner). It opens after three Tier I stamps. The drawings carry **no dimensions**, only a grid and a datum dot, and the tolerance tightens to 99.3%. Hover any view on the sheet to read millimetres from its datum.
  - **Tier III (fits)**: `given('bushing' | 'hub' | 'plate')` places a fixed part. Build an assembly with a mating part (a running shaft, a keyed shaft, a hex peg) that passes through the check points and keeps its clearance inside the band everywhere.
- **Parts bin**: a knob, a bracket, a sprocket, a twisted vase, a gyroid lattice sphere, a heat sink, and three sketch parts (a revolved goblet, an engraved tag, a spoked wheel).
- **Gallery**: the parts bin and a **community board**. **Submit this part** opens a prefilled GitHub issue on this repo (title `[part] …`, with a share link inside). Open `[part]` issues appear on the board, sorted by 👍 reactions, which serve as votes. The owner moderates by closing issues.
- **Share** copies a link that carries your code.
- **Sandbox**: all code you type, open from a link or load from the gallery runs in a **Web Worker**. It has no access to the page or its storage, and it is stopped after 2.5 s. Only plain data comes back, and every node is validated before the shader compiler sees it.

Keys: `1–4` views · `F` frame · `O` ortho · `S` section · `M` measure · `E` export · `C` machine (`Space` plays or pauses) · double-click to orbit around a point.

**Self-test**: open `index.html#selftest`. It checks volumes and envelopes against the analytic answers, confirms the mesher produces a closed, outward-facing surface, renders every preset on the GPU and compares the silhouette pixel by pixel with the JavaScript evaluator, grades every commission against itself and against deliberately wrong parts, checks the sandbox's isolation and loop timeout, measures known clearances and interference, grades good and bad fits, and confirms the machining plan finds a side hole unreachable. 93 checks in all.

See `LEARNING.md` for how it works.
