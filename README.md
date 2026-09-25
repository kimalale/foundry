# Foundry

A CAD bench where every part is a **signed distance field**: a function that tells any point in space how far it is from the surface. You type a part, and it is cast in machined metal, drawn on a cyanotype engineering sheet, weighed, and exported as an STL.

Open `index.html` in a browser (WebGL2). No build step, no dependencies.

```js
const d = param('diameter', 36, 20, 60);          // becomes a live slider
cyl(d, 18).round(1.5)
  .cut(cyl(4, 24).move(d / 2 + 0.8, 0, 0).polar(14), 1.2)   // fluted grip, blended
```

- **Code**: solids (`box cyl sphere cone torus hex gyroid`), booleans with optional blends (`.add .cut .and`), placement (`.move .rot .mirror .scale`), patterns (`.polar .array`) and forms (`.stretch .round .shell .twist`). The full list is under **Reference**. The last line is the part.
- **Sketches**: 2D profiles (`rect circle ngon polygon arc text`) with the same booleans, blends and patterns, plus `.offset()`. Finish a sketch with `.extrude(h, edgeRadius)` or `.revolve()` (x is the radius, y the height). `text()` uses a built-in single-stroke font, ready for engraving.
- **`param()`** makes a slider. Dragging it changes only numbers, never the shader: every number in the part lives in a uniform array, so the GPU program depends only on the shape of the tree.
- **Section** (`S`): cut along X, Y or Z. The cut face shows the field itself, coloured by how deep each point sits inside the material, with a contour line every step. It reads as a wall-thickness map.
- **Hover the floor** and a ring appears: the largest ball around that point that touches nothing. That radius is exactly how far a ray is allowed to leap, and it is the whole trick behind the renderer.
- **Measure** (`M`): click two points on the part.
- **Drawing**: third-angle top, front and right views plus an iso, auto-dimensioned, with a title block (material, mass, volume, scale). Export as PNG.
- **Export STL** (`E`): meshed with surface nets at Draft, Fine or Max resolution.
- **Commissions**: drawings to match. Your part is overlaid in red and graded by volume overlap, aligned by envelope centre. Progress is saved in the browser.
  - **Tier I**: seven dimensioned drawings at 98.5%, plus one more that stays locked until the other seven are stamped.
  - **Tier II**: six sketch parts (a nameplate, a cam, a gasket, a bottle, a curved slot, a ring spanner). It opens after three Tier I stamps. The drawings carry **no dimensions**, only a grid and a datum dot, and the tolerance tightens to 99.3%. Hover any view on the sheet to read millimetres from its datum.
- **Parts bin**: a knob, a bracket, a sprocket, a twisted vase, a gyroid lattice sphere, a heat sink, and three sketch parts (a revolved goblet, an engraved tag, a spoked wheel).
- **Share** copies a link that carries your code. Shared code is JavaScript, so a shared link never runs until you press **Run it**.

Keys: `1–4` views · `F` frame · `O` ortho · `S` section · `M` measure · `E` export · double-click to orbit around a point.

**Self-test**: open `index.html#selftest`. It checks volumes and envelopes against the analytic answers, confirms the mesher produces a closed, outward-facing surface, renders every preset on the GPU and compares the silhouette pixel by pixel with the JavaScript evaluator, and grades every commission against itself and against deliberately wrong parts.

See `LEARNING.md` for how it works.
