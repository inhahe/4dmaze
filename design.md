# 4D Maze — Design

The features and architecture of the project. Keep this current when either
changes.

Everything lives in **`4d-maze.html`** — one self-contained file, no build step,
no dependencies, opened directly from disk. Alongside it:

| File | Role |
|---|---|
| `4d-maze.html` | The whole application, five layers (below). |
| `test-core.js` | 96 headless tests of layers 1–3, extracted from the HTML. |
| `test-render.js` | 65 Playwright tests of the real page in a real browser. |
| `diag-view.js` | Visual diagnostics with a *known-correct* expected appearance. |
| `diag-maze.js` | Renders a real maze at full pane resolution, sweeping the near plane. |
| `README.md` | User-facing documentation. |
| `design.md` | This file. |

## Features

- First-person navigation of a 4-D maze: translate along all four axes
  (fwd/back, left/right, up/down, ana/kata) and turn 90° in any of the four
  planes containing forward or the roll plane.
- Rendering by projecting the 4-D maze onto a **3-D retina**, shown
  stereoscopically in four modes: wall-eyed, cross-eyed, red/cyan anaglyph, and
  mono (no stereo).
- **Line-of-sight visibility**: only geometry the player has a clear straight
  4-D line to is drawn, in three settings — *solid walls* (per-point
  hidden-line removal, the default), *line of sight, wireframe* (per-cell only),
  and *x-ray* (everything in range).
- **Wall colouring in eight surface directions** (±X ±Y ±Z ±W), or the older
  four-colour reading by the axis each edge runs along.
- Maze generation: recursive-backtracker perfect maze over the 4-D lattice,
  sizes 3⁴–7⁴, seeded and reproducible, with optional braiding to add loops.
- Goal is the cell farthest from the origin by passage distance; a win overlay
  reports steps taken versus the shortest path.
- A 4-D map overlay: a grid of (Z,W) panels, each showing an (X,Y) slice.
- Controls for FOV, sight range, pane width, stereo separation, ana
  exaggeration, animation speed, map mode, and fusion guides.
- Status readout: current cell, steps, passage distance to goal, visited count.

## The central design decision: where the fourth dimension goes

In 3-D, the retina is a 2-D plane. You project *along* the forward axis, and
right/up become the two image axes. Generalise that up one dimension and the
retina becomes a **3-D volume**: still project along forward, and
right / up / **ana** become the three image axes. So the image is genuinely
three-dimensional, and the natural way to display a 3-D image on a flat screen
is stereoscopically.

Consequences, all of which the code depends on:

- **Ana/kata becomes stereo depth.** Left/right and up/down are on the screen;
  ana/kata is into and out of it.
- **The player's own 3-space slice lands exactly on the zero-parallax plane**,
  and so looks like an ordinary 3-D perspective drawing. Ana recedes behind the
  screen; kata pops out.
- **Occlusion and parallax are carried by different quantities and never get
  confused.** Occlusion is governed by *forward* depth, which is the axis that
  got projected away — so it drives the painter's sort, the depth fade, and
  hidden-line removal. Stereo parallax is governed by ana. Two independent
  depth cues, two independent mechanisms.
- **Ana never occludes anything.** It is drawn as stereo depth, which makes it
  *look* like a front-to-back axis, but in the 4-D visual field it is the third
  axis of the retina — as lateral as left/right and up/down. Two points that
  differ only in ana hide each other exactly as much as two points side by side
  do: not at all. Any "opacity" feature must therefore act along forward only.
  This is the single easiest thing to get backwards in the whole project.

The alternative — projecting 4-D → 3-D by dropping W, then 3-D → 2-D — was
rejected: it makes ana a *shrink* cue that competes with forward distance for
the same visual channel, which is exactly the ambiguity stereo can resolve for
free.

## Layers

The file is divided by banner comments (`LAYER 1` … `LAYER 5`). The dependency
direction is strictly downward: **the maze data knows nothing about navigation,
navigation knows nothing about rendering, and the renderer knows nothing about
mazes.** This is the decoupling the project requires, and `test-core.js` relies
on it — it slices layers 1–3 out of the HTML between the `LAYER 1` and `LAYER 4`
markers and evaluates them standalone, so the markers are load-bearing.

