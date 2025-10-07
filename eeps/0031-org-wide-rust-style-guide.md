---
EEP: 31
Title: Org-Wide Rust Style Guide
Author: nrdxp
Sponsor:
EEP-Delegate:
Discussions-To: https://github.com/ekala-project/eeps/pull/31
Status: Draft
Type: Standards Track
Topic: Code Style
Requires:
Created: 2025-10-07
Post-History:
Replaces:
Superseded-By:
Resolution:
---

# Motivation

With the growing number of Rust projects within the ekala organization, there is a need to establish a consistent and unified style guide. A consistent style reduces cognitive overhead, improves readability, and simplifies collaboration, allowing developers to move between projects seamlessly. This proposal aims to formalize a standard Rust style guide based on the successful conventions already established in this project.

# Detailed implementation / Specification

This EEP proposes the adoption of a standardized style guide and formatting configuration for all Rust projects across the organization.

### 1. Official Style Guide

The following style guide (inspired by eka's [`STYLE_GUIDE.md`](https://github.com/ekala-project/eka/blob/master/STYLE_GUIDE.md)) will be adopted as the canonical style guide for all Rust projects. It codifies conventions for file structure, item ordering, sorting rules, and documentation, which are not covered by automated tooling.

#### 1.1. File Structure and Item Order

All Rust module files (`.rs`) must follow a strict top-level item order to ensure predictability and ease of navigation. The canonical order is as follows:

1.  **Module-level documentation (`//!`)**: Explains the purpose and scope of the module.
2.  **Outer attributes (`#![...]`)**: Compiler directives like `#![deny(missing_docs)]`.
3.  **`use` declarations**: External and internal imports.
4.  **Public re-exports (`pub use`)**: Items re-exported from other modules.
5.  **Submodules (`mod`)**: Child module declarations.
6.  **Constants (`const`)**: Compile-time constants.
7.  **Static variables (`static`)**: Globally allocated variables.
8.  **Types**: `struct`, `enum`, and `type` aliases.
9.  **Traits**: Trait definitions.
10. **Trait implementations and `impl` blocks**: Implementations of traits and inherent methods.
11. **Free-standing functions**: Module-level functions.
12. **Tests (`#[cfg(test)]` modules)**: Unit and integration tests for the module.

#### 1.2. Sorting and Grouping Rules

Within each category, items must be sorted to maintain a consistent structure.

##### `use` Declarations

`use` declarations are grouped in the following order, with each group sorted alphabetically:

1.  **`std`**: Standard library imports.
2.  **External Crates**: Third-party dependencies.
3.  **Local Modules**: Project-internal imports, starting with `crate::` or `super::`.

Example:

```rust
use std::collections::HashMap;
use std::path::PathBuf;

use anyhow::Result;
use log::info;

use crate::core::Atom;
use super::utils::helper_function;
```

##### Other Items

All other top-level items—including modules, constants, types, traits, and functions—must be sorted alphabetically by their identifier.

##### Visibility

Within any given category, **public (`pub`) items must always be placed before private items**. This rule applies before alphabetical sorting. For example, a public function `alpha` would come before a private function `beta`, but also before a public function `zeta`.

Example:

```rust
// Public items first, sorted alphabetically
pub const MAX_RETRIES: u32 = 3;
pub fn get_config() -> Config { /* ... */ }

// Private items next, sorted alphabetically
const DEFAULT_TIMEOUT: u64 = 10;
fn process_data() { /* ... */ }
```

#### 1.3. Documentation Comments

Clear and comprehensive documentation is mandatory for maintaining a high-quality codebase.

- **All public items** (modules, functions, types, traits, constants) must have descriptive documentation comments (`///`).
- **Module-level documentation (`//!`)** is required for every module. It should provide a high-level overview of the module's responsibilities and how it fits into the larger system.
- Comments should be clear, concise, and sufficient for a developer to understand the item's purpose and usage without needing to read the underlying source code.

#### 1.4. General Guidelines

- **Use `rustfmt`**: This proposal outlines an opinionated `rustfmt.toml` configuration to enforce a consistent code style. Running `cargo fmt` will automatically handle much of the formatting for you. However, the guidelines in this document (especially regarding item order and documentation) must still be followed manually.
- **Preserve Semantics**: Never alter the meaning or behavior of the code purely for the sake of conforming to style.
- **Preserve Comments and Attributes**: When reordering items, ensure that all associated documentation, comments (`//`), and attributes (`#[...]`) are moved along with the item they describe.

### 2. Standard `rustfmt` Configuration

To ensure consistent, automated code formatting, all Rust projects must use the following `.rustfmt.toml` configuration:

```toml
# See https://rust-lang.github.io/rustfmt/ for configuration options

# Use the 2021 edition of Rust
edition                  = "2021"
# Enforce Unix-style line endings
newline_style            = "unix"
# Use shorthand for struct field initialization
use_field_init_shorthand = true
# Use `?` instead of `try!`
use_try_shorthand        = true

# Enable unstable features for more opinionated formatting.
# This allows for features like macro formatting and import grouping.
unstable_features = true

# Set the maximum line width for comments
comment_width               = 100
# Condense wildcard suffixes in imports (e.g., `use std::io::{self, Read};`)
condense_wildcard_suffixes  = true
# Error if a line exceeds the maximum width
error_on_line_overflow      = true
# Format code snippets in documentation comments
format_code_in_doc_comments = true
# Format macro bodies
format_macro_bodies         = true
# Format macro matchers
format_macro_matchers       = true
# Format string literals
format_strings              = true
# Group imports into three sections: `std`, external crates, and local modules
group_imports               = "StdExternalCrate"
# Control the granularity of imports
imports_granularity         = "Module"
# Add a trailing comma to match blocks
match_block_trailing_comma  = true
# Normalize documentation attributes
normalize_doc_attributes    = true
# Reorder `impl` items
reorder_impl_items          = true
# Use the 2024 style edition for the latest formatting rules
style_edition               = "2024"
# Wrap comments to the specified width
wrap_comments               = true
```

### 3. Enforcement

Compliance with the style guide and `rustfmt` configuration should be enforced through automated checks in CI. We recommend using a tool like `treefmt` to run `rustfmt --check` and other formatters.

# Example usage

The following code snippet demonstrates the impact of applying the proposed style guide.

**Before:**

```rust
use crate::utils::helper;
use std::collections::HashMap;
use anyhow::Result;

fn my_func() -> Result<()> {
    Ok(())
}

pub struct MyStruct {
    pub field: String,
}
```

**After:**

```rust
use std::collections::HashMap;

use anyhow::Result;

use crate::utils::helper;

pub struct MyStruct {
    pub field: String,
}

fn my_func() -> Result<()> {
    Ok(())
}
```

# Prior art

This proposal is based on the existing [STYLE_GUIDE.md](https://github.com/ekala-project/eka/blob/master/STYLE_GUIDE.md) and `.rustfmt.toml` that have been successfully used in this project, proving their effectiveness in maintaining a clean and consistent codebase.

# Unresolved questions

- What is the process and timeline for rolling out these changes to existing Rust projects?
- How will we handle project-specific exceptions to the style guide, if any?
- Who will be responsible for maintaining the central style guide and `rustfmt` configuration?
- Not all style requirements are currently automatable by rustfmt, and will require up front investment from reviewers. How should we best coordinate this?

# Future work

- Create a centralized repository or shared configuration package for the `STYLE_GUIDE.md` and `.rustfmt.toml` to simplify adoption and updates.
- Implement CI checks in all existing and future Rust projects to enforce compliance.
- Develop a process for proposing and reviewing changes to the style guide.

# Acknowledgements

Thanks to the original authors of the `STYLE_GUIDE.md` for establishing a strong foundation for this proposal. Special thanks to @jonringer for his ongoing efforts in eka-ci and in driving ekapkgs forward.
