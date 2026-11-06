# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Documentation

- Put documentation in `docs/`.
- Keep a running debugging guide in `docs/debugging.md`. When you solve a problem that was not obvious, add it there as symptom, cause and fix. Read that file before debugging a build or runtime failure.

## Commands

First-time setup:

```sh
rustup target add wasm32-unknown-unknown
npm ci
cargo install trunk --locked
```

`cargo make <task>` (from `Makefile.toml`) chains the steps below, but `cargo-make` is optional. Order matters, because each step consumes the output of the one before it:

```sh
npm run build                            # index.css -> target/index.css (`release` to minify)
trunk build --dist target/dist           # add --release; compiles the frontend, bundles target/dist
cargo build -p app                       # add --release; compiles the desktop app
```

Run the app with a sample save (debug builds read `databases/` from the current directory, so run from the repo root):

```sh
cargo run -p app -- test/ME2Save.pcsav
```

Test and lint:

```sh
cargo test --all                         # the tests read test/ by relative path
cargo test -p trilogy-save-editor mass_effect_2::test::deserialize_serialize_legendary
cargo clippy --all
cargo check --target wasm32-unknown-unknown   # the target the frontend really ships on
```

- Prefer these over `cargo make test` and `cargo make outdated`, which run `cargo update` first and rewrite `Cargo.lock`. See `docs/debugging.md` for why that can break the build.
- `cargo make clippy` runs `cargo fmt --all` first, which rewrites files.
- `cargo make cook` builds the Windows installer with Inno Setup (`iscc`).
- CI (`.github/workflows/CI.yml`) is disabled and only runs when started by hand, so run the tests locally.

## Architecture

The workspace has three crates that form two programs, connected by an RPC bridge:

- **Root crate (`src/`)**: the UI and all save logic, written with Yew. It ships as wasm inside a webview. It also compiles natively, but only so `cargo test` can run.
- **`app/`**: the native wry/WebView2 shell. It owns the window, file dialogs, disk access and auto-update, and serves the frontend over the custom `tse://` protocol.
- **`macros/`**: the proc macros `#[rcize_fields]` and `#[derive(RawUi, RawUiRoot, RawUiChildren)]`.

Saves are parsed and written in wasm. The shell only moves bytes.

### Assets reach the app in different ways

- **Frontend:** `app` embeds `target/dist` with rust-embed. Release builds include it at compile time, so run `trunk` before `cargo build -p app`. Debug builds read the folder at runtime.
- **Databases:** `databases/*.ron` are not embedded. The frontend requests them over RPC when first needed. Debug builds resolve the path from the current directory and release builds from the exe's directory, so a release exe needs a `databases/` folder beside it. The installer copies it there.
- **CSS:** Tailwind scans `index.html` and `src/**/*.rs`, so a new class only exists after `npm run build`.

### RPC

A call goes `src/services/rpc.rs` -> `src/services/rpc.js` -> `window.rpc` (injected by wry) -> `app/src/rpc/mod.rs` -> `app/src/rpc/command.rs`. RPC therefore only works inside the app window, not in a browser tab.

To add a command, change all three Rust files: a wrapper in `src/services/rpc.rs`, an entry in the matching macro list in `app/src/rpc/mod.rs` (dispatch is by function name), and the function in `command.rs`. `RpcFile`, `Base64File` and `DialogParams` are defined separately on each side and must stay in sync. File contents cross the bridge as base64.

### Save format

`src/unreal/` holds custom serde `Deserializer` and `Serializer` implementations for the binary format. There are no field names or tags in the file, which has these consequences:

- **Struct field order in `src/save_data/` is the binary layout.** Never reorder fields.
- `Dummy<LEN>` preserves bytes whose meaning is unknown, and `List<T>` is a sequence written without a length prefix.
- Version and magic-number types have custom `Deserialize` impls that reject wrong values. `SaveHandlerProvider::deserialize` in `src/services/save_handler.rs` detects the game by trying each header type in turn, so the order of that chain matters.
- Writing is not symmetric across games. `SaveHandlerProvider::serialize` adds the CRC32 (BZIP2) checksum: appended for ME2, ME2LE and ME3, and written in place 12 bytes from the end for ME1LE. ME2 and ME3 pick big-endian Xbox 360 output from the `.xbsav` extension.
- Tests assert that a sample save round-trips byte for byte, so a layout mistake normally fails them. `misc/010 templates/` has 010 Editor templates for inspecting the format.

### Editable state and the raw editor

- `#[rcize_fields]` rewrites a struct's field types: primitives become `RcCell<T>`, everything else `RcRef<T>`, including inside `Vec`, `Option` and `IndexMap`. Fields starting with `_` are left alone. It generates accessors only for `pub` fields: `x()`/`set_x()` for primitives and `x()`/`x_mut()` otherwise.
- Components hold these shared handles and edit the save in place, so there is no central store or message per edit. Cloning a handle shares the data, and `PartialEq` compares pointers, not values.
- `RawUi` derives generate the Raw Data tab from the struct definition and skip `_` fields, so a new field appears there automatically. Hand-written views live in `src/gui/mass_effect_*`.
- `SaveHandler` (`src/services/save_handler.rs`) and `Databases` (`src/services/database.rs`) are provided through Yew context. A `Databases` getter returns `None` and starts the load if the database isn't loaded yet, and the component re-renders when it arrives.

### Versioning and changelog

- Set the version only in the root `Cargo.toml`. The root `build.rs` copies it into `app/Cargo.toml` and `InnoSetup.iss`.
- `CHANGELOG.md` is compiled into the app and parsed for display in `src/gui/app.rs`, so keep its existing format.
