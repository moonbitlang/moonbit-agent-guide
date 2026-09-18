## Conditional Compilation

Target specific backends/modes in `moon.pkg`. Legacy `moon.pkg.json` files remain supported; use the current format for new examples.

The configuration examples below are independent. When combining them in one package, merge their fields into a single `options(...)` block.

```moonbit
options(
  targets: {
    "wasm_only.mbt": ["wasm"],
    "js_only.mbt": ["js"],
    "debug_only.mbt": ["debug"],
    "wasm_or_js.mbt": ["wasm", "js"], // for wasm or js backend
    "not_js.mbt": ["not", "js"], // for nonjs backend
    "complex.mbt": ["or", ["and", "wasm", "release"], ["and", "js", "debug"]] // more complex conditions
  }
)
```

**Available conditions:**

- **Backends**: `"wasm"`, `"wasm-gc"`, `"js"`, `"native"`
- **Build modes**: `"debug"`, `"release"`
- **Logical operators**: `"and"`, `"or"`, `"not"`

## Link Configuration

### Basic Linking

Enable linking with `options(link: true)`, or configure individual backends:

```moonbit
options(
  link: {
    "wasm": {
      "exports": ["hello", "foo:bar"], // Export functions
      "heap-start-address": 1024, // Memory layout
      "import-memory": {
        // Import external memory
        "module": "env",
        "name": "memory"
      },
      "export-memory-name": "memory" // Export memory with name
    },
    "wasm-gc": {
      "exports": ["hello"],
      "use-js-builtin-string": true, // JS String Builtin support
      "imported-string-constants": "_" // String namespace
    },
    "js": {
      "exports": ["hello"],
      "format": "esm" // "esm", "cjs", or "iife"
    },
    "native": {
      "cc": "gcc", // C compiler
      "cc-flags": "-O2 -DMOONBIT", // Compile flags
      "cc-link-flags": "-s" // Link flags
    }
  }
)
```

## Warning Control

Disable specific warnings in `moon.pkg`:

```moonbit
warnings = "-unused_value-unused_package"
```

**Common warning names:**

- `unused_value` - Unused function or variable
- `partial_match` - Partial pattern matching
- `unreachable_code` - Unreachable code
- `unused_package` - Unused package

Use `moon explain --diagnostic` to list warnings and `moon explain --diagnostic <name>` for details. Use `--deny-warn` on validation commands to make enabled warnings fail the build. Legacy JSON configurations use `"warn-list"` instead of `warnings`.

## Pre-build Commands

Declare a reusable rule and apply it with `dev_build` in `moon.pkg` to embed external files as MoonBit code:

```moonbit
rule(
  name: "embed-text",
  command: ":embed -i $input -o $output --name data --text",
)
dev_build(rule: "embed-text", input: "data.txt", output: "embedded.mbt")
```

Paths are relative to the module root. These commands run during package development, not when downstream users build the package as a dependency. Commit generated outputs for downstream builds. The old JSON `"pre-build"` configuration is deprecated.

Generated code example:

```mbt check
///|
let data : String =
  #|hello,
  #|world
  #|
```
