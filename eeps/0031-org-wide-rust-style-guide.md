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

# Configuring the Rust edition for the project
edition = "2021"
# Justification: Specifies the Rust edition to ensure compatibility with modern Rust features and syntax.
# The 2021 edition is chosen for its balance of stability and access to newer language features like
# improved pattern matching and const generics, which are widely supported in the current Rust ecosystem.
# This aligns with the style guide's goal of maintaining a consistent and modern codebase.

# Setting the newline style for cross-platform consistency
newline_style = "unix"
# Justification: Enforces Unix-style line endings (\n) to ensure consistency across platforms, especially
# important for open-source projects with contributors using different operating systems. This prevents
# formatting conflicts in version control and aligns with the style guide's emphasis on consistency.

# Enabling shorthand syntax for struct initialization
use_field_init_shorthand = true
# Justification: Allows concise struct initialization (e.g., `Struct { field }` instead of `Struct { field: field }`).
# This reduces visual clutter, improving readability, which supports the style guide's goal of reducing cognitive
# overhead for developers.

# Enabling shorthand syntax for try expressions
use_try_shorthand = true
# Justification: Simplifies error handling by allowing `?` instead of `try!` or explicit `match` expressions.
# This promotes concise code, aligning with the style guide's aim for readable and maintainable codebases,
# especially in error-prone areas.

# Enabling experimental rustfmt features
unstable_features = true
# Justification: Enables experimental rustfmt features that may provide additional formatting capabilities
# not yet available in stable releases. This is useful for an open-source project aiming to stay aligned
# with evolving Rust standards, ensuring forward-compatibility while adhering to the style guide's
# emphasis on consistency.

# Setting maximum width for comments
comment_width = 100
# Justification: Limits comment line length to 100 characters to ensure readability on standard displays
# and in code review tools. This supports the style guide's requirement for clear, concise documentation
# comments that are easy to read without excessive horizontal scrolling.

# Condensing wildcard suffixes in use statements
condense_wildcard_suffixes = true
# Justification: Simplifies wildcard imports (e.g., `use foo::{bar, baz}` instead of `use foo::bar; use foo::baz;`).
# This reduces the number of import lines, improving readability and aligning with the style guide's
# requirement for organized and concise `use` declarations.

# Enforcing error on line length overflow
error_on_line_overflow = true
# Justification: Causes rustfmt to fail if lines exceed the maximum width (default or configured).
# This enforces strict adherence to formatting rules, ensuring consistency across the codebase and
# preventing overly long lines that harm readability, in line with the style guide's readability goals.

# Formatting code blocks within documentation comments
format_code_in_doc_comments = true
# Justification: Automatically formats code snippets in documentation comments (///) to match the project's
# style. This ensures that examples in public documentation are consistent with the codebase, supporting
# the style guide's mandate for comprehensive and clear documentation.

# Formatting macro bodies
format_macro_bodies = true
# Justification: Ensures that macro definitions are formatted consistently, improving readability of complex
# macro code. This supports the style guide's goal of reducing cognitive overhead in collaborative projects.

# Formatting macro matchers
format_macro_matchers = true
# Justification: Ensures consistent formatting within macro matchers, making macro logic easier to follow.
# This aligns with the style guide's emphasis on readable and maintainable code, especially for complex macros.

# Formatting string literals
format_strings = true
# Justification: Ensures consistent formatting of string literals, such as wrapping long strings. This
# improves readability of string-heavy code, supporting the style guide's focus on clear and maintainable code.

# Grouping imports by type
group_imports = "StdExternalCrate"
# Justification: Groups imports in the order: standard library (`std`), external crates, then local modules.
# This directly supports the style guide's requirement for `use` declarations to be grouped in this order,
# ensuring predictable and organized import sections that are easier to navigate in large projects.

# Setting import granularity to module level
imports_granularity = "Module"
# Justification: Groups imports by module (e.g., `use std::collections::{HashMap, Vec}`) rather than individual
# items. This reduces visual clutter and aligns with the style guide's requirement for organized `use`
# declarations, making imports easier to read and maintain.

# Adding trailing commas in match blocks
match_block_trailing_comma = true
# Justification: Adds trailing commas in match expressions, ensuring consistency and making it easier to
# add new arms without reformatting existing ones. This supports the style guide's goal of maintainability
# in collaborative open-source projects.

# Normalizing documentation attributes
normalize_doc_attributes = true
# Justification: Ensures consistent formatting of documentation attributes (e.g., `#[doc = "..."]` to `///`).
# This supports the style guide's requirement for clear and comprehensive documentation comments by
# ensuring they are formatted in a standard way, improving readability.

# Reordering implementation items
reorder_impl_items = true
# Justification: Reorders methods within `impl` blocks to follow a consistent order (e.g., trait methods
# before inherent methods). While not directly addressing the style guide's top-level item ordering, it
# improves consistency within `impl` blocks, supporting the guide's emphasis on predictability and
# ease of navigation.

# Setting the style edition for formatting
style_edition = "2024"
# Justification: Aligns rustfmt with the 2024 style edition, which includes modern formatting conventions
# and import sorting improvements. This ensures the project stays up-to-date with Rust's evolving style
# standards, supporting the style guide's goal of a consistent and modern codebase.

# Wrapping comments to fit within the comment width
wrap_comments = true
# Justification: Automatically wraps comments to fit within the `comment_width` (100 characters), ensuring
# they remain readable without manual intervention. This supports the style guide's requirement for clear
# and concise documentation comments that are easy to read in code reviews and documentation.
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