### Layer 1 — `Maze4D` (pure data)

The maze is nothing but *a set of 4-D cells with connections between them*.

- `size: [n,n,n,n]`, row-major `strides`, `count = n⁴`.
- `pass: Uint8Array(count)` — **one byte per cell, bit `a` set = this cell is
  open toward `+a`**. Each passage is stored once, on the lower-coordinate cell.
- API: `idx`, `cellFromIndex`, `inBounds`, `neighbor`, `isOpen(c, axis, sign)`,
  `openFace`, `degree`, `distances(from)` (BFS over passages).
- `generateMaze(size, seed, braid)` — randomized DFS (recursive backtracker)
  with a mulberry32 seeded RNG, then braiding: with probability `braid`, knock
  out one wall of each dead end.

**Invariants (tested):** passages are symmetric; no passage leaves the lattice;
the maze is fully connected; at braid=0 exactly `count − 1` passages exist (a
perfect maze / spanning tree); the same seed and size always give the same
maze.

Storing the *lower* cell's bit is the single place the asymmetry lives —
`isOpen` with a negative sign reads `pass[idx − stride]`. Everything else in the
codebase goes through `isOpen`, so nothing else has to know.

### Layer 2 — `Navigator` (pose and motion)

- `cell: [x,y,z,w]` — integer position.
- `basis[k]` — the **world** direction of local axis `k`, where the local axes
  are `L_RIGHT=0`, `L_UP=1`, `L_FWD=2`, `L_ANA=3`. Starts as the identity.
- `requestMove(localAxis, sign)`, `requestTurn(i, j, dir)`, `pose()`,
  `onArrive(cell)`.
- `pose()` returns `{pos, basis, rot}` where `rot = {i, j, theta}` describes an
  *in-progress* turn, so the renderer can animate smoothly without the
  Navigator ever holding a non-integer orientation.

**Orientation is exact.** A turn is `R_{i,j}(±90°)` applied to the basis, which
for a 90° rotation is just a swap-and-negate of two basis vectors:

    basis[i] ←  basis[j] · s
    basis[j] ← -basis[i] · s

So the basis is always a **signed permutation matrix** with determinant +1 — no
floating-point drift is possible, ever, no matter how many turns. This is why
4000 random turns in the test suite still leave an exact permutation. Continuous
angles exist only in `rot.theta`, which is a display-time interpolation and
never feeds back into state.

Turn planes: (2,0) = yaw, (2,1) = pitch, (2,3) = ana-turn, (0,1) = roll.

Blocked moves trigger a `bump` animation rather than being silently dropped, so
"I pressed a key and nothing happened" is distinguishable from "there is a wall
there."

### Layer 3 — Scene builder (the 4-D → 3-D projection)

`buildScene(maze, nav, goalCell, cfg)` → line segments in **retina space**,
sorted far-to-near.

**Visibility** happens in two stages.

`collectReachableCells` — BFS over *passages* out to `cfg.sightDepth`. This is
only the **candidate** set: it bounds the work regardless of maze size, but on
its own it is not visibility. It walks round corners, so it includes corridors
the player has no sightline to — and because the renderer is a wireframe with
no opaque surfaces, that geometry was drawn straight *through* the wall in
front of it. Using it as the visibility set was the original design and it was
wrong: it meant you could read the layout of the whole neighbourhood from
inside a sealed corridor.

`collectVisibleCells` filters the candidates by an actual **4-D line of sight**.
`losClear(maze, from, to, permissive)` is an Amanatides–Woo grid march one
dimension up: step to whichever axis boundary is nearest, check `isOpen` on the
face being crossed, stop just short of the target (which may itself lie on a
cell boundary and must not block itself).

Two details make it robust:

- **Corner grazes are reported blocked by default.** If two axis boundaries are
  crossed at the same instant the ray is threading a zero-width lattice corner,
  which is not real visibility. Reporting it blocked is the conservative
  direction. The `permissive` flag inverts that for hidden-line removal — see
  below, where the two callers genuinely want opposite answers.
