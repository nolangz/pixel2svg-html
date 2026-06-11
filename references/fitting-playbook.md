# Fitting Playbook — Field-Tested Geometry Strategies

Hard-won lessons from real vectorization runs. Read this BEFORE writing fit code; every item below was learned by burning an iteration on it.

## 1. Measure before modeling

- Extract per-part data first: foreground masks by color, part centers/radii (`sqrt(area/π)` for dots), mean colors, background type.
- Curve marks: sample edge runs per **column** where the curve is shallow and per **row** where it is steep (>45°). The scan-direction extent overstates true width on slopes — perpendicular width = run length × cos(θ), θ from the local centerline slope.
- Edge positions from scans are accurate to ≤1px *perpendicular to the edge* regardless of slope — scan quantization shifts samples *along* the edge, which is harmless for curve fitting.
- Exclude occluded regions (e.g. where a colored dot covers the stroke end) from all measurements.

## 2. Constrain every feature search

`argmin`/`argmax` over a whole curve finds the wrong feature (a global extremum elsewhere on the curve) more often than the right one. Always window feature detection: "the hump apex is the min-y *within x∈(150,500)*". When a bend location is needed, use a tangent-angle criterion (the point where the local tangent crosses ~45°) rather than discrete curvature, which is noisy at sample scale.

## 3. Variable width ⇒ filled outline; fit boundaries from SOURCE pixels

- A stroked centerline is only correct for truly constant-width marks. Calligraphic/tapered ribbons need a filled outline path (top boundary forward + bottom boundary reversed + caps).
- **Do NOT build the outline by offsetting a fitted centerline.** A G1 centerline has curvature jumps at its knots; the offset curve inherits them as genuine tangent breaks, and you end up fitting your own construction artifacts (a 10–15° phantom kink that no amount of refitting removes). Fit each boundary **directly against source-measured edge points** — real artwork edges are smooth.
- Tips under covering shapes are analytic: tip = part center ± normal × half-width; use butt caps hidden under the covering dot; never let round caps protrude.
- **Exception — closed and/or self-intersecting ribbons** (∞ marks, scripts): two open boundaries don't exist, so per-boundary fitting is impractical. Use §8: a dense C1 centerline as *scaffolding* (never analytically offset from sparse knots) with measured recentering and a final source-pixel edge snap.

## 4. Segment fitting recipe (per boundary)

1. Knots at real shape features only: extrema (windowed search), genuine bends (tangent-angle criterion).
2. **Free LSQ per segment**: endpoints fixed, P1/P2 fully free (closed-form normal equations), chordal parameterization, then 2–3 rounds of t-refinement (reproject each sample to the nearest point on the current curve). Free fits routinely reach 0.3–1.8px RMS on clean data.
3. **Then check G1**: measure the tangent-direction gap across each shared knot.
   - Gap ≤ ~4°: repair at zero cost — set both handles to the bisector direction and refit *handle lengths only* (linear LSQ over two scalars).
   - Gap > ~8°: a genuine local curvature spike at that knot. Do **not** force a single-knot G1 (it taxes both neighbors by 2–5px RMS, and angle/position searches just relocate the pain). Instead **bracket the spike with a short bridge segment**: two knots ~100–130px of arc on either side, so the tight curvature lives inside its own segment with relaxed G1 joins at both ends. This converts a 12° kink + 3px RMS into <1px RMS with G1 everywhere.
4. Re-audit after any repair: compute turn angles at every join directly from the control points (in-tangent vs out-tangent). Classify intentional corners (tip caps, designed sharp points) separately from kinks — only undesigned >8° joins are failures.

## 5. One feedback-correction round

After the first decent fit, render and measure render-vs-source edge deltas per scan station, convert to perpendicular units (× cos θ), and apply as corrections to the boundary offsets. **Model corrections smoothly** — a low-order polynomial over the arc, never raw wiggly interpolation: chasing per-station deltas (±3px noise) injects fake features that later read as kinks. One smooth correction round typically recovers 2–4px of systematic width/center bias; further rounds chase noise.

## 6. Iteration discipline

- One change per iteration, overlay after each, artifacts numbered (`NN_name_overlay.png`).
- Default budget: 10 iterations. Stop when either an accepted fit is reached (smoothness, structural, and visual checks pass, with IoU pushed as high as practical) or 10 iterations have been attempted (see SKILL.md).
- Metrics (IoU, src_only/render_only, boundary RMS) decide *whether* an iteration helped; eyes still decide *whether the shape is right*. There is no fixed global IoU threshold. A slightly lower-IoU smooth fit can pass when residuals are explainable; an IoU gain with a new tangent kink or stair-step edge is a regression.
- Persist every fit script, sampled data (`.npy`), and parameter set under `outputs/fit_work/` — resumability is a deliverable. Never leave fit state in `/tmp`.

## 7. Environment pragmatics

- System Python is often externally managed (PEP 668): create a venv for Pillow/numpy (`python3 -m venv .venv && .venv/bin/pip install pillow numpy`).
- No SVG rasterizer needed: headless Chrome renders SVG exactly (`scripts/render_overlay.py` wraps render + overlay + IoU in one command). The same Chrome screenshots the JS HTML deliverable.
- Patch fit scripts by rewriting the file, not by fragile string-replace one-liners — a silently failed in-place patch reruns the old code and wastes an iteration.

## 8. Closed / self-intersecting ribbons: centerline scaffold + recenter + snap

For ∞ marks and script signatures the outline is one closed self-intersecting loop — there are no two open boundaries to fit per §3. Field-tested recipe (`scripts/fit_ribbon_centerline.py`, details in `references/ribbon-fitting.md`):

1. Seed a closed Catmull-Rom centerline through 15–25 hand-read keypoints (extrema, caps, crossing once per pass, taper); ±5px accuracy suffices. Put the start/seam where an occluder will hide both outline caps.
2. Sample densely; measure along each normal: the mask interval containing the sample gives a midpoint (recenters) and a width. Exclude occluders and every self-crossing with exclusion circles; bridge gaps by interpolation along arclength **circularly** (gaps may wrap the seam); smooth lightly (midpoints σ≈2.5, widths σ≈3 — heavier smoothing curve-cuts high-curvature caps ~1px).
3. Rebuild controls from smoothed midpoints **spaced by arclength** (~55px) and repeat; converges in 2–3 passes (mean shift 1.6 → 0.3px). Stride-based rebuilds inflate the control count every pass (22 → 56 in one run).
4. Sweep edges = center ± width/2 · normal, then **snap each edge sample to the source boundary** (bilinear threshold crossing along the normal, clamp ±2.5px, skip where width < 6px — hairline edges snap onto each other — and inside exclusions; smooth shifts σ≈2). The snap removes the ~1px systematic smoothing bias and is what reconciles this recipe with §3's boundaries-from-source rule.
5. Fit one cubic per ~30-sample chunk per edge (fixed endpoints, data tangents, LSQ handle lengths); emit left-forward + cut + right-reversed + cut + `Z` as one closed path with `fill-rule: nonzero` (a self-crossing stroke outline traversed this way winds consistently — the overlap unions, no holes).
6. Report each exclusion's arc fractions — a crossing's two passes are the split-cut parameters a downstream draw-on animation needs.

Gradient-filled parts (beads, glossy dots): binary-mask diffs over them are threshold artifacts, not shape errors — judge structurally and say so in the report.
