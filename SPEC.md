# SPEC — Browball (working title)

Eyebrow-controlled mini webgame. Balls fall from the top of the screen toward a hit-zone; the player raises their eyebrows at the right moment to hit them. Front camera + face landmark detection is the only input.

**Status:** Design locked 2026-07-19. Ready for build handover.
**Owner:** Ham. Personal project. Not LINE-related.

---

## 1. Goals

- Shareable mobile-first webgame — send a link to friends, they play in ~60 seconds
- Zero cost: no backend, no accounts, no data leaves the device
- Single-file deliverable, hosted on GitHub Pages

## Non-goals (v1)

- Leaderboard / any server component
- Audio (fully cut — no sound, no toggle)
- Combo scoring
- In-app browser (LINE/IG/FB) camera support — detect and redirect instead
- Multiple game modes

---

## 2. Core mechanic

- Rhythm/timing game. Balls spawn at top, fall at constant (ramping) speed toward a fixed **hit-zone** near the bottom.
- A **brow-raise event** while a ball is inside the hit window = HIT (ball pops, score +1).
- Ball passes through the zone with no raise = MISS (lose 1 life).
- A raise event with no ball in the window = no penalty (noise forgiveness).
- 3 lives. Game over at 0. Final score = total hits.
- Difficulty: fall speed and spawn rate ramp gradually over time.

## 3. Input & detection

- **MediaPipe Face Landmarker** (WASM, loaded from CDN) in blendshape mode.
- Signal: max of `browOuterUpLeft`, `browOuterUpRight`, `browInnerUp` per frame.
- **Raise event** = signal crosses calibrated threshold upward. Followed by a cooldown (~300 ms, tunable) before the next event can fire. Signal must drop below a lower re-arm threshold (hysteresis) before re-triggering.
- All detection runs client-side. Camera frames never leave the device.

## 4. Calibration (before each game)

1. "Relax your face" — sample signal ~1 sec → `baselineRelax`
2. "Raise your eyebrows!" — sample ~1 sec → `baselineRaise`
3. Threshold = midpoint of the two. If the gap between baselines is too small (face not detected well / no brow movement), show retry prompt.
- Doubles as the tutorial. Skippable on replay within the same session (reuse last calibration).

## 5. Screens / flow

1. **Start screen** — title, "Start" button (button tap satisfies the user-gesture requirement for camera permission), short one-line instruction.
2. **In-app browser overlay** — if LINE/FB/IG in-app browser detected (user-agent check), full-screen prompt: "Open in Safari/Chrome" with platform-specific instructions. Game does not attempt to run. LINE-specific escape hatch: URLs carrying `?openExternalBrowser=1` are auto-opened in the external browser by the LINE in-app browser — all shared links must include this param, so most LINE recipients never see the overlay (overlay stays as fallback for FB/IG and stripped params).
3. **Camera permission** → denied = friendly explainer screen with retry.
4. **Calibration** (Section 4).
5. **Gameplay** — mirrored camera feed as background, zoomed/cropped to the eyebrow/eye band (not full face); balls (emoji or plain circles) fall over it; hit-zone line near bottom; HUD: score, lives, small live brow-meter bar.
   - **Auto-pause:** game pauses when the page is backgrounded (`visibilitychange`) or when no face is detected for >~500 ms ("face not found" message). Resume on tap / face re-acquired. Prevents unfair misses from tab switches, notifications, or leaning out of frame — lives forgive frame-level noise only.
6. **Game over** — score, "Play again", "Share".

## 6. Share card

- Rendered client-side to a canvas: score + game branding, optional player snapshot.
- **Snapshot is opt-in**: a checkbox/toggle on the game-over screen, off by default. If on, use a frame captured at a hit moment (eyebrows up) during play; hold at most one such frame in memory, discard on new game.
- Delivered via `navigator.share` (files) with clipboard/download fallback on desktop.
- Any game URL included in shared text carries `?openExternalBrowser=1` (see §5.2).
- Nothing is uploaded anywhere, ever.

## 7. Tech stack

