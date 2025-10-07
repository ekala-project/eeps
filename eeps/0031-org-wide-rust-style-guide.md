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

The existing [STYLE_GUIDE.md](https://github.com/ekala-project/eka/blob/master/STYLE_GUIDE.md) will be adopted as the canonical style guide for all Rust projects. It codifies conventions for file structure, item ordering, sorting rules, and documentation, which are not covered by automated tooling.

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
