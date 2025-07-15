[← Back to Learn](/learn/)

# Zig Package Management Guide

A comprehensive guide to understanding and using Zig’s decentralized, content-addressed package manager.

## Table of Contents

- [Introduction](#introduction)
- [Core Concepts](#core-concepts)
- [Getting Started](#getting-started)
- [Common Workflows](#common-workflows)
- [Advanced Usage](#advanced-usage)
- [Reference](#reference)

## Introduction

### What Makes Zig’s Package Manager Different?

Most package managers rely on central registries and version numbers to manage dependencies. Zig takes a fundamentally different approach: it’s **decentralized** and **content-addressed**.

**Traditional approach:**

```json
{
  "lodash": "^4.17.21",  // Might resolve to 4.17.22 tomorrow
  "react": "~18.2.0"     // Could be any 18.2.x version
}
```

**Zig’s approach:**

```zig
.dependencies = .{
    .lodash = .{
        .url = "https://example.com/lodash-4.17.21.tar.gz",
        .hash = "1220abc123...", // Always gets EXACTLY this code
    },
}
```

### Key Benefits

**Reproducible Builds**: The same `build.zig.zon` file always resolves to identical source code across all environments and developers.

**Enhanced Security**: Every package download is verified against its cryptographic hash, preventing tampering or unexpected changes.

**No Single Point of Failure**: No central registry means packages can be hosted anywhere. The hash guarantees integrity regardless of the source.

**Automatic Caching**: Packages are cached by their content hash, so multiple projects automatically share identical dependencies.

## Core Concepts

### Content-Addressed Packages

In Zig, **the content hash IS the version**. When code changes, the hash changes, making it effectively a different package. This eliminates the ambiguity of semantic versioning ranges.

```zig
// These are completely different packages to Zig
.my_lib_v1 = .{ .hash = "1220abc..." },  // Original version
.my_lib_v2 = .{ .hash = "1220def..." },  // Updated version
```

### The Package Manifest

Every Zig project with dependencies has a `build.zig.zon` file that serves as the package manifest. This file defines:

- Project metadata (name, version)
- Which files belong to the package
- External dependencies

### Dependencies vs Modules

- **Dependency**: An external package listed in `build.zig.zon`
- **Module**: A Zig code unit that can be imported with `@import()`
- A dependency can expose one or more modules to your code

### The Global Cache

Zig maintains a global cache of all downloaded packages:

- **Linux/macOS**: `~/.cache/zig/`
- **Windows**: `%LOCALAPPDATA%\zig\cache\`

Packages are stored by their hash, enabling automatic sharing across projects.

## Getting Started

### Your First Dependency

Let’s add a real dependency to understand the workflow. We’ll use the popular Zig Zap web framework.

**Step 1: Add the dependency**

```bash
zig fetch --save https://github.com/zigzap/zap/archive/v0.1.7-pre.tar.gz
```

This command:

1. Downloads the package from the URL
1. Computes its SHA-256 hash
1. Creates or updates `build.zig.zon` with the dependency entry

**Step 2: Examine the generated manifest**

After running `zig fetch`, your `build.zig.zon` will look like this:

```zig
.{
    .name = "my-project",
    .version = "0.1.0",
    .paths = .{
        "build.zig",
        "build.zig.zon", 
        "src",
    },
    .dependencies = .{
        .zap = .{
            .url = "https://github.com/zigzap/zap/archive/v0.1.7-pre.tar.gz",
            .hash = "122036b1948caa15c2c9054286b3057877f7b152a5102c9262511bf89554dc836ee5",
        },
    },
}
```

**Step 3: Wire up the dependency in your build script**

Edit your `build.zig` to make the dependency available to your code:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // Get the dependency and add its module to our executable
    const zap_dep = b.dependency("zap", .{
        .target = target,
        .optimize = optimize,
    });
    exe.root_module.addImport("zap", zap_dep.module("zap"));

    b.installArtifact(exe);
}
```

**Step 4: Use the dependency in your code**

Now you can import and use Zap in your Zig source files:

```zig
const std = @import("std");
const zap = @import("zap");

pub fn main() !void {
    std.log.info("Using Zap framework!", .{});
    // Use Zap functionality here
}
```

### Understanding the Manifest Structure

Let’s break down each part of `build.zig.zon`:

```zig
.{
    // Project identification
    .name = "my-project",        // Your package name
    .version = "0.1.0",          // Informational only

    // Package contents (what gets hashed)
    .paths = .{
        "build.zig",             // Build script
        "build.zig.zon",         // This manifest
        "src",                   // Source directory
        "README.md",             // Documentation
        // Add other files/directories as needed
    },

    // External dependencies
    .dependencies = .{
        .dependency_name = .{
            .url = "https://...",    // Where to download
            .hash = "1220...",       // Content verification
        },
    },
}
```

**Important**: The `.paths` field determines what gets included when another project depends on yours. Only list files that are part of your package’s public interface.

## Common Workflows

### Adding Dependencies

**Method 1: Using `zig fetch` (Recommended)**

```bash
# Automatically downloads, hashes, and updates build.zig.zon
zig fetch --save https://github.com/user/package/archive/v1.0.0.tar.gz
```

**Method 2: Manual entry**

```zig
// Add to build.zig.zon dependencies
.my_package = .{
    .url = "https://github.com/user/package/archive/v1.0.0.tar.gz",
    .hash = "", // Leave empty initially
},
```

Then run `zig build` to get the correct hash from the error message.

### Updating Dependencies

To update to a newer version:

```bash
# Update existing dependency to new version
zig fetch --save=package_name https://github.com/user/package/archive/v2.0.0.tar.gz
```

This updates both the URL and hash for the named dependency.

### Local Development

When developing a library alongside an application that uses it, use path dependencies:

```zig
.dependencies = .{
    .my_local_lib = .{
        .path = "../my-library-project",
        // No .hash needed - changes are reflected immediately
    },
}
```

Path dependencies are perfect for:

- Developing multiple related packages
- Testing unreleased changes
- Monorepo setups

### Version Management Strategy

Since Zig uses content hashes instead of semantic versions:

**For Library Authors:**

- Use clear, descriptive Git tags
- Provide stable URLs for releases
- Document breaking changes in release notes

**For Library Users:**

- Pin to specific release URLs
- Use path dependencies during active development
- Test thoroughly before updating hashes

### Troubleshooting Hash Mismatches

If you see a hash mismatch error:

```
error: hash mismatch: manifest declares
1220aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
but the fetched package has  
122036b1948caa15c2c9054286b3057877f7b152a5102c9262511bf89554dc836ee5
```

This is a **security feature**, not a bug. It means:

1. The content at the URL has changed
1. Someone may have updated the package
1. The URL might point to different content

**To fix:** Copy the correct hash from the error message into your `build.zig.zon`.

## Advanced Usage

### Creating Your Own Package

Any Zig project can be a package. The key is exposing modules in your `build.zig`:

```zig
// In your library's build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // Standard target and optimization setup
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // Create a module that other projects can import
    const my_lib_module = b.addModule("my_library", .{
        .root_source_file = b.path("src/lib.zig"),
        .target = target,
        .optimize = optimize,
    });

    // Optional: create an executable for testing
    const exe = b.addExecutable(.{
        .name = "my_library_test",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    exe.root_module.addImport("my_library", my_lib_module);

    b.installArtifact(exe);
}
```

**Package Layout:**

```
my-library/
├── build.zig          # Build script with module definition
├── build.zig.zon      # Package manifest
├── src/
│   ├── lib.zig        # Main library interface
│   └── main.zig       # Optional test executable
└── README.md
```

### Lazy Dependencies

Mark dependencies as lazy to avoid downloading them unless actually used:

```zig
.dependencies = .{
    .optional_feature = .{
        .url = "https://...",
        .hash = "1220...",
        .lazy = true,  // Only downloaded if b.dependency() is called
    },
}
```

Use lazy dependencies for:

- Platform-specific code
- Optional features
- Development-only tools

### Composite Packages

Create packages that re-export functionality from multiple dependencies:

```zig
// In a "web_toolkit" package's build.zig
pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // Get sub-dependencies
    const json_dep = b.dependency("json_parser", .{
        .target = target,
        .optimize = optimize,
    });
    const http_dep = b.dependency("http_client", .{
        .target = target,
        .optimize = optimize,
    });

    // Create composite module
    _ = b.addModule("web_toolkit", .{
        .root_source_file = b.path("src/toolkit.zig"),
        .dependencies = &.{
            .{ .name = "json", .module = json_dep.module("json_parser") },
            .{ .name = "http", .module = http_dep.module("http_client") },
        },
    });
}
```

Users of `web_toolkit` can then access both JSON and HTTP functionality through a single import.

### Dependency Resolution and Conflicts

Zig handles dependency conflicts gracefully:

**Diamond Dependencies:**

```
Your Project
├── Library A (uses JSON v1.0)
└── Library B (uses JSON v2.0)
```

Both JSON versions coexist in the cache. Each library gets exactly the version it expects.

**Duplicate Dependencies:**
Multiple projects using the same hash automatically share the cached package.

## Reference

### Command Reference

**Package Management Commands:**

```bash
# Add new dependency
zig fetch --save <url>

# Update existing dependency  
zig fetch --save=<name> <new_url>

# Download all dependencies without building
zig build --fetch

# Clean and rebuild
zig build clean
zig build
```

**Cache Management:**

```bash
# View cache location
zig env

# Clear entire cache (safe - will re-download as needed)
rm -rf ~/.cache/zig/  # Linux/macOS
rmdir /s %LOCALAPPDATA%\zig\cache\  # Windows
```

### build.zig.zon Schema

```zig
.{
    // Required fields
    .name = "string",           // Package name
    .version = "string",        // Semantic version (informational)
    
    // Optional fields
    .paths = .{                 // Files/dirs included in package hash
        "string",               // File or directory path
        // ...
    },
    
    .dependencies = .{
        .dep_name = .{
            // Remote dependency
            .url = "string",     // Download URL
            .hash = "string",    // SHA-256 hash (required)
            .lazy = bool,        // Optional: lazy loading
        },
        .local_dep = .{
            // Local dependency
            .path = "string",    // Relative path to local package
        },
    },
}
```

### Cache Structure

```
~/.cache/zig/
├── p/                      # Packages
│   ├── 1220abc123.../      # Package by hash
│   ├── 1220def456.../      # Another package
│   └── ...
├── h/                      # HTTP cache
├── tmp/                    # Temporary files
└── ...
```

Each package directory contains the extracted contents of the downloaded archive, exactly as specified by the hash.

### Best Practices

**For Library Authors:**

- Keep `build.zig.zon` minimal and focused
- Use clear module names in your `build.zig`
- Provide stable release URLs
- Document your public API thoroughly
- Test with multiple Zig versions when possible

**For Library Users:**

- Pin to specific release URLs rather than branch URLs
- Use descriptive dependency names in your manifest
- Keep dependencies up to date with security patches
- Use path dependencies during active development
- Document your dependency choices

**For Teams:**

- Commit `build.zig.zon` to version control
- Never commit the cache directory
- Use consistent dependency naming across projects
- Establish update policies for dependencies
- Consider using a local mirror for critical dependencies
