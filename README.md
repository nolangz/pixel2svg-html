# Pixel2SVG-HTML

Pixel2SVG-HTML is a Codex skill for converting raster logos into clean, smooth, editable SVG and standalone JavaScript-rendered HTML.

The skill is built for logo reconstruction work where vector craft matters more than blind pixel tracing. It favors the lowest-complexity geometry that passes visual QA, uses overlay evidence for each fitting iteration, and treats smooth edges as a hard quality gate. IoU is measured and optimized, but it is not used as a fixed global pass/fail threshold.

Use this repository when the request is static logo vectorization. If the request includes logo animation, motion studies, replay controls, timeline tuning, or animated showcase HTML, use the companion `pixel2motion` skill instead.

## What It Produces

- `logo.svg`: semantic, editable SVG with stable ids per visual part
- `logo.html`: dependency-free HTML that rebuilds the SVG through JavaScript DOM calls
- `outputs/fit_iterations/*.png`: overlay snapshots for fitting iterations
- `outputs/final_render.png` and `outputs/html_render.png`: browser-rendered checks
- `outputs/overlay_progress_strip.png`: source-to-final QA strip
- `outputs/fit_work/`: resumable fitting state and iteration data

## Repository Contents

- `SKILL.md`: Codex-facing workflow and acceptance criteria
- `agents/openai.yaml`: UI metadata for the skill
- `references/`: fitting playbooks and specialized ribbon/wordmark guidance
- `scripts/`: deterministic helpers for tracing, rendering, overlays, path audits, ribbon fitting, and HTML generation

## Requirements

- Python 3.10+
- `Pillow` and `numpy` for image analysis helpers
- Chrome or Chromium for headless SVG/HTML rendering

Recommended local setup:

```bash
python3 -m venv .venv
.venv/bin/pip install pillow numpy
```

If Chrome is not on the default path, set `CHROME_BIN` before running render checks:

```bash
export CHROME_BIN="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
```

## Basic Workflow

1. Read `SKILL.md` and the relevant reference file before fitting.
2. Measure the source image and choose the simplest plausible geometry.
3. Render each candidate and save an overlay:

```bash
python3 scripts/render_overlay.py logo.svg source.png \
  --out outputs/fit_iterations/01_overlay.png \
  --render-out outputs/final_render.png \
  --report outputs/fit_metrics.json
```

4. Audit complex curves when smoothness is a concern:

```bash
python3 scripts/svg_path_audit.py logo.svg \
  --out-svg outputs/bezier_segments.svg \
  --report outputs/bezier_audit.json
```

5. Generate the standalone HTML deliverable:

```bash
python3 scripts/svg_to_js_html.py logo.svg --out logo.html --title "Vectorized Logo"
```

6. Build the progress strip:

```bash
python3 scripts/overlay_progress_strip.py \
  --source source.png \
  --dir outputs/fit_iterations \
  --pattern "*overlay*.png" \
  --final-image outputs/final_render.png \
  --out outputs/overlay_progress_strip.png
```

## GitHub Upload Checklist

- Confirm `SKILL.md`, `agents/openai.yaml`, `references/`, and `scripts/` are committed.
- Keep generated deliverables, local virtual environments, caches, and per-logo `outputs/` out of git.
- Add a `LICENSE` file before publishing if this repository should grant reuse rights.
- After creating the GitHub repository, add the remote and push:

```bash
git remote add origin git@github.com:<owner>/pixel2svg-html.git
git branch -M main
git push -u origin main
```