- **Each candidate is tested with a fan of nine rays** — the cell centre plus
  one point pushed 0.36 toward each of the eight faces (`LOS_AIM`). A cell is
  visible if *any* ray gets through. This is what makes the conservative corner
  rule safe: a genuine sightline that merely happens to graze a corner on the
  centre ray is still found by an offset one. It is also what makes a cell
  glimpsed *diagonally* through a doorway correctly visible — the centre-to-centre
  ray for a diagonal pair always hits a corner exactly, so without the fan every
  diagonal would be wrongly culled.

**Visibility is cached per cell** (`visibleCellsFor`). It depends only on which
cell the player stands in — not on where they are mid-step and not at all on
which way they face — so the grid march runs once per cell rather than once per
frame. Measured effect: a scene build averages 0.07 ms. The cache is keyed by
`cellIdx·64 + depth·2 + xray` and dropped when the maze changes.

Mid-step the player straddles two cells, so `buildScene` draws the **union** of
what the source and destination cells can see. Without that the visible set
would snap at the instant of arrival and geometry would pop in a step behind
the movement.

`cfg.xray` restores pure passage-reachability as a deliberate, labelled option
— it is genuinely useful for studying how the 4-D lattice fits together, just
not for being lost in it.

**`collectWallEdges`** — extract the wireframe. A cell is a unit tesseract; a
wall between two cells is a unit 3-cube. Drawing all 12 edges of every wall cube
turns a flat run of wall into a dense ladder of rungs, so **coplanar walls are
merged**: within one wall hyperplane the wall cubes tile a 3-D grid, and every
lattice edge of that grid is bordered by four cube slots. Classify by how many
are solid:

| Solid slots | Meaning | Action |
|---|---|---|
| 4 | fully interior | skip |
| 2, adjacent | flat surface | skip — this is what kills the rungs |
| 2, diagonal | crease | draw |
| 1 or 3 | convex / concave edge | draw |

What survives is exactly the silhouettes of wall slabs plus the rims around
openings — so **doorways outline themselves for free**. Measured effect: 115 →
85 segments in a typical view, and a dramatic readability gain.

Coordinates are **doubled** so every corner lands on an integer, which makes
edge deduplication an exact integer-key `Map` lookup with no epsilon. The edge
record is a flat array of eleven numbers:

| Index | Meaning |
|---|---|
| 0–3 | first endpoint, doubled lattice coordinates |
| 4–7 | second endpoint |
| 8 | the world **axis the edge runs along** — 4 values |
| 9 | the **surface direction** of the wall face that owns it: `axis·2 + (wall on the +axis side ? 1 : 0)` — 8 values |
| 10 | goal flag |

Both tags are absolute world orientations, so either colouring gives a fixed
orientation cue that survives any amount of turning. An edge is shared by up to
four wall faces, which may face different ways; the first claim wins, and since
cells arrive in BFS order from the player, the first claim is the nearest
surface — the one you are actually looking at.

**`hideOccludedEdges(maze, list, eye)`** — hidden-line removal, and what makes
the walls read as *solid* rather than as a wireframe you can see through. Cell
visibility already drops whole cells with no sightline, but inside the set that
survives, walls still overlap: the rim of a doorway two cells away is
half-hidden behind the wall in front of it, and drawing it whole is exactly
what makes the picture look transparent. Testing visibility per **point**
instead of per cell turns every wall face into a real occluder, and a wall you
are facing then draws as nothing but its own outline.

Each edge is sampled at 13 points (`HLR_SAMPLES = 12` intervals); visible runs
are coalesced, and each run boundary is refined by 7 bisection steps so the cut
lands on the occluding silhouette rather than on a sample boundary. Sub-segments
inherit the parent's tags. The all-visible case — the common one — is a fast
path that pushes the original edge through untouched.

**Why the permissive graze rule exists.** The points HLR aims at lie *on*
lattice planes by construction (an edge is a lattice edge), so a ray from the
eye to one of them grazes corners routinely. Under the conservative rule a graze
would report "blocked" and punch a hole in a line the cell gate has already
agreed is visible — visibly worse than a hairline leak. So `permissive` treats a
simultaneous crossing as clear if **any ordering** of the tied crossings stays
in open space. It has to be any ordering, not the first open one: a greedy walk
can take a branch that dead-ends at its *second* crossing while the other order
walks straight through. The branch search shares one 4096-step budget so a
pathological ray cannot explode.

