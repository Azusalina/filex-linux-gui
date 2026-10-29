# filex-linux-gui — description

## Architecture

- `src/main.rs` compiles `config`, `edit`, `files`, `git_actions`, `git_status`, `jobs`, `links`, `model`, `ordering`, `transfer` directly from `../filex/src` via `#[path]`, so tree model, ordering, Git status, transfers and config parsing stay identical to the TUI. `terminal_profile.rs` is a small stand-in (language + desktop flag). `FILEX_ROOT` defaults to `../filex` so `{filex}` opener paths resolve.
- `app.rs`: non-blocking port of the TUI `Ui`. Key → action resolution is copied from `tui.rs`; the TUI's blocking prompt / choose / confirm / pager / reorder loops are modals with continuations (copy/move replacement confirmations, Git menu → message → confirm, open-with, hidden items, trash).
- `layout.rs`: pixel version of `model::Layout` (classic and grouped `v`), plus the ancestor route.
- `view.rs`: header (breadcrumb, mode chips, client-side window controls), tree, dashboard (`q`), status line, modals. `paint.rs`: software painter on the 0RGB framebuffer. `font.rs`: embedded Geist / Geist Mono (SIL OFL, `assets/fonts/OFL.txt`) with fontconfig fallbacks (CJK etc.) memory-mapped on demand.
- `main.rs`: winit 0.30 + softbuffer, `ControlFlow::Wait` (redraw only on input, brief 200 ms polling while Git snapshots finish, short eased animations). `terminal.rs`: terminal programs run on a real PTY with a vt100 screen inside the same window. Reader events wake the window; exit refreshes the tree. `launch.rs` retains external shell detection and byte-exact POSIX command quoting.
- `dnd.rs`: outbound drag-and-drop. winit cannot be a drag source, so a private Wayland event queue on winit's own connection (`Backend::from_foreign_display`) binds `wl_seat` and `wl_data_device_manager`; a second `wl_pointer` records the button-press serial that `start_drag` needs. A press on a node that moves 8 px starts the drag; the source offers `text/uri-list` (RFC 8089 `file://` URIs, percent-encoded bytes) and `text/plain`, action copy. Marked items travel together when the dragged node is marked. The receiver accepts local URI lists, reads bounded payloads off the UI thread, and maps logical Wayland drop coordinates to physical node/breadcrumb hits. Internal drops move (Ctrl copies); external drops copy. All use the existing transfer planner and replacement confirmations. Outbound X11 drags remain unsupported.
- Design: constructivist minimalism, graphite / paper palettes with one red accent; configured filex colours are harmonised into the palette; clean Git states (P/C) are muted.

## Differences from the TUI

- Terminal openers, `@`, Git actions use one embedded PTY page; Ctrl+Shift+Tab switches between it and the tree without stopping the process. tmux right-pane reuse does not apply.
- `x` writes `FILEX_CWD_FILE` if set; started from a terminal it then opens `$SHELL` in the folder, otherwise it opens a terminal emulator there.
- Fcitx auto-English is replaced by enabling the input method only inside prompts.
- Search runs on Enter (not incrementally), like the TUI.

## Integration

