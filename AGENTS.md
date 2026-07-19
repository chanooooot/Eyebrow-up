# AGENTS.md — Build instructions for Browball

Instructions for any coding agent (Codex, etc.) working in this repo. **SPEC.md is the source of truth.** Read it fully before writing any code. This file tells you HOW to work.

## Working rules (mandatory)

1. **Think before coding.** State assumptions explicitly. If SPEC.md is ambiguous, ask the owner (Ham) — do not pick an interpretation silently.
2. **Simplicity first.** Minimum code that satisfies SPEC.md. No features beyond it. No abstractions for single-use code. No config systems or speculative flexibility. If `index.html` exceeds ~500 lines, you have probably over-engineered — justify or cut.
3. **Surgical changes.** When iterating, touch only what the change requires. Match existing style. Do not refactor working code unasked.
4. **Goal-driven.** A step is done when its acceptance check (SPEC.md §9) passes on a real device — not when the code "looks right."

## Deliverable

- Exactly **one file: `index.html`** at repo root. Inline `<style>` and `<script>`. No build step. No package.json. No framework.
- Only external dependency: MediaPipe Tasks Vision from CDN (`@mediapipe/tasks-vision`, FaceLandmarker, blendshapes enabled, `runningMode: "VIDEO"`, GPU delegate with CPU fallback).
- All tunable constants from SPEC.md §8 in one clearly-marked block at the top of the script.

## Implementation notes (established during design)

- `getUserMedia` requires HTTPS + a user gesture on iOS Safari — request the camera only after the Start button tap.
- Mirror the video (`transform: scaleX(-1)` or draw mirrored) — unmirrored selfie view feels wrong.
- Blendshape scores are per-frame floats 0–1. Signal = max of `browOuterUpLeft`, `browOuterUpRight`, `browInnerUp`.
- Raise-event detection needs **hysteresis + cooldown** (SPEC §3), not raw threshold crossing — raw crossing double-fires on noisy frames.
- Run detection in the `requestAnimationFrame` loop via `detectForVideo(video, timestamp)`; skip detection when the video timestamp hasn't advanced.
- In-app browser detection: user-agent sniff for `Line/`, `FBAN|FBAV`, `Instagram`. Show the redirect overlay before any camera code runs. Shared links must append `?openExternalBrowser=1` — LINE's in-app browser auto-opens such URLs externally, so the overlay is only a fallback.
- `<video>` needs `playsinline muted autoplay` attributes or iOS Safari forces it fullscreen.
- Preload the MediaPipe wasm + model during the start screen (fetch needs no user gesture — only the camera does); otherwise the <5 s on 4G budget (SPEC §9) fails.
- Auto-pause (SPEC §5): pause the game loop on `visibilitychange` (hidden) and when no face is detected for >~500 ms; resume on tap / face re-acquired. Backgrounded `requestAnimationFrame` stops, so without this, balls teleport past the hit-zone on resume.
- Share card: keep at most one snapshot frame in memory (captured at a hit moment), opt-in only, discard on restart, never transmit. `navigator.share({files})` needs `navigator.canShare` feature-detection; fallback = canvas → download link.

## Build order (verify each step on a real device before the next)

1. Static skeleton: start screen → camera permission → mirrored fullscreen video. Verify on iPhone Safari over HTTPS.
2. MediaPipe wired: live brow-meter bar showing the signal. Verify: moves with eyebrows, works with glasses, indoor light.
3. Calibration flow (relax → raise → midpoint). Verify: low-gap retry triggers when brows don't move.
4. Game loop: balls, hit-zone, raise events, score, lives, speed ramp. **Main tuning pass — do it live with Ham on his iPhone + one Android.**
5. Game over + share card (+ opt-in snapshot). Verify: shares into a LINE chat as an image.
6. In-app browser overlay. Verify: opening the URL from a LINE message shows the overlay.

Kill criterion (SPEC §10) is evaluated at step 4: if detection feels unfair after honest tuning, stop and report — do not add complexity to rescue it.

## Deployment

- GitHub repo, `index.html` at root, GitHub Pages from main branch. That is the entire pipeline.

## When to ask the owner

- Any deviation from SPEC.md
- Visual style decisions beyond "neon/arcade, emoji balls" that materially change the feel
- Before adding ANY dependency beyond MediaPipe
- Working title "Browball" — confirm or rename before deploy
