# X11 partial presentation experiment note

Tracking note for the CPU-only typing-latency work in
[shawn111/herdr-app#1](https://github.com/shawn111/herdr-app/issues/1),
[tracking issue #3](https://github.com/shawn111/herdr-app/issues/3), and
[PR #2](https://github.com/shawn111/herdr-app/pull/2).

## Goal

Reduce typing latency on RHEL 8 / X11 machines without a GPU. The cases that
matter are caret blinks, one-line terminal output, and small edits. Do not
change the shipped no-GPU tuning until a benchmark shows a real improvement.

## Why this path

- The current renderer redraws and presents the whole window for every frame.
- wGPU gained damage-aware presentation in
  [gfx-rs/wgpu#10152](https://github.com/gfx-rs/wgpu/pull/10152), merged
  2026-11-14. It adds `Queue::present_with_damage`, but is not in the released
  wGPU 30.0.2, so this branch pins wGPU to
  `f061efb29a8b28236e0e634c5c0c23a88ff47cae`.
- The GPUI scene-damage diff is ported from Zed's open, unmerged
  [#67194](https://github.com/zed-industries/zed/pull/67194) rather than
  cherry-picked.

## What the branch does

- Diffs consecutive GPUI scenes into `SceneDamage::{Full, Rect, Unchanged}`.
- Enables the partial path only for X11 windows with a CPU adapter and a surface
  that supports `COPY_DST`. Hardware, Wayland, WASM, headless rendering,
  `render_to_image`, and non-unit scale factors stay on the full-frame path.
- Renders damaged regions with a scissor into a persistent target, copies the
  complete target into the swapchain, and passes the damage rect to
  `present_with_damage`. The full copy is still required; see
  [gfx-rs/wgpu#11038](https://github.com/gfx-rs/wgpu/issues/11038).
- Provides `HERDR_APP_DAMAGE_TRACKING=0` to restore the full-frame path.
- Forces full damage on resize, transparency/subpixel changes, surface
  reconfigure, atlas generation changes, and device recovery.

## Status

- Branch: `perf/x11-present-with-damage`.
- Fork PR: https://github.com/shawn111/herdr-app/pull/2.
- Initial CI: failed on Linux wGPU 30 API changes and stale third-party notices.
- Local `cargo check -p gpui-pre-wgpu`: passing after the API fixes.
- Third-party notices: regenerated with `cargo-about` 0.9.3 and Bun.
- No performance improvement has been proven yet.

## Next checks

1. Push the CI fixes and get all platform jobs green.
2. Use `GPUI_WGPU_FRAME_LOG=1` and `HERDR_APP_DAMAGE_TRACKING=0` to A/B the
   full-frame and partial paths in the X11 sandbox and on the target RHEL 8
   machine.
3. Verify whether X11 with llvmpipe/lavapipe actually consumes damage rects;
   backends without incremental presentation ignore them.
4. Keep hardware, macOS, Windows, Wayland, headless rendering, and
   `render_to_image` on the existing path unless a separate experiment proves
   otherwise.
5. Discuss any app-core or herdr-core API changes with Can before landing them;
   otherwise keep the experiment in vendored GPUI and mods.
