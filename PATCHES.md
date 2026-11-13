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
`gpui-fast-macos` 0.1.14 with four changes: its library path points at
`crates/gpui_macos/src`, `publish = false`, the `include` list is gone (it names paths
relative to the published crate), and it has a `[workspace]` of its own. Cargo's
generated header gives way to a comment saying what the file is. An app replaces the published crate
with `[patch.crates-io] gpui-fast-macos = { git = …, rev = … }` and keeps every other
gpui-fast crate from crates.io, so there is one copy of `gpui`. The published source
differs from `crates/gpui_macos/src` at `v0.1.14` only by a licence header line.

## Ported

| NativeGPUI | What it does | Port |
| --- | --- | --- |
| 0003 | `run` calls `finishLaunching` and the app's launch callback itself, rather than waiting for LaunchServices' open-application event (about 35 ms idle on an M4). | Same logic, through `msg_send!`. |
| 0004 | Opt-in launch trace (`GPUI_LAUNCH_TRACE=1`, or a file path) and the public `on_first_drawable_completed` / `on_first_present`, released 250 ms after the first committed drawable or at the third. | The main-thread check uses gpui's `main_thread_marker`. Commits are counted on gpui-fast's `draw_composed` path too. As in the original, a draw that got no drawable still counts. |
| 0005 | `defer_initial_window_show`: windows opened in the launch callback stay hidden until it returns, then each is ordered front and draws its first frame in one `CATransaction`. | The display link starts through the window's foreground executor, since the state is an `Rc` and can't cross `DispatchQueue`; that executor dispatches to the main queue too. Presents-with-transaction goes back off after the first frame, since 0002 isn't here to keep it on. New: called after launch it does nothing, where the original left every later window hidden. |
| 0007 | The launch work `run` now does runs in its own autorelease pool, so a window the callback opens is freed when it closes. | `objc2::rc::autoreleasepool`. |
| 0001 | Windows accept file URLs and browser URL and image drags, not only Finder's file list. | Rewritten for objc2: typed `NSString`s and raw `msg_send!`, as the surrounding code does. `public.png` and `public.tiff` stand in for `NSPasteboardTypePNG` and `NSPasteboardTypeTIFF`. Changed: image data is written when the drag is dropped, not when it enters (gpui gets the path it will have on entry), and a failed write ends the drag instead of submitting it. File promises are left out: gpui takes a drag's paths on entry, so a promise would be fulfilled on every hover, writing into a new temp directory each time. |

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
first frame presented inside one transaction. In a macOS 27.2.1 VM, driven through
cua with a probe window that logs drops (FAIM's `gui/crates/faim-gui-macos/vm/`), a
Safari link arrived as its URL, a Safari image hovered over the window wrote nothing
and dropped on it was written as a valid PNG, and a Finder file arrived as its path.

## Updating

Rebase onto the gpui-fast release FAIM moves to and keep these commits. Then copy that
release's published `gpui-fast-macos/Cargo.toml` (in `~/.cargo/registry/src/*/`) over
`faim/gpui-fast-macos/Cargo.toml` and make the four changes above again. Check that the
published `src/` still matches `crates/gpui_macos/src` at the release tag apart from
licence headers.
