# 2026

## Added

- `corkey-key-files` / `corkey-signal-files` - variables of which key/signals to
  load by default (when not explicitly passed in)
- `corkey-register-key-file` / `corkey-register-signal-file` - functions which
  add to the above variables, meant to be used in an `(eval-after-load 'corkey)`
  by packages which provide additional signal or key overrides 

## Changed

- Drop the `a.el` dependency

# 2022

## Fixed

- When (re)loading keys/signals, if the key/signal file is already open in a
  buffer, preserve the position of point
- Support `which-key` remapping for minor-modes and evil states, by putting
  descriptions inside the keymaps

## Changed

- Resolve symlinks when loading/watching key/signal files, this ensures that
  changes are picked up by the watcher, even if the files are symlinks, e.g. in
  straight build directories
- Provide better error messages when parsing files
