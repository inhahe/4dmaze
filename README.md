# 4D Maze

A maze in four spatial dimensions, rendered in stereoscopic 3-D. You walk it in
first person — forward/back, left/right, up/down, and **ana/kata** — and you can
turn in any of those planes too, including swinging your forward axis into the
fourth dimension.

**[Play it here](https://inhahe.com/4d-maze.html)** — or open `4d-maze.html`
in a browser. No build step, no dependencies, one file.

## The idea in one paragraph

In an ordinary 3-D first-person view, your retina is a 2-D plane: you project
*along* your forward axis, and your right/up axes become the two image axes.
Generalise that one dimension up and the retina becomes a **3-D volume**: you
still project along forward, and right/up/**ana** become the three image axes.
So the image you get is genuinely three-dimensional — and the natural way to
show a 3-D image on a flat screen is stereoscopically. **The fourth dimension
becomes stereo depth.** Left/right and up/down are on the screen; ana/kata is
*into and out of* the screen.

Your own 3-space slice lands exactly on the zero-parallax plane. Things one
step ana of you sit behind the screen; things one step kata pop out in front of
it. Occlusion and depth-fade still come from forward distance, exactly as in a
3-D game — the two depth cues are carried by two different mechanisms and never
get confused with each other.

## Stereo modes

| Mode | How to view |
|---|---|
| **Wall-eyed (parallel)** | Look *through* the screen into the distance until the two images drift together and a third appears between them. |
| **Cross-eyed** | Cross your eyes slightly until a third image forms in the middle, then let it sharpen. |
| **Red / cyan glasses** | Red lens over the **left** eye. Hue is spent on the channels, so the lines are monochrome in this mode. |
| **Mono** | No stereo at all — the fourth dimension collapses onto the picture plane. Useful for comparison, useless for navigation. |

Two small grey **fusion dots** sit at the top of each pane. Converging until
those two dots merge into one is the fastest way to lock the fusion; then look
back down at the maze.

Wall-eyed fusing has a hard physical ceiling: matching points in the two images
must be no further apart than your interocular distance (~63 mm ≈ 240 CSS px),
because your eye axes have to stay parallel or diverge slightly — they cannot
splay outward. Cross-eyed has no such limit. The **pane width** therefore
defaults to 240 px in wall-eyed and 460 px in cross-eyed, and re-targets
automatically when you switch modes — *unless* you have moved the slider
yourself, in which case your choice is respected from then on. On a screen too
narrow for the default, it re-targets to half the width instead, which is why
the pair fuses on a phone without touching anything.

Pane *height* is free: it costs nothing optically and simply widens the
vertical field, so the panes take whatever height the window gives them, up to
1.6× their width.

## Controls

### Keyboard

| Key | Action | | Key | Action |
|---|---|---|---|---|
| **W** / **S** | move forward / back | | **←** / **→** | yaw left / right |
| **A** / **D** | move left / right | | **↑** / **↓** | pitch up / down |
| **R** / **F** | move up / down | | **E** / **Q** | turn ana / kata |
| **T** / **G** | move ana / kata | | **Z** / **X** | roll |

Every turn is exactly 90°, animated. **Turn ana** swings your forward axis into
the fourth dimension: what was ana becomes straight ahead, and what was ahead
becomes kata. That is the move that has no 3-D analogue and the one worth
practising first — the Move section has clickable pads for all sixteen actions
if you'd rather not learn the keys yet.

Walking into a wall produces a short bump animation rather than silence, so a
missed input is distinguishable from a blocked one.

### On a phone

The layout goes single-column below 820px: the maze on top, the sixteen
movement and turn pads on a deck underneath it, and everything else behind the
**☰** button as a slide-out drawer. Turn the phone sideways and the deck moves
to a column on the right instead, because in landscape height is the scarce
axis and the view should keep it. The 4-D map scales itself to the view rather
than sitting there at a fixed size.

**All four display modes work on a phone, and it starts wall-eyed like the
desktop does.** Stereo is not a compromise at phone size — it is arguably
easier there. The pair automatically takes half the screen width, so on a
typical phone the two images end up only ~35mm apart, comfortably inside the
~63mm wall-eyed limit, and cross-eyed has no limit at all. Hold the phone at a
normal reading distance and look *through* it.

Do try mono once to see what it costs you: **in mono the fourth dimension is
invisible**, because ana and forward both collapse into "things get smaller",
which is exactly the ambiguity the stereo display exists to resolve. In mono
the 4-D map is your only way to tell where W went.

One thing that changes in stereo on a small screen: **the 4-D map stands
down**. An overlay drawn across a fused pair lands in one eye only, and the
resulting rivalry is far more distracting than a missing map, so the map takes
whatever margin the panes leave and hides itself when there isn't one. Narrow
the panes, or switch to mono or red/cyan, if you want it back.

### Sidebar

| Control | What it does |
|---|---|
| Stereo mode | Wall-eyed / cross-eyed / red-cyan / mono. |
| Size | 3⁴ = 81 up to 7⁴ = 2401 cells. |
| Seed | Integer. The same seed and size always give the same maze. |
| Loops (braiding) | 0% is a *perfect* maze — exactly one route between any two cells, no loops. Raising it knocks out dead-end walls, adding cycles and making the maze less tree-like and harder to solve by wall-following. |
| Field of view | 40°–130°. |
| Sight range | How many cells of passage-reachable geometry to draw. Higher is prettier and slower. |
| Pane width | Size of each stereo pane. See the fusing note above. |
| Stereo separation | Interocular distance for the projection. 0 makes both eyes identical (and the 4th dimension invisible). |
| Stereo depth (ana scale) | Exaggerates the ana axis. Above 1× the fourth dimension is stretched into more parallax than it strictly has, which makes it easier to read at the cost of realism. |
| Animation speed | Turn/step animation rate. |
| Sight | Solid walls / line-of-sight wireframe / x-ray. See below. |
| Wall colour | Eight colours by surface direction, or four by edge axis. See below. |
| Fusion guides | The two convergence dots. |
| Reveal (4D map overlay) | Fog of war for the **map** only — cells you have stood in / the whole maze / hide it. It does not change what the 3-D view shows. In a stereo mode the map keeps out of the panes, and hides itself if there is no room beside them. |

## Sight: solid walls, wireframe, x-ray

**Solid walls** (the default) makes every wall face a real occluder. You see
only what you have a clear straight 4-D line to, point by point, so a wall you
are facing draws as nothing but its own outline and the geometry behind it is
gone. Corridors that bend away are invisible until you reach the bend — which
is what makes it a maze.

**Line of sight, wireframe** does the cheaper, coarser thing: it drops whole
cells you have no sightline to, but draws every edge of the ones that remain,
including the far side of the wall in front of you.

**X-ray** draws every wall within sight range regardless, including corridors
that run round a corner behind a wall, straight through whatever is in the way.
It is a good way to study how the 4-D lattice fits together, and a bad way to be
lost in it.

The **Reveal** setting under *4D map overlay* is a separate thing entirely: it
is fog of war for the little map in the corner, and it has no effect on the 3-D
view.

### Why solid walls are still lines

Opaque does not mean *filled* here, and that is not a shortcut. A wall in 4-D is
a three-dimensional face, and it projects onto a **region** of the 3-D retina,
not onto a flat patch. Filling that in would mean filling a volume of the stereo
image, and your eyes look *through* that volume — so a filled wall would blot
out everything at every stereo depth, including things merely beside it rather
than behind it. What an opaque 4-D wall actually looks like is its visible
outline: the silhouette, plus the rims where it stops. That is what you get.

For the same reason, **the fourth dimension never hides anything**. It is drawn
as stereo depth, which makes it feel like a front-to-back axis, but in the 4-D
view ana is as sideways as left/right is. Occlusion runs along *forward*, and
only along forward.

## Reading the view

Walls are drawn as lines coloured by **which way the surface faces**. There are
eight possibilities, one per signed axis, with the two colours on an axis kept
related so you can read the axis at a glance and the sign on a second look:

| Axis | − direction | + direction |
|---|---|---|
| X (left/right) | orange | red |
| Y (up/down) | teal | green |
| Z (forward/back) | violet | blue |
| W (ana/kata) | pink | yellow |

Switching **Wall colour** to *By axis (4)* gives the simpler older reading —
red X, green Y, blue Z, yellow W — coloured by the axis each line runs along
rather than by the surface it bounds.

Either way the colours are **absolute world directions**, not relative to you,
so they stay put as you turn. The "your axes" compass in the top-left is
painted from the same palette and doubles as the legend; there is also a swatch
key in the sidebar.

The goal cell is outlined in bright warm white and drawn thicker. Everything
fades with forward distance, so nearer geometry reads as brighter and heavier.

A straight hypercorridor looks like the classic **cube within a cube**: the near
cross-section as an outer cube, the far one nested inside it. A side passage
running ana appears as a hole displaced *in stereo depth* — not left, right, up
or down. If you can see that displacement, the projection is doing its job.

## The 4-D map

The overlay in the corner is a 4-D map drawn as a **grid of panels**. Each panel
is one (Z, W) location and shows the (X, Y) slice at that location. X/Y walls are
drawn on the cell borders inside a panel; Z and W passages — which lead *out* of
a panel — are drawn as coloured dots pointing toward the neighbouring panel.
Your cell and the goal are marked. In "visited only" mode you see only what you
have walked through.

## The maze itself

Generated by a randomized depth-first search (recursive backtracker) over the
4-D lattice, which yields a *perfect* maze: fully connected, with exactly N−1
open faces for N cells and no loops at all. Braiding then removes a fraction of
dead-end walls to add cycles.

You always start at the origin cell. The goal is the cell **farthest** from the
start by passage distance, which is the classic maze convention — for a 4⁴ maze
that's typically 200+ steps, so expect a real walk rather than a quick one.

Recursive backtracking in 4-D produces long, snaking corridors rather than the
open braided rooms you may expect from 2-D mazes; the local view is often a
corridor with a single choice at each end.

## Architecture

The code is one file in five clearly separated layers, top to bottom. The maze
*data* knows nothing about navigation or rendering; navigation knows nothing
about rendering; the renderer knows nothing about mazes.

| Layer | What it is |
|---|---|
| 1. `Maze4D` | Pure data: one byte per cell, bit *a* = "open toward +a". Plus generation, BFS distances, neighbour queries. |
| 2. `Navigator` | Where you are and which way you face. Turns are exact integer operations on a signed permutation matrix, so orientation never drifts. |
| 3. Scene builder | Line-of-sight visibility (a 4-D grid march), wall-edge extraction with coplanar merging, per-point hidden-line removal, and the 4-D → 3-D projection with near-plane and stereo-guard clipping. |
| 4. `StereoRenderer` | Off-axis stereo projection of the 3-D retina image onto the canvas, in all four modes. |
| 5. Map | The (Z,W)-grid-of-(X,Y)-slices overlay. |

`design.md` has the full detail, including why the projection is set up the way
it is and where the non-obvious correctness constraints live.

## Tests

    run-tests.bat           # both suites
    node test-core.js       # 96 tests, layers 1-3, no browser
    node test-render.js     # 98 tests, Playwright, the whole app in a real browser

`test-core.js` slices layers 1–3 straight out of the HTML and evaluates them, so
there is no duplicated copy of the logic to drift out of sync. It covers maze
invariants (connectivity, perfect-maze edge count, no boundary leaks, seed
determinism), navigation (4000 random turns leaving the basis an exact signed
permutation, a 20000-step random walk that never passes through a wall),
visibility (an L-shaped corridor whose far arm is reachable but not seeable, a
straight corridor visible end to end along all four axes, a cell glimpsed
diagonally through a doorway), hidden-line removal (a sealed cell keeping all
32 of its edges, a straight corridor drawn whole, real occlusion in a real maze,
and every surviving piece lying inside an original edge), and projection
(near-plane clipping, finite coordinates over 300 poses, a stereo divisor that
never goes negative).

`test-render.js` drives the real page: all four stereo modes, pane geometry and
the fusing limit, anaglyph channel balance, that ana really does become
parallax, that solid walls draw less than the wireframe which draws less than
x-ray, that the eight-colour palette reaches the actual pixels, that the map's
reveal setting does *not* change the 3-D view, keyboard navigation, wall
blocking, an end-to-end shortest-path solve that triggers the win state, and
the mobile layout in a real phone context with touch — deck placement in both
orientations, the drawer, tapping the pads, touch-target sizes, that a fusable
stereo pair really is laid out at phone size, that the map never overlaps a
stereo pane, and that every keyboard action has a pad button, since a phone has
no keyboard.

`diag-view.js` and `diag-maze.js` are visual diagnostics — they render scenes
whose correct appearance is known in advance (a straight hypercorridor, a
corridor with an ana side-passage) so the projection can be *judged* rather than
guessed at.

## License

Public domain. Do whatever you want with it.