There is no such thing as an occlusion test along ana; see "the fourth
dimension never occludes" above. Solid walls are opacity along **forward**,
which is the only direction that has any.

Cost: about 1.5 ms for the worst case the UI allows (7⁴, 100 % braid, sight
range 14), against 0.9 ms for the same scene as a wireframe. Unlike cell
visibility it cannot be cached per cell, because the eye moves continuously
during a step animation — but it is cheap enough not to need it.

`cfg.opaque` gates it; `cfg.xray` overrides it, since drawing through walls is
the entire point of x-ray and hidden-line removal would undo it.

**`projectEdges`** — transform to local coordinates (`basis` transpose, then
`Rᵀ(theta)` for any turn in progress), clip, then divide:

    x = right · F  / fwd
    y = up    · F  / fwd
    w = ana   · Fa / fwd        Fa = F · anaScale
    depth = fwd                 (kept for sorting and fading)

**Clipping happens in local 4-D space, before the divide.** Both clip planes are
*linear* there; perspective maps straight segments to straight segments but not
affinely, so clipping after the divide would put the cut in the wrong place.
Two planes:

- **near**: `fwd ≥ cfg.near`
- **stereo guard**: `Fa·ana + guard·fwd > 0`, where `guard = stereoDist · 0.9` —
  this keeps retina-depth `w` from reaching or passing the virtual eye, which
  would make the renderer's stereo divisor `1 + w/D` zero or negative and turn
  the geometry inside-out.

The guard **must** be written in terms of `Fa`, not `F`. Using the
un-exaggerated focal was a real bug: at `anaScale = 3` the exaggeration was
applied after clipping, points slipped past the eye, and the divisor went to
−1.7. The test suite now asserts the divisor stays above 0.05 across the whole
`anaScale` range.

`cfg.near` is small (0.06) on purpose. Larger values (0.6) discard the walls of
the player's own cell, which reads as a hole around the viewer rather than as a
cleaner view — verified with `diag-maze.js`.

### Layer 4 — `StereoRenderer`

Draws retina-space segments to the canvas. Retina depth `w` is turned into
horizontal parallax with an **off-axis** stereo projection — the eye vectors
stay parallel and the frusta are asymmetric:

    t  = D / (D + w)
    px = eyeX·(1 − t) + x·t
    py = y·t

`D = cfg.stereoDist`. At `w = 0`, `t = 1` and parallax is exactly zero: the
player's own slice is pinned to the screen plane. This form gives **zero
vertical parallax and no keystone distortion**, which toed-in cameras would not.

Mode dispatch in `render()`:

- **wall / cross** — two square panes side by side, centred, each
  `min(cfg.paneWidth, W/2)` wide. Cross-eyed simply swaps which pane gets which
  eye. A 1px divider and two grey fusion dots complete the layout.
  `renderer.lastPanes` is exposed for the test suite.
- **anaglyph** — one full-viewport image drawn twice with
  `globalCompositeOperation = 'lighter'`, pure red for the left eye and pure
  cyan for the right, luminance-only line colours so the channels stay clean.
- **mono** — one full-viewport image with `eyeX = 0`.

**Pane width and the fusing ceiling.** Wall-eyed viewing requires matching
points in the two images to be no further apart than the interocular distance
(~63 mm ≈ 240 CSS px), because the eye axes must stay parallel or diverge
slightly — they cannot splay. Cross-eyed has no such limit. Hence
`PANE_DEFAULT = { wall: 240, cross: 460 }`, re-targeted on mode change unless
`paneTouched` records that the user has moved the slider, after which their
choice is respected.

Segments are bucketed by (colour index, depth band) and stroked one bucket per
`beginPath`, so the per-frame cost is a couple of dozen draw calls rather than
one per segment. Alpha and line width both fall off with the depth band,
reinforcing forward depth as the occlusion cue.

**The renderer owns no palette of its own.** It renders a 3-D line soup whose
segments carry a `col` index into `cfg.palette`, plus a `goal` flag; the
highlight colour is the slot one *past* the end of the palette, so a palette of
any length works and the bucket keys can never collide with the goal. That is
what keeps layer 4 ignorant of mazes: the scene builder decides what `col`
means.