- `scripts/install.sh` copies the release binary to `~/.local/bin/filex-gui`, writes `~/.local/share/applications/filex-gui.desktop` (Name `filex`, `StartupWMClass=filex-gui`, matching the Wayland app id) and `hicolor/scalable/apps/filex-gui.svg`, then refreshes the desktop database, KDE's sycoca and the icon cache. It does not register a MIME type, so the default file manager is unchanged. `--rebuild`, `--remove`.
- `../filex/filex.sh` accepts `--gui` anywhere in its arguments: it runs `~/.local/bin/filex-gui` (or this folder's build) with the remaining arguments via `setsid -f`, unsets `FILEX_CWD_FILE`, and returns to the shell.

## Validation (2026-10-25, KDE Plasma Wayland, scale 1.25)

- `cargo test --release`: 27 passed (13 GUI parity/layout/key/URI tests + 14 shared fileX module tests).
- 2026-10-26: the drag source initialises on the live KWin session (seat and data-device manager bound, window unaffected, RSS 22.7 MB); `desktop-file-validate` passes and KDE's service cache contains the entry; `filex --gui` through the Bash function returns immediately with the GUI detached.
- RSS 23 MB idle (heap ~2 MB, rest is the shared framebuffer), 27 MB with CJK fallback loaded, 29 MB after `E` expanding 5,000 nodes. Idle CPU: 0 ticks over 10 s; 1 thread. Frame 2–3 ms; layout rebuild ~19 ms at 5,000 nodes.
- Visual checks by screenshot: tree/route, dashboard, open-with, copy mode with `+` and marks, help, rename prompt, trash confirm, pager, grouped mode, light theme, CJK names, symlink target selection.

## Development

`FILEX_GUI_KEYS="right enter q"` replays space-separated key names at startup (`space`, `ctrl+f` aliases); `FILEX_GUI_TRACE=1` prints frame times.

## Limitations

- A complete drag into a browser was not exercised automatically: Wayland does not let a test synthesise the compositor's press serial, so the drop itself needs a manual check.
- Building needs `fileX` next to this folder (`../filex`). Independent clones must place `Azusalina/fileX` and `Azusalina/filex-linux-gui` side by side; this repository does not vendor the shared TUI modules. Repository publication does not create a GitHub binary release.
- Real mouse/keyboard input under X11 and other compositors untested; IME composition tested only through code paths, not live.

## History

- v1.0 (2026-10-26): GUI, outbound drag, launcher entry, `filex --gui`; committed to the hub (no GitHub release).
- 2026-10-27: Changed distribution from hub-only to the independent public
  `Azusalina/filex-linux-gui` repository. `products.json` registers this folder
  with the existing root pre-push coordinator. Product history is preserved;
  ordinary hub pushes publish committed GUI changes and skip an unchanged tip.
  Source remains in the parent repository, with no nested `.git`, and shared
  sibling `fileX` compilation remains unchanged. No runtime code or installed
  application is modified by this repository split.


## Feature extensions (2026-10-26)

- Fullscreen is the default (`[gui] fullscreen = true`); F11 switches modes. On a window-mode transition the existing information dashboard expands in fullscreen and collapses when windowed. Its manual q toggle remains available.
- Hover updates dashboard metadata without changing selection. Supported photos render below metadata, with EXIF orientation. PDFs render one page at a time through Poppler with previous/next controls. Normal files keep their existing text preview.
- `preview.rs` has a 150 ms hover debounce, one background render at a time, stale-result rejection by path/page, a 1400 px preview bound and bounded decoder allocation. Preview pixels remain in memory; no exports, persistent caches or source-file modifications. PDF processes time out after 20 seconds. Hidden dashboard does not schedule previews. Encrypted/damaged PDFs and missing Poppler report an inline error.
- `terminal.rs` owns the PTY session/process group, handles resize/SIGWINCH, VT colours/cursor/alternate screen, Unicode/IME and editor control keys. Normal exit restores the tree. Closing the GUI ends its PTY session. Terminal output and PDF loading do not block the GUI event loop.
- Optional `widget/` provides a transparent Plasma directory navigator, filesystem-notification updates, explicit GUI/TUI launchers and hover-folder x → external terminal. See its own description for differences from the full FileX model.
- Existing GUI palettes, typography, decoration, layout and animations are retained. Changes add only requested interactions and content in the existing dashboard / new terminal page.

### Current validation

Rust tests cover drop moves/copies, duplicate paths, replacement cancellation, recursive/root move rejection, byte-preserving local URI decoding, PTY input/resizing, actual Vim and Nano save/exit, photo pixels, two distinct rendered PDF pages, hover/selection independence, fullscreen config and existing shared FileX behavior. Widget native and QML checks are documented in `widget/description.md`.

Earlier RSS/CPU/screenshot measurements above describe the original build and have not been remeasured for these extensions. Native Wayland drag negotiation and final Firefox drops still require a live pointer gesture. X11 inbound file drops are supported by winit; outbound X11 remains unavailable. PDF viewing here is raster page navigation, without text selection/search or interactive PDF forms.

### Implementation references

- [Wayland data-device protocol](https://wayland.freedesktop.org/docs/html/apa.html)
- [vt100 terminal parser](https://docs.rs/vt100/0.18.1/vt100/)
- [image decoder](https://docs.rs/image/0.25.6/image/)

2026-10-26 extension validation: all 36 Rust tests passed, including real Vim/Nano save-and-exit and Poppler page rendering. The optional widget's native CTest, Qt 6 QML tests/lint, and isolated Plasma-host load passed. Both READMEs retain exactly the two required content sections.

The rebuilt release binary also passed an isolated Xvfb window boot/render smoke test (1360×860 frame). This checks startup and software rendering on X11; it does not verify Wayland pointer serials or a physical Firefox drop.

## 独立仓库迁移（2026-10-27）

源码及本地数据迁至 `/home/a/Documents/products/filex-linux-gui`，已有提交历史保留。现在使用本项目自己的 `.git` 和 `origin` 独立提交、推送；不再通过 `someProducts` 的 push hook 发布。本次目录／仓库重命名保留配置目录、应用名称和已有用户数据。


## Bottom terminal (2026-10-28)

Ctrl+J opens a bottom PTY pane using `tmux new-session -c <selected directory>` by default. It unsets inherited TMUX/TMUX_PANE values for the new client, without attaching to or changing an existing tmux session. Missing tmux produces an explicit message. The top tree and information dashboard keep their existing rendering in the remaining space; the terminal uses the existing fonts and palette.

The existing one-PTY lifecycle is retained: Ctrl+J docks an existing editor session instead of replacing it, and hiding preserves the same process. Click either pane or use Ctrl+Shift+Tab to route keyboard/IME input. Ctrl+J is reserved globally and ignores auto-repeat. Full-page editor openers retain their normal behavior when no terminal is already running. Terminal exit restores the full tree viewport; hiding or opening the pane does not change filesystem selection.


## Rename input workload (2026-10-28)

Previously `Shell::changed()` invalidated the complete tree layout after every prompt keystroke and IME update, causing all visible filenames to be measured again. Input-only updates now redraw the prompt and caret over a cached, already dimmed background. One temporary framebuffer is kept while editing (at most 32 MiB), then released when the prompt closes. Resize, scale/theme changes and actual tree operations invalidate this cache. Tree/Git preparation and Git polling are not restarted for text input. Submission/cancellation still invalidates the tree normally.

The regression workload uses a directory with 3,000 loaded files, checks pixel equality against a full redraw for typing and Chinese IME composition, verifies layout reuse and unchanged Git polling, and confirms a submitted rename appears in the rebuilt tree. This is a controlled render benchmark, not a measurement of the user's complete desktop latency. Device-attribute replies also distinguish primary and secondary terminal queries so tmux startup does not leak handshake bytes into its shell.

Terminal control reference: [XTerm control sequences](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html).


2026-10-28 validation: `cargo build --release --locked --offline` succeeded and all 41 release-mode tests passed, including tmux on an isolated socket, editor save/exit, dock visibility/focus/geometry and rename-cache rendering equality. An isolated Xvfb session injected actual Ctrl+J key events and verified that hide/show restored the same tmux client PID. A separate release benchmark with 3,000 loaded files measured six cached input redraws at 115.95 ms versus 437.01 ms for six forced full layout/background rebuilds (about 73% less time). This controlled render comparison does not measure full desktop input-to-display latency. Live KDE Wayland input remains a separate manual boundary.
