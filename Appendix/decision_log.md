# Decision Log

## Perception Q1 — The Road Disappears

- **Baseline: brightness-based boundary detection**
  - At the required rows `v = 170` and `v = 260`, the image was split into left and right halves.
  - The brightest pixel in each half was taken as the corresponding lane boundary, subject to a brightness threshold.
  - The lane centre was then calculated as the midpoint of the two detected boundaries.
  - **Why:** This was deliberately kept simple so that the baseline could clearly show where a direct brightness-based approach fails, especially when a shadow or bright road seam becomes stronger evidence than the actual lane marking.

- **Revision: use information across the image height**
  - Instead of detecting the boundary only at the two required rows, valid bright boundary points were collected across rows and separate linear models were fitted for the left and right boundaries.
  - At the required rows, an actual detected boundary was preferred; the fitted line was used only when the boundary was missing.
  - **Why:** A missing lane marking at one row should not make the whole estimate unknown when the same boundary is visible elsewhere in the frame. This also provides a way to continue a boundary through the shadow/missing-line region.

- **Use actual evidence before regression**
  - Regression was treated as a fallback rather than replacing valid image observations.
  - **Why:** This avoids unnecessarily imposing a geometric estimate when the current frame already contains usable evidence.

- **Confidence based on evidence source and regression fit**
  - Full confidence was assigned when both boundaries came directly from image evidence. Lower confidence was used when one or both boundaries had to be estimated from regression, with the regression MSE affecting the score.
  - **Why:** A coordinate obtained from extrapolation should not be treated as equally trustworthy as one directly supported by visible lane evidence.

---

## Perception Q2 — A Crossing with Conflicting Evidence

- **Baseline: rule-based decision system**
  - The supplied `red_score` and `person_score` were combined with the V2X light state to produce one of `GO`, `SLOW`, or `STOP`.
  - The decision hierarchy was conservative: `STOP > SLOW > GO`, with the person evidence handled separately because a visible person does not automatically mean the vehicle must stop.
  - **Why:** A transparent rule-based system makes it possible to inspect exactly why a decision was made and is easier to test under conflicting evidence than an opaque learned model.

- **Use SLOW as an intermediate state**
  - Moderate or unresolved evidence leads to `SLOW` rather than immediately choosing either `GO` or `STOP`.
  - **Why:** The vehicle should reduce its risk while evidence is still uncertain instead of treating uncertain perception as permission to continue normally.

- **Treat V2X as time-dependent evidence**
  - Message age was calculated from `time_s - v2x_sample_time_s`, and the revised logic checks whether the V2X message is stale before treating its state as current.
  - **Why:** A V2X message describes an earlier observation. A stale `GREEN` message should not override newer visual evidence of a red light.

- **Add image-based person detection in the revision**
  - The revised system uses image evidence to identify the person's position rather than relying only on the supplied person score.
  - **Why:** A person being visible somewhere in the image is different from a person actually occupying the vehicle's path. Position information is therefore more useful for deciding whether `STOP` is necessary.

- **Keep the decision logic interpretable**
  - The revision remained a rule-based system rather than replacing the whole pipeline with a learned classifier.
  - **Why:** The important part of this question was handling contradictory and delayed evidence safely, so an explicit decision hierarchy made the behaviour easier to inspect and debug.

---

## Motion Planning Q1 — Planning for a Vehicle that Carries People

- **Start with normal 2D A\***
  - A conventional 8-connected grid A* planner was implemented first, with user-defined grid size, start, goal and obstacles.
  - **Why:** This provides a simple baseline and makes the limitation of position-only planning visible before introducing vehicle-specific constraints.

- **Model the road as a constrained drivable region**
  - A road-like environment was created using a horizontal lane connected to a vertical lane through a 90° bend, with four static obstacles.
  - Cells outside the road were treated as blocked.
  - **Why:** This creates a simple but realistic enough environment in which A* has to navigate both road geometry and obstacles.

- **Assume 1 grid cell = 1 m for turning-radius analysis**
  - **Why:** The grid itself has no physical scale, so a scale assumption was required to compare the discrete A* path with the buggy's physical minimum turning radius.

- **Validate the A* path after planning**
  - The minimum turning radius was calculated from the buggy's wheelbase and maximum steering angle:
    `R_min = L / tan(δ_max) ≈ 3.68 m`.
  - Consecutive path segments were then checked for heading changes and the corresponding approximate required turning radius.
  - **Why:** A path being collision-free on the grid does not mean that the passenger-carrying buggy can physically execute it.

- **Use Hybrid A* as the vehicle-aware extension**
  - The optional Hybrid A* implementation uses `(x, y, θ)` rather than only `(x, y)`.
  - The buggy motion is generated using the kinematic bicycle model with `L = 2.30 m` and steering limited to `±32°`.
  - **Why:** The buggy has a heading and cannot turn arbitrarily. Including heading and steering constraints allows the planner to search for trajectories that are kinematically feasible.

- **Discretise heading and steering**
  - Heading was represented using 72 bins, and five steering actions were used: maximum left, half-left, straight, half-right and maximum right.
  - A motion step of `0.5` grid cells was used.
  - **Why:** These choices keep the Hybrid A* implementation computationally manageable while still giving it enough steering and heading resolution to produce smoother trajectories than grid A*.

- **Penalise steering and steering changes in Hybrid A\***
  - The Hybrid A* cost includes distance, steering effort and change in steering.
  - **Why:** Minimising distance alone could produce unnecessarily aggressive steering. Penalising steering magnitude and sudden steering changes encourages trajectories that are more practical for the buggy and its passengers.

- **Use a multi-term cost for competing paths**
  - Candidate paths were evaluated using path length, steering effort/curvature, obstacle clearance and steering-change cost:
    `J = wL JL + wδ Jδ + wC JC + wΔδ JΔδ`.
  - **Why:** The shortest collision-free path is not necessarily the best path for a passenger-carrying vehicle. A longer path may be preferable if it is smoother and maintains more clearance from obstacles.

- **Use speed-dependent cost weights**
  - At 5 km/h, greater relative weight was given to path length.
  - At 20 km/h, greater weight was given to curvature, obstacle clearance and steering changes.
  - **Why:** As speed increases, aggressive steering and small obstacle clearances leave less time and margin for correction, so safety and smoothness should become more important than simply minimising distance.

- **Keep the Hybrid A* collision model simplified**
  - Collision checking was performed using the vehicle's position on the grid rather than a full geometric footprint.
  - **Why:** The extension was intended to demonstrate heading and steering feasibility without turning the assignment into a full vehicle-footprint collision-planning implementation. This is a known limitation of the prototype.
