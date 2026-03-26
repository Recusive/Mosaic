# Plan 15: mosaic init & mosaic list

**Objective:** Implement the `mosaic init` command (scaffold `.mosaic/` with an example test) and `mosaic list` command (list available tests with metadata).

**Depends on:** Plan 02

**Estimated scope:** S (< 1 day)

## Context

`mosaic init` creates the `.mosaic/` directory and drops in an example test file that demonstrates the core API. `mosaic list` scans `.mosaic/` and shows available tests with last-modified timestamps. Both are simple filesystem operations with formatted output.

## Research Findings

No external research needed. Standard filesystem operations and string formatting.

## Files to Create

- `src/cli/init_cmd.rs` — mosaic init implementation
- `src/cli/list_cmd.rs` — mosaic list implementation

## Files to Modify

- `src/cli/mod.rs` — wire commands to implementations

## Implementation Details

### mosaic init (src/cli/init_cmd.rs)

```rust
use std::path::Path;
use colored::*;

const EXAMPLE_TEST: &str = r#"// .mosaic/example.test.ts
// This is an example Mosaic test. Edit it or create new tests in this directory.
//
// Run with:  mosaic test example
// Run all:   mosaic test
// List:      mosaic list

export default mosaic.test('example', async (t) => {

  await t.phase('Smoke Test', async () => {

    await t.step('Page is loaded', async () => {
      const title = document.title;
      t.assert(title.length > 0, 'Page should have a title');
      t.log(`Page title: ${title}`);
    });

    await t.step('App is interactive', async () => {
      // Wait for the app to finish loading
      await t.waitFor(() => {
        return document.querySelector('[data-testid]') !== null
            || document.querySelector('button') !== null
            || document.querySelector('input') !== null;
      }, { timeout: 10_000 });

      t.log('App has interactive elements');
    });

  });

});
"#;

pub fn run_init(cwd: &Path) -> anyhow::Result<()> {
    let mosaic_dir = cwd.join(".mosaic");

    if mosaic_dir.exists() {
        let existing = std::fs::read_dir(&mosaic_dir)?
            .filter_map(|e| e.ok())
            .filter(|e| e.file_name().to_string_lossy().ends_with(".test.ts"))
            .count();

        if existing > 0 {
            println!("{}", "[mosaic] .mosaic/ already exists with tests.".yellow());
            println!("  Use 'mosaic list' to see existing tests.");
            return Ok(());
        }
    }

    // Create directory
    std::fs::create_dir_all(&mosaic_dir)?;

    // Write example test (don't overwrite)
    let example_path = mosaic_dir.join("example.test.ts");
    if !example_path.exists() {
        std::fs::write(&example_path, EXAMPLE_TEST)?;
    }

    println!("{}", "[mosaic] Created .mosaic/".green());
    println!("{}", "[mosaic] Created .mosaic/example.test.ts".green());
    println!();
    println!("  Edit .mosaic/example.test.ts to write your first test, then run:");
    println!("    {}", "mosaic test example".bold());

    Ok(())
}
```

### mosaic list (src/cli/list_cmd.rs)

```rust
use std::path::Path;
use std::time::SystemTime;
use colored::*;

pub fn run_list(cwd: &Path) -> anyhow::Result<()> {
    let mosaic_dir = cwd.join(".mosaic");

    if !mosaic_dir.exists() {
        println!("{}", "[mosaic] No .mosaic/ directory found.".yellow());
        println!("  Run '{}' to create one.", "mosaic init".bold());
        return Ok(());
    }

    let mut tests: Vec<(String, String, SystemTime)> = vec![];

    for entry in std::fs::read_dir(&mosaic_dir)? {
        let entry = entry?;
        let name = entry.file_name().to_string_lossy().to_string();
        if name.ends_with(".test.ts") {
            let test_name = name.trim_end_matches(".test.ts").to_string();
            let file_path = format!(".mosaic/{}", name);
            let modified = entry.metadata()?.modified()?;
            tests.push((test_name, file_path, modified));
        }
    }

    if tests.is_empty() {
        println!("{}", "[mosaic] No tests found in .mosaic/".yellow());
        println!("  Create a .test.ts file in .mosaic/ to get started.");
        return Ok(());
    }

    // Sort alphabetically
    tests.sort_by(|a, b| a.0.cmp(&b.0));

    println!("{}", "[mosaic] Tests in .mosaic/".bold());
    println!();

    for (test_name, file_path, modified) in &tests {
        let relative = format_relative_time(*modified);
        println!(
            "  {:<24} {:<40} {}",
            test_name,
            file_path.dimmed(),
            relative.dimmed(),
        );
    }

    println!();
    println!("  {} tests found", tests.len());

    Ok(())
}

fn format_relative_time(time: SystemTime) -> String {
    let elapsed = SystemTime::now()
        .duration_since(time)
        .unwrap_or_default();

    let secs = elapsed.as_secs();
    if secs < 60 { return format!("{}s ago", secs); }
    if secs < 3600 { return format!("{}m ago", secs / 60); }
    if secs < 86400 { return format!("{}h ago", secs / 3600); }
    format!("{}d ago", secs / 86400)
}
```

### Wiring in src/cli/mod.rs

```rust
Commands::Init => init_cmd::run_init(&cwd),
Commands::List => list_cmd::run_list(&cwd),
```

## Acceptance Criteria

- [ ] `mosaic init` creates `.mosaic/` directory if it doesn't exist
- [ ] `mosaic init` creates `.mosaic/example.test.ts` with a working example test
- [ ] `mosaic init` does not overwrite existing tests
- [ ] `mosaic init` warns if `.mosaic/` already contains tests
- [ ] Example test demonstrates: `mosaic.test()`, `t.phase()`, `t.step()`, `t.assert()`, `t.waitFor()`, `t.log()`
- [ ] Example test includes comments with usage instructions
- [ ] `mosaic list` shows test names, file paths, and relative modification times
- [ ] `mosaic list` handles missing `.mosaic/` directory gracefully
- [ ] `mosaic list` handles empty `.mosaic/` directory gracefully
- [ ] Tests are sorted alphabetically
- [ ] Relative time formatting: "Xs ago", "Xm ago", "Xh ago", "Xd ago"
- [ ] Both commands work from any directory within the project (uses cwd)
