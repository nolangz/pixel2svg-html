<p align="center">
  <a href="https://www.pixel2motion.com">
    <img src="docs/community/pixel2motion-wordmark.png" width="520" alt="Pixel2Motion wordmark commercial service preview">
  </a>
  <br>
  <strong><a href="https://www.pixel2motion.com">www.pixel2motion.com</a></strong>
  <br>
  Better commercial Pixel2Motion services are coming online: polished logo-to-motion workflows, project-ready previews, and production support beyond the open-source skills.
  <br>
  中文：更完整的 Pixel2Motion 商业服务正在上线，面向更稳定的 logo-to-motion 交付、预览和项目支持。
</p>

---

# Pixel2SVG-HTML

**Raster logo → clean, smooth, minimal-complexity SVG → standalone JS-rendered HTML.**

[Commercial preview](https://www.pixel2motion.com) · [Skill instructions](https://github.com/nolangz/pixel2svg-html/blob/main/SKILL.md) · [Companion skill: Pixel2Motion](https://github.com/nolangz/pixel2motion)

Pixel2SVG-HTML is a Codex skill for converting raster logos (PNG/JPG/WebP/screenshots) into clean, smooth, editable SVG and standalone JavaScript-rendered HTML.

The skill is built for logo reconstruction work where vector craft matters more than blind pixel tracing. It favors the lowest-complexity geometry that passes visual QA, records overlay evidence for every fitting iteration, and treats edge smoothness as a hard quality gate. IoU is measured and optimized as high as practical, but it is never used as a fixed global pass/fail threshold.

This repository covers static logo vectorization. If the request includes logo animation, motion studies, replay controls, timeline tuning, or an animated showcase HTML, use the companion [Pixel2Motion](https://github.com/nolangz/pixel2motion) skill instead.

## Fitting Process

CueRecord fitting evidence, read left to right:

![CueRecord overlay progress strip](docs/process/cuerecord-overlay-progress-strip.png)

The teal overlays are QA checkpoints, not the deliverable. They show how the vector candidate is repeatedly compared against the raster source throughout fitting.

| Stage | What It Checks | Why It Matters |
| --- | --- | --- |
| Source raster | Mark shape, dot placement, wordmark baseline, spacing, and ink weight. | The reconstruction can only be credible if the static target is measured faithfully first. |
| Early overlay | Coarse vector geometry over the source image. | Misalignment is visible immediately: mark scale, wordmark width, dot position, and baseline drift. |
| Refinement overlays | Local geometry corrections while keeping the shape smooth. | The skill improves fit without falling back to noisy pixel-grid tracing. |
| Final vector | Clean semantic SVG: mark, dot, and wordmark as separate addressable parts. | This is the editable deliverable — and, downstream, the final-frame contract for Pixel2Motion. |

Pixel2SVG-HTML optimizes IoU as a diagnostic, but smoothness and structure are the hard gates. A high-IoU jagged trace is rejected when a lower-complexity smooth vector explains the logo better.

## Deliverables

- `logo.svg`: semantic, editable SVG with stable ids per visual part
- `logo.html`: dependency-free HTML that rebuilds the SVG through JavaScript DOM calls
- `outputs/fit_iterations/*.png`: overlay snapshots for fitting iterations
- `outputs/final_render.png` and `outputs/html_render.png`: browser-rendered checks
- `outputs/overlay_progress_strip.png`: source-to-final QA strip
- `outputs/fit_work/`: resumable fitting state and iteration data

## Workflow

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

## Repository Layout

- `SKILL.md`: Codex-facing workflow and acceptance criteria
- `agents/openai.yaml`: UI metadata for the skill
- `references/`: fitting playbooks and specialized ribbon/wordmark guidance
- `scripts/`: deterministic helpers for tracing, rendering, overlays, path audits, ribbon fitting, and HTML generation
- `docs/`: fitting-process evidence and README assets

## Publishing Checklist

- Confirm `SKILL.md`, `agents/openai.yaml`, `references/`, `scripts/`, and `docs/` are committed.
- Keep generated deliverables, local virtual environments, caches, and per-logo `outputs/` out of git.
- Add a `LICENSE` file before publishing if this repository should grant reuse rights.
- After creating the GitHub repository, add the remote and push:

```bash
git remote add origin git@github.com:<owner>/pixel2svg-html.git
git branch -M main
git push -u origin main
```

## More From Nolanlai

<p align="center">
  <a href="https://www.nolanlai.com">
    <img src="docs/community/nolanlai-creative-tools.png" width="520" alt="Nolanlai's Creative Tools">
  </a>
  <br>
  <a href="https://www.nolanlai.com"><strong>www.nolanlai.com</strong></a>
  <br>
  More creative tools, experiments, and notes from Nolanlai.
</p>

<p align="center">
  <img src="docs/community/vibe-creators-wechat-qr.jpg" width="360" alt="Nolanlai WeChat QR code for Vibe-Creators invite">
  <br>
  <strong>Vibe-Creators 交流</strong>
  <br>
  Vibe-Creators 群已满。扫码加我为好友，我邀请你进群。
  <br>
  The Vibe-Creators group is currently full. Scan the QR code to add me on WeChat, and I can invite you in.
</p>
