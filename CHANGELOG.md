# main

## Added

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
