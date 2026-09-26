# Learning the Maths Behind Foundry

Commercial CAD stores a part as its boundary: faces, edges and the topology between them (a B-rep). Foundry stores a part as a single function instead, `f(x, y, z)`, which returns the distance from that point to the surface. The distance is negative inside and positive outside. Everything else, including rendering, booleans, fillets, patterns, volume, grading and meshing, is a consequence of that one choice.

## What the maths actually needs

| Concept | What you need | Where it appears in the code |
|---|---|---|
| **Distance to simple shapes** | Pythagoras, `abs`, `max` | `sdBox`, `sdCyl`, `sdCone`, `sdHex` (GLSL `LIB` and JS `jsf`) |
| **Booleans** | `min` and `max` | `u`, `s`, `i` nodes |
| **Blends** | One quadratic | `smin` / `smax` |
| **Transforms** | Move and rotate the *point*, not the shape | `mv`, `rot`, `pol`, `arr`, `twi` |
| **Rendering** | A loop and one inequality | `march()` (sphere tracing) |
| **Volume** | Counting, with a soft edge | `analyseGrid()` |

## The key idea in one sentence

If `f(p) = 5`, then **nothing in the part is within 5 mm of p**, so a ray starting at `p` can safely jump 5 mm. That is **sphere tracing**. `march()` steps along each pixel's ray by whatever the field says until the distance is nearly zero. Hover the floor and the ring shows the ball for that point.

## Shapes are formulas

- Sphere: `length(p) − r`.
- Box: take `q = |p| − halfSize`. Outside, the distance is the length of the positive parts of `q`. Inside, it is the largest component, which is negative. `sdBox` is those two cases in one line.
- Cylinder: the same idea in 2D, on the pair (distance from the Z axis − r, |z| − h/2).

## Sketches: 2D fields with a third dimension added

A sketch is a distance field in the plane, `f(x, y)`. It gets thickness in two ways:
- **Extrude** (`opExtrude`): treat the 2D distance and the height distance `|z| − h/2` like the two sides of a rectangle, and combine them exactly as `sdBox` does. A rounded edge first shrinks both by `r`, then grows the result back by `r`.
- **Revolve**: ask the sketch about the point `(√(x² + y²), z)`. The distance from the axis *becomes* the sketch's x. Nothing is swept and nothing is meshed.

The 2D shapes:
- **Polygon** (`sdPoly`): the distance is to the nearest edge. The sign comes from counting how many edges a horizontal ray crosses: an odd count means inside. The points live in the uniform array, and the shader reads them in a loop.
- **Arc**: a ring `|length(p) − r| − w/2`, except past the ends of the sweep, where the distance is measured to the end points instead. That gives the round ends.
- **Text** (`text()`): each glyph is a few polylines on a 4 × 6 grid. The field is the distance to the nearest segment minus half the stroke, like a pen of fixed width.

## Booleans are min and max

- **Union**: a point is as close to A∪B as it is to the nearer of the two, so the result is `min(a, b)`.
- **Intersection**: `max(a, b)`.
- **Cut**: A minus B is A intersected with *not-B*, and not-B is just `−b`. The result is `max(a, −b)`.

These are exact on the outside of a union and approximate elsewhere, but they never *overestimate* the distance, and that is all sphere tracing needs.

## A fillet is a soft minimum

`smin(a, b, k)` equals `min` when the two distances differ by more than `k`. When they are close, it dips below both by up to `k/4`, which fills the inside corner with a smooth blend. That is the whole implementation of `.add(b, 4)`. Blended fillets are notoriously hard in boundary CAD. Here they cost one line.

## Transforms move the question, not the part

To draw a shape moved by `t`, ask the original shape about `p − t`. To rotate it, ask about the inverse-rotated point. Look at `rot()` in the code: it stores the *transpose* of the rotation matrix for exactly this reason.

That idea buys **patterns for free**:
- `.polar(n)` folds every angle into one wedge (`opPolar`), so a single tooth answers for all n. Twelve teeth cost the same as one.
- `.array(n, v)` snaps the point back by a whole number of steps, clamped to `0 … n−1`.
- `.mirror('x')` replaces x with |x|.
- `.stretch()` removes a slab from the middle of space, so a cylinder becomes a slot.

This is why `teeth` can be a slider: the tooth count is just a number fed to `opPolar`.

## When the field lies: Lipschitz bounds

Sphere tracing is safe only if the field never claims more room than there is. Formally, the field's slope (its *Lipschitz constant*) must be at most 1. `.twist()` breaks this rule: points far from the axis are swept sideways, which stretches the field. `twist()` computes the worst-case stretch, `√(1 + (k·r)²)` at the part's outer radius r, and divides by it. Without that fix, the renderer overshoots and the vase grows holes. `march()` also takes 0.85 of each step as a margin for the blends.

## One tree, two compilers

The part is compiled twice:
1. **GLSL** (`emitGLSL`), for the GPU renderer. Every number becomes `U[i]`, a slot in a uniform array, so the shader's source depends only on the *shape* of the tree. Change a number and the page only uploads new uniforms. Add a node and it compiles a new shader in the background (`KHR_parallel_shader_compile`) while the old one keeps drawing.
2. **JavaScript closures** (`jsf`), for everything the CPU needs: hover, measuring, volume, grading and meshing.

