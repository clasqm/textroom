# TextRoom: Qt 4 to Qt 5 conversion

This document summarises the work done to bring TextRoom (last released against
Qt 4) up to Qt 5, with Haiku as the target platform. All changes are on the
`converttoqt5` branch.

## Build system

The qmake project file (`textroom.pro`) was replaced by a `CMakeLists.txt` that
uses Qt 5 Widgets, CMake's `AUTOMOC`, `AUTOUIC` and `AUTORCC` (with `ui/` added
to `AUTOUIC_SEARCH_PATHS`, since the forms do not sit next to the sources), and
a single compile of `resource/textroom.qrc`. The install prefix defaults to
`$HOME/config/non-packaged` on Haiku rather than `/usr/local`, because the
system tree is read-only there. The workflow is the usual
`mkdir build && cd build && cmake .. && make`.

SDL is now optional. CMake probes for a matching SDL2 + SDL2_mixer pair first,
then SDL 1.2 + SDL_mixer, and defines `HAVE_SDL` only when a complete pair is
found; `-DTEXTROOM_SOUND=OFF` builds without any SDL dependency. The sound
files are no longer loaded from a hardcoded `/usr/local/share/textroom` path —
`findSoundFile()` looks next to the binary, in the data directory and in
`resource/sounds/` at runtime.

## Source changes

* Qt 5 split the monolithic `QtGui` module, so widgets moved to `<QtWidgets>`
  and the per-class includes were updated in every source and header.
* `QWidget` constructors take `Qt::WindowFlags` instead of `Qt::WFlags`.
* Palette, style and text-alignment calls were updated to the Qt 5 signatures
  (`setColor`, `setStyle(QStringLiteral("Fusion"))` because `plastique` is
  gone, `Qt::AlignLeft` and friends).
* The old `QShortcut(key, parent, SLOT(...))` constructor was removed in Qt 5.
  A small `addShortcut()` template now creates each shortcut and connects it
  with the pointer-to-member syntax.
* Old-style `SIGNAL`/`SLOT` connections were replaced with modern `connect()`
  calls, and `setTabStopWidth()` (deprecated) became `setTabStopDistance()`.
* The unused `src/dialog.cpp`/`dialog.h` pair was deleted: it included a
  `ui_dialog.h` that does not exist and was never part of the build.

## Behaviour fixes found while porting

* `readSettings()` returned early when no settings file existed, so on a first
  run the designer palette (yellow on white) and the expanding spacers stayed
  in place and squeezed the editor into a small centred block. It now creates
  the settings file and carries on, so colours, fonts, spacers and the default
  editor width (the screen width) are always applied.
* The previous session's filename was adopted inside `readSettings()`, which
  also runs when the settings file changes and from the Options dialog, so the
  window title could show a file that was never opened. Restoring the last
  document is now an explicit step in the constructor, with a fallback to an
  untitled document.

## Documentation

Haiku's Qt maps the `Ctrl` modifier to Alt, so the shortcut lists in `README`,
`docs/HELP.SHORTCUTS` and the F1 help screen now all say Alt and match the
bindings actually installed in `src/textroom.cpp`. The F1 help text was
enlarged to 10 pt monospace with a fixed-width key column so the F3/Shift+F3
rows line up.

## Verification

The UI files were checked with an XML parser and the shortcut lists compared
against the source; the code was compiled and run manually on Haiku, where the
build succeeds and typing sounds are enabled.
