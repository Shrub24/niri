# niri Copilot Instructions

## Project Overview

niri is a scrollable-tiling Wayland compositor built with Rust and Smithay. Windows arrange horizontally in columns on an infinite strip, with dynamic vertical workspaces per monitor. This is not a window manager overlay—it's a full compositor handling display, input, and Wayland protocols.

## Architecture

### Workspace Structure

- **Main crate (`src/`)**: Core compositor built around `src/niri.rs` (6490 lines)—the central state machine
- **`niri-config/`**: KDL config parsing with split types (`Layout`/`LayoutPart`) for multi-file includes
- **`niri-ipc/`**: IPC types following niri version (NOT semver-stable; use exact versions like `=25.8.0`)
- **`niri-visual-tests/`**: GTK app for visual testing of layout/animations with mock windows

### Key Modules

- **`src/layout/`**: Scrollable tiling logic with extensive property-based tests
- **`src/backend/`**: Three backends (TTY/DRM, Winit nested, Headless)
- **`src/protocols/`**: Wayland protocol implementations (screencopy, gamma-control, foreign-toplevel, etc.)
- **`src/handlers/`**: Smithay protocol handlers
- **`src/ui/`**: Built-in UI (screenshot overlay, exit dialog, config errors)

## Development Workflows

### Building & Testing

```bash
# Run ALL tests (including sub-crates)
cargo test --all

# Run slow randomized property tests (do this before pushing)
env RUN_SLOW_TESTS=1 PROPTEST_CASES=200000 PROPTEST_MAX_GLOBAL_REJECTS=200000 \
    RUST_BACKTRACE=1 cargo test --release --all

# Format code (nightly required)
cargo +nightly fmt --all
```

### Running Locally

- Nested in window: `cargo run --release`
- TTY testing: Switch to a different TTY and run niri there
- Main compositor: Install normally, then `sudo cp ./target/release/niri /usr/bin/niri`

### Tracy Profiling

Instrument functions with `tracy_client::span!()`:

```rust
pub fn some_function() {
    let _span = tracy_client::span!("some_function");
    // function body
}
```

Build with features:
- On-demand: `--features=profile-with-tracy-ondemand` (can run as main compositor)
- Always-on: `--features=profile-with-tracy` (startup profiling, not for daily use)
- Allocations: Add `--features=profile-with-tracy-allocations`

## Code Patterns

### Logging Levels (via `tracing`)

- `error!`: Bugs/programming errors (always indicates a bug)
- `warn!`: User/hardware issues (e.g., config errors)
- `info!`: Essential messages (should not overwhelm users)
- `debug!`: Less critical operational info
- `trace!`: Compiled out of release builds

### Testing Patterns

- **Property tests**: When adding layout operations, add to `Op` enum in `src/layout/mod.rs` and `every_op` arrays
- **Config tests**: New config options must be covered in parsing tests
- **Visual tests**: Use `niri-visual-tests` for animation/rendering verification
- **Client-server tests**: Add to `src/tests/` for complex Wayland interactions

### Error Handling

Use `error!` logging + recovery instead of `unwrap()` where possible—compositor crashes kill the entire session.

## Critical Conventions

### Code Style

- Rustfmt config: `imports_granularity = "Module"`, `group_imports = "StdExternalCrate"`
- Comment wrap width: 100 chars
- Clippy allows `new_without_default` and ignores interior mutability for Smithay types

### Configuration

- Config is KDL format (see `resources/default-config.kdl`)
- Config types split for includes: `Foo` (final) and `FooPart` (parsed chunk)
- Defaults in `Default` impls should match `default-config.kdl`
- Document new options in wiki with examples

### Protocols

- Custom protocols in `src/protocols/` (e.g., `screencopy.rs`, `foreign_toplevel.rs`)
- Handlers in `src/handlers/` interface with Smithay
- Check [wayland.app](https://wayland.app) for protocol support status

### Smithay Integration

- Using smithay from git (not crates.io)
- Multi-backend support: TTY/DRM, Winit, Headless
- Renderer: GLES + Pixman fallback

## Pull Request Guidelines

- **Scope**: One feature/fix per PR; split self-contained commits
- **Testing**: Run slow tests, test edge cases manually, check animations
- **Documentation**: Update wiki for new config options
- **Squashing**: Address review comments by squashing into relevant commits
- **Rebasing**: Rebase (don't merge) when updating from main
- **History**: Clean history required before merge (see [git-rebase.io](https://git-rebase.io/))

## Common Pitfalls

- Forgetting to run `cargo test --all` (sub-crate tests)
- Not testing on actual hardware/TTY (nested Winit behaves differently)
- Breaking config backwards compatibility without migration path
- Adding `unwrap()` instead of `error!` recovery
- Not adding layout operations to randomized tests
- Leaving commits with "fix review comments" messages
