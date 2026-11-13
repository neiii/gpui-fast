# The `faim` branch

FAIM's GUI builds on gpui-fast (neiii/faim-http, ADR-0017). This branch is
longbridge/gpui-fast at `v0.1.14` with NativeGPUI's `gpui_macos` patches ported onto
gpui-fast's `crates/gpui_macos`, one commit each. FAIM pins it by `rev`.

NativeGPUI (Cafekery/NativeGPUI) wrote its seven patches against zed's `gpui_macos` at
`90d8514` (9 Oct 2026). gpui-fast 0.1.14 imports zed at `d2a4fb6` (8 Nov 2026). In
between, zed moved `gpui_macos` from the `cocoa` crate to `objc2`, keeps window state
in an `Rc`, and paces frames with a `CAMetalDisplayLink` on macOS 14 and later. None of
the patches applied as written. Each one ported is rewritten for that code, keeping its
behaviour.

## How FAIM uses it

`faim/gpui-fast-macos/Cargo.toml` is the manifest crates.io publishes for
`gpui-fast-macos` 0.1.14, with its library path pointed at `crates/gpui_macos/src`,
`publish = false`, and a `[workspace]` of its own. An app replaces the published crate
with `[patch.crates-io] gpui-fast-macos = { git = …, rev = … }` and keeps every other
gpui-fast crate from crates.io, so there is one copy of `gpui`. The published source
differs from `crates/gpui_macos/src` at `v0.1.14` only by a licence header line.

## Ported

| NativeGPUI | What it does | Port |
| --- | --- | --- |
| 0003 | `run` calls `finishLaunching` and the app's launch callback itself, rather than waiting for LaunchServices' open-application event (about 35 ms idle on an M4). | Same logic, through `msg_send!`. |
| 0004 | Opt-in launch trace (`GPUI_LAUNCH_TRACE=1`, or a file path) and the public `on_first_drawable_completed` / `on_first_present`, released 250 ms after the first committed drawable or at the third. | The main-thread check uses gpui's `main_thread_marker`. Commits are counted on gpui-fast's `draw_composed` path too. |
| 0005 | `defer_initial_window_show`: windows opened in the launch callback stay hidden until it returns, then each is ordered front and draws its first frame in one `CATransaction`. | The display link starts through the window's foreground executor, since the state is an `Rc` and can't cross `DispatchQueue`. Presents-with-transaction goes back off after the first frame, since 0002 isn't here to keep it on. |
| 0007 | The launch work `run` now does runs in its own autorelease pool, so a window the callback opens is freed when it closes. | `objc2::rc::autoreleasepool`. |
| 0001 | Windows accept browser URL and image drags and file promises, not only Finder's file list. | Rewritten for objc2: typed `NSString`s and raw `msg_send!`, as the surrounding code does. `public.png` and `public.tiff` stand in for `NSPasteboardTypePNG` and `NSPasteboardTypeTIFF`. |

## Not ported

- **0006 (stop an idle window's display link).** Upstream already does it on the
  Metal path: `MetalFrames::end_frame` pauses the link after a frame that drew
  nothing and asked for nothing, and `frame_waker` / `schedule_frame` wake it.
- **0002 (draw inside input events, always present with the transaction).** It
  tunes the `CVDisplayLink` `step` path, which gpui-fast now uses only before macOS
  14. On the Metal path a draw inside an input event would take `nextDrawable` while
  the link holds drawables too. Porting it means redesigning that path and measuring
  input-to-frame latency first.

## Checked

On macOS 27 (Apple silicon), with FAIM's macOS app built against this branch:
`GPUI_LAUNCH_TRACE=1` shows the launch callback starting right after
`finishLaunching`, and with `defer_initial_window_show` the window is ordered and its
first frame presented inside one transaction. The drag-type code compiles but hasn't
been exercised with a real drag.

## Updating

Rebase onto the gpui-fast release FAIM moves to, keep these commits, and copy that
release's published `gpui-fast-macos` manifest over `faim/gpui-fast-macos/Cargo.toml`
with the same three changes.