Two palettes are wired up in the app:

| Reading | Slots | Colours |
|---|---|---|
| **By direction** (default) | 8, the wall's surface orientation | X− orange `#ff9f45` / X+ red `#ff6b6b`; Y− teal `#35d0c0` / Y+ green `#6bff8f`; Z− violet `#9b8cff` / Z+ blue `#6bb2ff`; W− pink `#f778ba` / W+ yellow `#ffd24a` |
| **By axis** | 4, the axis the edge runs along | X red `#ff6b6b`, Y green `#6bff8f`, Z blue `#6bb2ff`, W yellow `#ffd24a` |

The two hues on an axis are deliberately *related* — a warm pair, a green pair,
a blue pair, a warm-bright pair — so the axis is still readable at a glance
while the sign is distinguishable on inspection. Eight unrelated hues would make
the axis unreadable, which is the cue that matters most while navigating.

### Layer 5 — `drawMap`

The 4-D map as a **grid of (Z,W) panels, each an (X,Y) slice**. X/Y walls are
drawn on cell borders within a panel; Z and W passages lead out of a panel and
so are drawn as coloured dots pointing toward the neighbouring panel. The
player's cell and the goal are marked. Modes: `visited` (fog of war), `full`,
`off`.

## Application wiring

- `cfg` — the single render-config object threaded through layers 3 and 4:
  `mode, focal, near, halfExtent, sightDepth, sep, stereoDist, anaScale,
  animSpeed, guides, paneWidth, xray, opaque, wallColor, palette`.
  `fovToFocal(deg)` converts the FOV slider.
- The **Sight** dropdown is one control over two flags: `solid` →
  `{xray: false, opaque: true}`, `los` → `{xray: false, opaque: false}`,
  `xray` → `{xray: true}`. Keeping them as one user-facing choice avoids the
  meaningless fourth combination (x-ray *with* hidden-line removal).
- The **Wall colour** dropdown sets `wallColor` and swaps `cfg.palette`; a
  swatch key is generated under it from the palette itself, and the on-screen
  **compass doubles as the legend** — `axSpan` paints "fwd → +X" in the same
  colour the walls facing +X are drawn in, so it stays correct automatically
  when the palette changes.
- `MOVE_KEYS` — w/s forward/back, a/d left/right, r/f up/down, t/g ana/kata.
- `TURN_KEYS` — arrows yaw/pitch, e/q turn ana/kata, z/x roll.
- Sidebar pads carry `data-move` / `data-turn` and call the same Navigator
  methods, so pointer and keyboard input share one path.
- **Sidebar sections must not imply scope they don't have.** The map's reveal
  dropdown originally sat in the *View* section labelled just "Map", which read
  as if "visited only" constrained the 3-D view — it never did; it is fog of war
  for the corner overlay alone. It now lives in its own **4D map overlay**
  section with a hint saying so explicitly, and the thing that *does* constrain
  the 3-D view is the **Sight** control next to it. A browser test asserts the
  two are independent.
- **Keyboard focus rule.** The keydown handler must *not* blanket-ignore
  `INPUT`/`SELECT` — doing so silently killed navigation the moment you touched
  any slider or dropdown. It ignores only genuinely typable fields
  (`TEXTAREA`, and `INPUT` of type text/number/search/email/password), lets a
  focused range/select keep the arrow keys so it stays operable, and blurs
  sliders and selects on `change`.

## Known quirks (not bugs)

- **Identical segment counts across sizes.** A 4⁴ and a 7⁴ maze both report 27
  segments from the start cell. This is a genuine coincidence of local corridor
  topology near the origin, not shared state: recursive backtracking produces
  long snake-like corridors, and from a corner cell the sightline is a short
  straight run whose *shape* is often the same even when the mazes differ.
  `maze.count` differs as expected and `test-core.js` proves the mazes
  themselves differ. Line of sight makes this more common than it used to be,
  since it clips the view down to just that run.
- **The solid-wall view is sparse.** A sealed cell alone is 32 edges, so a view
  of 27 segments is "part of your own cell plus one doorway" — which is what
  being in a corridor actually looks like. Switch Sight to x-ray to see the
  neighbourhood.