The self-test renders each preset on the GPU and ray-casts the same view in JavaScript, then checks that the two silhouettes agree pixel for pixel. That keeps the two compilers honest.

## Volume, envelope and the section colours

`analyseGrid()` samples the field on a grid of about 64 cells along the longest side.
- **Volume**: each sample counts `clamp(0.5 − f/h, 0, 1)` of a cell, a soft inside test that is exact for a flat surface through the cell. The results land within about 0.1% of the analytic volumes for a cube, a cylinder and a sphere (see the self-test).
- **Envelope**: wherever the field changes sign between two neighbouring samples, linear interpolation places the surface to a fraction of a cell. The page keeps the extremes along each axis.
- **Deepest point**: the most negative sample. It is the radius of the largest ball that fits inside the part, and it sets the top of the section's colour scale.

## Measuring an undimensioned drawing

The Tier II sheets have no dimensions. The cursor still knows each view's scale (pixels per mm, printed as SCALE in the title block) and its datum, the lower-left corner of the part's envelope. It converts screen position back to millimetres, which is exactly what an inspector does with a ruler on a print.

## Grading a commission

Both parts are sampled on a shared grid, after moving yours so the two envelope centres coincide. The score is **intersection over union** (IoU): shared volume divided by combined volume, using the same soft occupancy. A 1 mm oversize bore on the spacer scores 95.6%, and five holes instead of six on the flange score 91.4%. 98.5% is tight enough to catch mistakes and loose enough to forgive the sampling. Tier II asks for 99.3%, which a 0.5 mm error in the cam's lobe misses (97.6%).

## Meshing: surface nets

`meshSurface()` puts one vertex in every grid cell the surface passes through, at the average of the points where the field crosses zero along the cell's edges. It then nudges each vertex onto the surface with one Newton step along the gradient. Every grid edge that crosses the surface becomes a quad joining the four cells around it, wound so that it faces outward. The result is closed by construction; the self-test checks that every edge is shared by exactly two triangles, and that the signed volume is positive and matches the sphere's.


## Assemblies: clearance comes free with the field

For two solids A and B, **the gap between them is the smallest value B's field takes anywhere on A's surface**. The page puts points on each part's surface (the surface-nets vertices, each nudged onto the surface by two Newton steps), evaluates the other part's field at every one, and takes the minimum. A positive minimum is the clearance. A negative one is the depth of the overlap, and a grid count of the points inside both parts gives its volume. `gapBetween()` is under ten lines. A 0.2 mm running fit measures 0.200.

The exploded view doesn't move anything either: each part's function is sampled at `p − offset`, and the offsets are uniforms, so the slider never recompiles the shader.

## Machining: what a 3-axis mill can see

A 3-axis mill only reaches down from above, so the part is reduced to a **heightmap**: for every column (x, y), the highest solid point, found by ray marching straight down through the field (`planMill()`, step 1).

- **Tool compensation** (`toolComp`): a flat cutter of radius r may lower its tip at (x, y) only to the tallest heightmap value inside its footprint disc. For a ball cutter, each neighbour's height is raised by `√(r² − d²) − r`, the shape of the ball. This is a *max-filter*, the mirror image of the soft min used for fillets.
- **Roughing** rasters a flat end mill at stepped depths and stays 0.3 mm above the compensated surface. **Finishing** rides a ball end mill along the ball-compensated surface. A pass is kept only if it removes material, which the planner checks by simulating each pass as it goes.
- **Simulation**: the stock is another heightmap. Every tool position lowers the cells under the cutter to the cutter's shape. On screen, the stock is a heightfield raymarched from a floating-point texture. A heightfield isn't a true distance (a tall wall beside a low floor fools it), so the renderer steps cautiously there.
- **Undercuts**: whatever the finished stock holds beyond the part's true volume is material the tool could not reach. A side hole through a 20 mm block reports about π·3²·20 ≈ 565 mm³.

## The sandbox

Share links and the gallery run other people's code. The whole language is written as one function, `DSL()`, which runs in the page for trusted presets and is also turned into source text for a **Web Worker**. The worker has no DOM and no page storage, and the page kills it if it runs longer than 2.5 s. It returns only JSON: a tree of nodes. `validNode()` then checks every node's type, field names and the finiteness of every number *before* the GLSL compiler sees it, because the compiler writes numbers into shader source.

## Things worth trying

- Load **Lattice sphere**, cut along Y, and drag `cell`. That gyroid infill is a single line: `sphere(48).and(gyroid(cell, wall))`. In boundary CAD it would mean thousands of faces.
- Load **Knob** and set `.cut(grip, 1.2)` to `0`, then to `4`. Watch how far the soft cut spreads.
- Try `box(20).add(sphere(24).move(0, 0, 12), 6)` and increase the blend until the two shapes melt into one.
- Load **Twisted vase**, remove the `* inv` scaling in `jsf`'s `twi` case (and `${L(n.a[1])}` in GLSL), and watch the renderer fail. That is the Lipschitz bound at work.
- Load **Pillow block (assembly)**, push `shaft Ø` past `bore Ø`, and cut along Y: the red ring is the interference.
- Press **Machine** on the **Knob**. The rounded top edge and the skirt underneath create an overhang, and the report tells you how much a second setup would have to remove.
- Cut the **Bracket** along X and look at the colours in the fillet. The blended corner is visibly the thickest region.
