# Ball Game — Build Spec (one-shot)

## Goal
Mobile-first static web app called **Ball Game** that computes the optimal ball-throwing order for a pool game. The user draws the pool shape, places player markers, picks a start player, and the app renders the optimal throw order directly on the drawing.

## Game context
Players stand around the edge of a pool and pass a ball through everyone in a loop (players jump in, climb out, and the cycle continues). Rules that shape the optimizer:
- A ball can NEVER be thrown to a player standing directly next to the thrower.
- Throws across the pool are preferred over laterals (laterals risk jumping players being in the way).
- Very long throws are risky (drops); very short throws give low recovery time — a medium "sweet spot" length is ideal.

## Tech constraints
- Single `index.html` with inline CSS + JS. Vanilla JS, zero dependencies, Canvas 2D API.
- Deployable as a static site (GitHub Pages / Netlify).
- `localStorage` persists the last pool + players.
- Mobile-only usage: full-viewport canvas, touch events (`touchstart/move/end` with `preventDefault`), `touch-action: none` on canvas, proper viewport meta, minimum 44px tap targets, toolbar at the BOTTOM of the screen for thumb reach.

## UI flow (bottom mode toolbar: Draw | Players | Start | Generate | Clear)
1. **DRAW mode**: freehand drag traces the pool outline; on release, auto-close the path and simplify with Ramer-Douglas-Peucker (epsilon ≈ 2% of the bounding diagonal). Drawing again replaces the shape. Render as a filled water-blue polygon with a darker edge.
2. **PLAYERS mode**: tap inside the polygon to add a numbered circular marker; drag an existing marker to move it; long-press (~500ms) deletes it. Reject taps outside the polygon (point-in-polygon ray cast).
3. **START mode**: tap a marker to set the start player (highlight gold / ring).
4. **GENERATE**: computes and draws the throw cycle as arrows with sequence numbers at midpoints; also shows the ordered list (P3 → P7 → P1 …) in a collapsible bar.
5. **CLEAR**: confirm dialog, wipes state.

## Algorithm (min-cost Hamiltonian CYCLE starting at the start player — the game loops)
- Let `D` = max pairwise player distance (pool scale). `C` = polygon centroid.
- **Adjacency (HARD constraint)**: sort players by perimeter position — project each onto the nearest point of the polygon edge, order by cumulative perimeter distance (fallback: angle around centroid). Immediate circular neighbors are "adjacent"; throws between adjacent players are forbidden.
- **Edge cost** for throw i→j with distance `d` and segment `S`:
  - Lateral penalty (laterals hug the edge; across-throws pass near center):
    `cost_lat = w_lat * (dist(C, S) / (D/2))^2`, with `w_lat = 1.0`
  - Length penalty (U-shaped: long = drop risk, short = low recovery), ideal `d* = 0.6 * D`:
    `cost_len = w_len * ((d - d*) / D)^2`, with `w_len = 1.5`
- Total = sum of edge costs over the cycle. Solve exactly with **Held-Karp DP for n ≤ 14**; otherwise nearest-neighbor + 2-opt. Typical n is 4–12, so Held-Karp suffices.
- **Infeasibility fallback**: if no cycle satisfies the adjacency ban (e.g., n = 3), convert the ban into a large soft penalty (+10 per violation), pick the best cycle, and show a notice: "adjacent throw unavoidable".

## Rendering the result
- Arrows as slightly curved quadratic béziers (avoids all lines overlapping through the center), with arrowheads.
- Small numbered badge at each arrow midpoint showing throw order.
- Start player highlighted; the final arrow returning to start is dashed to show the loop.

## Edge cases
- Fewer than 3 players → disable Generate with a hint.
- Markers remain draggable after generation; regenerate on demand.
- Support concave pool shapes — no geometry may assume convexity.
- Handle canvas resize / orientation change: store all points in normalized 0–1 coordinates and scale to the canvas.

## Acceptance criteria
Works one-handed in a phone browser: draw a pool → place 6 players → pick a start → Generate shows a sensible order that prefers cross-pool, medium-length throws and never sends the ball to an adjacent neighbor.