- **Solid walls are still drawn as outlines, not filled surfaces, and that is
  correct.** A wall face in 4-D is a 3-cube, and it projects to a 3-D *region*
  of the 3-D retina, not to a 2-D patch. Filling it would mean filling a volume
  of the stereo image — the eye integrates along its own sightline through that
  volume, so a filled wall would read as opaque fog that hides everything at
  every ana level, including things that are merely beside it. The correct
  rendering of an opaque 4-D wall on a stereo display is its visible outline:
  silhouette plus the rims where it stops. That is what hidden-line removal
  produces.
- **A straight hypercorridor shows what look like four spurious diagonals.**
  They are eight nearly-coincident corner edges that separate only under stereo.
  Correct, not missing geometry.
- **Downscaled screenshots look like a starburst.** A thumbnail of the wireframe
  is a moiré artifact of the downscale; `diag-maze.js` renders at full pane
  resolution precisely so the view can be judged rather than squinted at.

## Testing

    run-tests.bat           # both suites, with NODE_PATH wired to the npx cache
    node test-core.js       # 96 tests, no browser
    node test-render.js     # 65 tests, Playwright

`test-core.js` covers index round-trips, passage symmetry, boundary
containment, connectivity, perfect-maze edge count for 3⁴–7⁴, seed determinism,
braiding, 4000 random turns preserving an exact signed permutation with
determinant +1, four ana-turns returning to identity, a 20000-step random walk
that never crosses a wall, blocked-move bumps, visibility never leaking through
walls, line of sight being a strict subset of reachability with an L-corridor
whose far arm is reachable but unseeable and a straight corridor visible end to
end along all four axes, a cell glimpsed diagonally through a doorway,
`losClear`'s conservative corner rule and its permissive counterpart (including
a graze that only the *second* ordering gets through, and one that no ordering
does), hidden-line removal (a sealed cell keeping all 32 edges — the test that
catches an over-strict graze rule, since every one of those rays ends on a
lattice plane — a straight corridor drawn whole, real occlusion in a real maze,
never emitting more than it was given, every surviving piece lying inside an
original edge, and x-ray ignoring HLR entirely), both colour readings staying
inside their palettes, a sealed cell producing exactly 32 edges
(a tesseract), edge dedup and
axis tagging, perspective-divide correctness, near-plane culling and clipping,
finite coordinates over 300 poses × 3 ana scales, a strictly positive stereo
divisor, painter-sort order, and the rotation-convergence identity
`R(θ)ᵀ·Mᵀ·d == (M·R)ᵀ·d`.

`test-render.js` covers page load errors, maze construction, all four stereo
modes, pane geometry and the fusing limit, the cross-eyed pane swap (with the
width pinned, since mode switching re-targets it), anaglyph channel balance and
unpaired-pixel parallax, zero-parallax straddling, the ordering
`solid < wireframe < x-ray` in drawn geometry with solid and wireframe seeing
*the same cells* (so the whole difference is per-point occlusion), a solid-wall
scene build staying under 12 ms, the two wall-colour palettes reaching both the
segment records and the actual pixels, **the map's reveal setting leaving the
3-D view byte-identical**, the visibility cache holding a scene build under 4 ms,
keyboard navigation, wall blocking, an end-to-end shortest-path solve through
the real `Navigator` that triggers the win state, and a settings sweep over
sizes/fov/depth/sdepth/sep/braid asserting finite geometry and a positive stereo
divisor throughout.

Playwright is not a project dependency; the browser suite is run with
`NODE_PATH` pointed at an npx cache.

**Two lessons the test suite learned the hard way**, both worth preserving:

- *Do not compare projected geometry by pairing sorted arrays.* Two poses that
  produce the identical segment *set* can differ by 1 ULP in the sort key,
  which reshuffles the arrays and produces a spurious "drift" of 33. Test the
  underlying algebraic identity instead.
- *The player stands at a cell centre*, so no wall corner is ever exactly on the
  zero-parallax plane — the plane runs *between* wall planes. The correct
  assertion is that geometry straddles it (ana and kata both present, W-running
  edges satisfying `w1·w2 < 0`), not that anything sits on it.
