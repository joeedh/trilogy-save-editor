# Debugging guide

A running list of problems that were not obvious, and how they were solved. Add an entry whenever you solve a new one: give the symptom (with the exact error text so it can be searched for), the cause and the fix. Say so if a cause comes from reading the code and hasn't been reproduced.

## Building

### `older versions of the wasm-bindgen crate are incompatible with current versions of Rust`

- **Cause:** rustc rejects `wasm-bindgen` older than 0.2.88.
- **Fix:** move `wasm-bindgen` in `Cargo.lock`. A plain `cargo update -p wasm-bindgen` stops at 0.2.84 because of other locked crates, so pin the version with `--precise`, which moves `proc-macro2` and `quote` too:

  ```sh
  cargo update -p once_cell
  cargo update -p wasm-bindgen --precise 0.2.100
  cargo update -p js-sys --precise 0.3.77
  cargo update -p web-sys --precise 0.3.77
  cargo update -p wasm-bindgen-futures --precise 0.4.50
  ```

### `struct takes 0 generic arguments but 1 generic argument was supplied`, pointing at `#[wasm_bindgen]` in `src/services/rpc.rs`

- **Cause:** `wasm-bindgen` and `js-sys` are mismatched. Newer `wasm-bindgen` generates a generic `Promise<T>`, which older `js-sys` doesn't have.
- **Fix:** these crates must move as a matched set. Use the versions above.

### `failed to select a version for futures-util` when updating `js-sys`

- **Cause:** `js-sys` 0.3.107 needs a newer `futures-util` than the one locked by `reqwest`.
- **Fix:** stay on the 0.2.100 / 0.3.77 set, which doesn't have this dependency.

### `E0793: reference to field of packed struct is unaligned` in `ntapi`

- **Cause:** a bug in `ntapi` 0.3.x, which 0.3.7 doesn't fix. `tokio` 1.15 pulled it in through `mio` 0.7.
- **Fix:** `cargo update -p tokio --precise 1.38.2`, which moves to `mio` 0.8 and drops `ntapi`.

### The build breaks after `cargo make test` or `cargo make outdated`

- **Cause:** both run `cargo update` first. On 2026-11-05 a dry run showed this would move 212 packages, including `wasm-bindgen` to 0.2.130 and `tokio` to 1.54. That result has not been built.
- **Fix:** run `git checkout Cargo.lock` to restore the working lockfile, and run `cargo test --all` directly.

### `cargo fmt --check` reports `Incorrect newline style` on every file

- **Cause:** `.rustfmt.toml` asks for Unix newlines, but with `core.autocrlf=true` a Windows checkout has CRLF on disk. The files stored in git use LF.
- **Fix:** this is harmless. Ignore those lines and look for real formatting differences.

## Running

### Plot, Raw Plot, ME1LE General or ME1LE Inventory fail in a release exe

Found by reading the code, not yet reproduced.

- **Cause:** these views load `databases/*.ron`, and release builds look for that folder next to the exe. `target/release/` doesn't have one. Debug builds use the current directory, and the installer copies the folder into place.
- **Fix:** copy `databases/` next to the exe: `cp -r databases target/release/`.

### Checking that the built app really works

- The window starts hidden, so finding the process, or a visible window, doesn't prove the UI rendered. Take a screenshot and look at it.
- `Process.MainWindowHandle` can return a small untitled helper window (258x71) rather than the real one. Enumerate the process's top-level windows and pick the one titled `Trilogy Save Editor*`.
- `PrintWindow` with flag 2 (`PW_RENDERFULLCONTENT`) captures the WebView2 content correctly.

## Save files

### A round-trip test fails with `assert!(input == output)`

- **Cause:** usually a struct in `src/save_data/` that doesn't match the file layout: wrong field order, type or `Dummy` length.
- **Fix:** the tests in `src/save_data/mass_effect_2/mod.rs` keep a commented-out loop that compares the files 4 bytes at a time and reports the first differing offset. Use that offset with the matching template in `misc/010 templates/`.

### `Unexpected end of file, some data in your save are unexpected or your save is corrupted`

- **Cause:** this is `unreal::Error::Eof`. The deserializer read past the end of the file, which can also mean the structs don't match the layout, not just that the save is damaged.
