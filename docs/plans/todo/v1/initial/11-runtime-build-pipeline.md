# Plan 11: Runtime Build Pipeline

**Objective:** Set up the build pipeline that compiles the TypeScript runtime into a single IIFE JavaScript file and embeds it in the Rust binary via `build.rs` and `include_str!()`.

**Depends on:** Plan 09

**Estimated scope:** S (< 1 day)

## Context

The TypeScript runtime needs to be compiled to a single JS file and embedded in the Rust binary. This happens at `cargo build` time via `build.rs`, which runs `esbuild` to produce the IIFE bundle, and then `include_str!()` bakes it into the binary.

## Research Findings

- **esbuild 0.27.4** — `--bundle --format=iife --global-name=__mosaic --platform=browser --target=es2022 --minify` produces exactly what's needed.
- **`include_str!()`** — zero-dep, bakes the file into the `.rodata` segment. Perfect for a single ~20-50KB JS file.
- **`build.rs`** — `cargo::rerun-if-changed=runtime/src/` ensures the runtime is only rebuilt when TypeScript source changes.
- **`build.rs` requires Node.js** — acceptable since Mosaic tests JavaScript applications. Document as a build dependency.

## Files to Create

- `runtime/dist/.gitkeep` — ensure dist directory exists in repo (actual built file is gitignored)

## Files to Modify

- `build.rs` — add esbuild invocation
- `src/runtime/mod.rs` — embed the built JS file with `include_str!()`
- `.gitignore` — ignore `runtime/dist/*.js`

## Implementation Details

### build.rs

```rust
use std::process::Command;

fn main() {
    // Re-run build.rs when TypeScript source changes
    println!("cargo::rerun-if-changed=runtime/src/");
    println!("cargo::rerun-if-changed=runtime/package.json");
    println!("cargo::rerun-if-changed=runtime/tsconfig.json");

    // Check if runtime/dist/runtime.js exists already (for CI without Node.js)
    let runtime_path = std::path::Path::new("runtime/dist/runtime.js");
    if runtime_path.exists() {
        // Check if source is newer than output
        let src_modified = latest_modified("runtime/src");
        let out_modified = std::fs::metadata(runtime_path)
            .and_then(|m| m.modified())
            .ok();

        if let (Some(src), Some(out)) = (src_modified, out_modified) {
            if out > src {
                return; // Output is newer, skip build
            }
        }
    }

    // Run esbuild via npm
    let status = Command::new("npm")
        .args(["run", "build"])
        .current_dir("runtime")
        .status();

    match status {
        Ok(s) if s.success() => {},
        Ok(s) => panic!("TypeScript runtime build failed with exit code: {}", s),
        Err(e) => {
            // If npm is not available, check if the pre-built file exists
            if runtime_path.exists() {
                eprintln!("cargo:warning=npm not found, using pre-built runtime.js");
            } else {
                panic!(
                    "Failed to build TypeScript runtime: {}. \
                     Node.js and npm are required to build from source. \
                     Install Node.js: https://nodejs.org",
                    e
                );
            }
        }
    }
}

fn latest_modified(dir: &str) -> Option<std::time::SystemTime> {
    walkdir(std::path::Path::new(dir))
        .filter_map(|entry| {
            std::fs::metadata(&entry)
                .and_then(|m| m.modified())
                .ok()
        })
        .max()
}

fn walkdir(path: &std::path::Path) -> Vec<std::path::PathBuf> {
    let mut results = vec![];
    if let Ok(entries) = std::fs::read_dir(path) {
        for entry in entries.flatten() {
            let path = entry.path();
            if path.is_dir() {
                results.extend(walkdir(&path));
            } else {
                results.push(path);
            }
        }
    }
    results
}
```

### src/runtime/mod.rs

```rust
/// The pre-compiled Mosaic runtime JavaScript.
/// This IIFE sets up the global `mosaic` object with the full test API.
pub const RUNTIME_JS: &str = include_str!(concat!(
    env!("CARGO_MANIFEST_DIR"),
    "/runtime/dist/runtime.js"
));
```

### .gitignore additions

```
runtime/dist/*.js
runtime/dist/*.map
runtime/node_modules/
```

### Initial setup for developers

```bash
cd runtime && npm install
```

This installs esbuild and TypeScript as dev dependencies. After that, `cargo build` handles everything automatically.

## Acceptance Criteria

- [ ] `cargo build` runs `npm run build` in the `runtime/` directory
- [ ] `build.rs` only re-runs when TypeScript source files change
- [ ] `include_str!()` successfully embeds the built JS file in the binary
- [ ] `runtime::RUNTIME_JS` is a valid `&str` containing the IIFE bundle
- [ ] The IIFE starts with `var __mosaic=` (confirming correct format)
- [ ] If Node.js is not installed and `runtime/dist/runtime.js` exists, build succeeds with a warning
- [ ] If Node.js is not installed and no pre-built file exists, build fails with a clear error
- [ ] `runtime/dist/*.js` is gitignored
- [ ] `runtime/node_modules/` is gitignored
- [ ] Full round trip: modify TypeScript → `cargo build` → binary contains updated runtime