- **One `index.html`** — vanilla JS, Canvas 2D, CSS inline or one `<style>` block. No build step, no framework, no dependencies except MediaPipe Tasks Vision from CDN.
- Hosting: **GitHub Pages** (HTTPS required for getUserMedia — Pages provides it).
- Target: Safari iOS + Chrome Android (recent versions), desktop Chrome/Safari as free bonus.
- Portrait orientation primary; landscape can letterbox or just work naively.

## 8. Tunable constants (must be exposed at top of file)

| Constant | Initial value | Notes |
|---|---|---|
| `HIT_WINDOW_PX` | ~80 | Vertical zone height around hit line |
| `COOLDOWN_MS` | 300 | Min gap between raise events |
| `HYSTERESIS_RATIO` | 0.6 | Re-arm level as fraction of threshold |
| `START_FALL_SPEED` | tune on device | px/sec |
| `SPEED_RAMP` | tune on device | % per 10 sec |
| `START_SPAWN_INTERVAL_MS` | ~1500 | |
| `MIN_SPAWN_INTERVAL_MS` | ~600 | Ramp floor |
| `LIVES` | 3 | |
| `MIN_CALIBRATION_GAP` | 0.15 | Min raise−relax gap to accept calibration |

All initial values are guesses (opinion) — real values come from on-device testing.

## 9. Verification / acceptance

- [ ] Plays end-to-end on Ham's actual iPhone (Safari) and one Android (Chrome)
- [ ] Calibration succeeds in normal indoor lighting; retry path works
- [ ] Raise events feel responsive (<150 ms perceived lag from brow to pop)
- [ ] No false game-overs from detection dropouts (lives absorb hiccups)
- [ ] In-app browser overlay triggers when opened from LINE chat
- [ ] Share card generates and shares via LINE; snapshot only appears when opted in
- [ ] Works with glasses on
- [ ] Page loads playable in <5 sec on 4G

## 10. Kill criterion

After tuning constants on 2 real devices: if brow detection still feels unfair (frequent missed raises or ghost raises during normal play), **kill or pivot the input mechanic**. Do not sink time into detection heroics — the game only works if the control feels fair.

## 11. Backlog (v1.1+)

- Combo multiplier scoring
- Flappy-style mode (only if v1 detection proves very stable)
- Audio with off-by-default toggle
- Special ball types (bombs to avoid = deliberately DON'T raise)
- Leaderboard (would require backend — big scope jump, likely never)

---

## Decision log

| Decision | Choice | Rationale |
|---|---|---|
| Audience | Shareable link for friends, mobile-first | Ham's pick; drives all UX choices |
| Browser support | Safari iOS + Chrome Android; in-app browsers redirected | In-app browsers unreliably block getUserMedia (fact); fighting them not worth it for v1 |
| Mechanic | Rhythm (discrete events), not Flappy (continuous) | Discrete input tolerates detection noise; Flappy punishes exactly what face tracking is worst at; ~half the code |
| Left/right brow control | Rejected | Many players can't raise one brow independently; detection noisy |
| Calibration | Per-player relax→raise midpoint | Baselines vary by face/glasses/lighting (fact); 5 sec cost kills biggest UX risk |
| Round structure | 3 lives + speed ramp | Lives forgive detection hiccups; ramp creates replay loop; sudden-death too punishing with noisy input |
| Score | Plain hit count (v1) | Combo deferred to v1.1 — scope discipline |
| Sharing | Client-side image card via navigator.share | Score alone won't travel in chat; image will. Zero backend keeps zero cost |
| Face snapshot on card | Opt-in only, client-side only | Privacy; never auto, never uploaded |
| Stack | Single HTML, vanilla JS, Canvas 2D | ~300-line game; no build system = nothing to over-engineer; maximally portable |
| Hosting | GitHub Pages | Ham's pick; free, HTTPS, static-file-native |
| Camera on screen | Mirrored background, zoomed/cropped to eyebrow/eye band (not full face) | Ham's call 2026-07-19 — keeps focus on the mechanic, not a full selfie feed; share-card charm comes from the snapshot, not the live view |
| Audio | Cut entirely | Ham's call — not even a toggle in v1 |
