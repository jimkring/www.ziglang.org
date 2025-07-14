[← Back to Learn](/learn/)

# Package Management

Zig’s package manager provides a decentralized approach to dependency management that prioritizes reproducibility and security through content-based addressing.  This guide covers the complete workflow from adding dependencies to troubleshooting common issues.

## Overview

The Zig package manager uses hash-based integrity verification to ensure reproducible builds. Unlike traditional package managers, Zig has no central registry - packages are fetched directly from URLs and verified using SHA-256 hashes. 

Key components:

- **build.zig.zon** - Package manifest file using Zig Object Notation  
- **zig fetch** - Command-line tool for adding dependencies 
- **Global cache** - Shared package storage across projects 

## Getting started with dependencies

### Adding your first dependency

The simplest way to add a dependency is using `zig fetch`: 

```bash
$ zig fetch --save https://github.com/user/package/archive/refs/tags/v1.0.0.tar.gz
```

This command downloads the package, calculates its hash, and adds it to your `build.zig.zon` file. 

### Creating build.zig.zon

Every Zig project with dependencies needs a `build.zig.zon` file: 

```zig
.{
    .name = "my-project",
    .version = "0.1.0",
    .minimum_zig_version = "0.14.0",
    
    .dependencies = .{
        .my_dependency = .{
            .url = "https://github.com/user/repo/archive/refs/tags/v1.0.0.tar.gz",
            .hash = "1220abc...", // Use dummy hash first, then update
        },
    },
    
    .paths = .{
        "build.zig",
        "build.zig.zon",
        "src",
    },
}
```

### Integrating dependencies in build.zig

Once a dependency is declared, integrate it in your build script: 

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    // Load the dependency
    const my_dep = b.dependency("my_dependency", .{
        .target = target,
        .optimize = optimize,
    });
    
    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    // Add as module import
    exe.root_module.addImport("my_dependency", my_dep.module("my_dependency"));
    
    b.installArtifact(exe);
}
```

### Using dependencies in source code

Import and use your dependencies like any other module: 

```zig
const std = @import("std");
const my_dep = @import("my_dependency");

pub fn main() !void {
    const result = my_dep.doSomething();
    std.debug.print("Result: {}\n", .{result});
}
```

## The zig fetch command

### Basic usage

```bash
$ zig fetch [options] <url>
$ zig fetch [options] <path>
```

### Available options

- `--save` - Add the fetched package to build.zig.zon
- `--save=[name]` - Add with a specific dependency name
- `--save-exact` - Store the URL verbatim without normalization
- `--global-cache-dir [path]` - Override the global cache location
- `--debug-hash` - Print verbose hash information

### Supported sources

#### GitHub releases

```bash
$ zig fetch --save https://github.com/user/repo/archive/refs/tags/v1.0.0.tar.gz
```

#### Git repositories

```bash
$ zig fetch --save git+https://github.com/user/repo#commit-hash
$ zig fetch --save git+https://github.com/user/repo/#HEAD
```

#### Local paths

```bash
$ zig fetch --save ../path/to/local/package
$ zig fetch --save file:///absolute/path/to/package
```

#### Branch snapshots

```bash
$ zig fetch --save https://github.com/user/repo/archive/refs/heads/main.tar.gz
```

 

### Hash verification

When you first add a dependency, you’ll see a hash mismatch error: 

```bash
$ zig build
error: hash mismatch: manifest declares
122053da05e0c9348d91218ef015c8307749ef39f8e90c208a186e5f444e818672da
but the fetched package has
122036b1948caa15c2c9054286b3057877f7b152a5102c9262511bf89554dc836ee5
```

Copy the correct hash from the error message and update your `build.zig.zon` file. 

## Understanding build.zig.zon

### File format

The build.zig.zon file uses Zig Object Notation (ZON), which supports: 

- Comments with `//`
- Trailing commas
- Field names prefixed with `.`
- Zig-style identifiers 

### Essential fields

```zig
.{
    .name = "package-name",              // Required: valid Zig identifier
    .version = "1.0.0",                  // Optional: currently advisory only
    .minimum_zig_version = "0.14.0",     // Optional: compatibility requirement
    
    .paths = .{                          // Required: included files/directories
        "src",
        "build.zig",
        "build.zig.zon",
        "README.md",
    },
    
    .dependencies = .{                   // Optional: package dependencies
        // dependency specifications
    },
}
```

### Dependency specifications

#### Remote dependencies

```zig
.dependencies = .{
    .json = .{
        .url = "https://github.com/user/json-lib/archive/v1.0.0.tar.gz",
        .hash = "1220abc...",
    },
}
```

#### Local dependencies

```zig
.dependencies = .{
    .local_lib = .{
        .path = "./libs/local-lib",  // No hash needed for local paths
    },
}
```

#### Lazy dependencies

```zig
.dependencies = .{
    .optional_lib = .{
        .url = "https://github.com/user/optional/archive/main.tar.gz",
        .hash = "1220abc...",
        .lazy = true,  // Only fetch if actually used
    },
}
```

### Hash format

Zig uses multihash format for package integrity:

- Prefix `1220` indicates SHA256 (0x12) with 32-byte digest (0x20)
- Hash is computed from package contents after applying `.paths` inclusion rules 
- The hash in build.zig.zon serves as the source of truth - URLs are just mirrors 

## Complete dependency workflows

### Adding dependencies step-by-step

1. **Find the package source URL**
- GitHub releases provide `.tar.gz` archives 
- Use tagged releases for stability
- Check the package’s documentation for recommended URLs
1. **Add to build.zig.zon**
   
   ```bash
   $ zig fetch --save https://github.com/zigzap/zap/archive/refs/tags/v0.1.7-pre.tar.gz
   ```
1. **Update build.zig**
   
   ```zig
   const zap = b.dependency("zap", .{
       .target = target,
       .optimize = optimize,
   });
   exe.root_module.addImport("zap", zap.module("zap"));
   ```
1. **Import in source code**
   
   ```zig
   const zap = @import("zap");
   ```

### Updating dependencies

To update a dependency to a new version:

```bash
# Update to new version
$ zig fetch --save https://github.com/user/repo/archive/refs/tags/v2.0.0.tar.gz

# Clear cache if needed
$ rm -rf ~/.cache/zig/*

# Rebuild
$ zig build
```

### Removing dependencies

1. Remove from `build.zig.zon`:
   
   ```zig
   .dependencies = .{
       // Remove the dependency entry
   },
   ```
1. Remove from `build.zig`:
   
   ```zig
   // Remove dependency loading and imports
   ```
1. Remove from source files:
   
   ```zig
   // Remove @import statements
   ```

### Working with transitive dependencies

Zig automatically handles transitive dependencies.   If package A depends on package B: 

```zig
// Package A's build.zig.zon
.dependencies = .{
    .package_b = .{
        .url = "https://github.com/user/package-b/archive/v1.0.0.tar.gz",
        .hash = "1220...",
    },
},

// Your project only needs to declare package A
.dependencies = .{
    .package_a = .{
        .url = "https://github.com/user/package-a/archive/v1.0.0.tar.gz",
        .hash = "1220...",
    },
},
```

## Local development

### Using local paths during development

For active development, use local paths instead of URLs: 

```zig
.dependencies = .{
    .my_lib = .{
        .path = "../my-local-library",
    },
},
```

### Development override pattern

Switch between local and remote sources:

```zig
.dependencies = .{
    .my_dep = .{
        // Comment out for local development
        //.url = "https://github.com/user/repo/archive/v1.0.0.tar.gz",
        //.hash = "1220...",
        .path = "../my-dependency-fork",
    },
},
```

### Creating packages for others

To make your package available to others:  

```zig
// build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    _ = b.addModule("my_library", .{
        .root_source_file = b.path("src/main.zig"),
    });
}
```

Ensure your `build.zig.zon` includes all necessary files: 

```zig
.{
    .name = "my_library",
    .version = "1.0.0",
    .paths = .{
        "src",
        "build.zig",
        "build.zig.zon",
        "LICENSE",
        "README.md",
    },
}
```

## Cache management

### Understanding the cache

Packages are stored in a global cache: 

- Linux/macOS: `~/.cache/zig/`
- Windows: `%LOCALAPPDATA%\zig\cache\` 

Structure:

```
~/.cache/zig/
└── p/
    ├── 1220abc.../  # Package contents indexed by hash
    ├── 1220def.../
    └── ...
```

 

### Cache operations

```bash
# Clear entire cache [![Zig Package Manager - WTF is Zon](claude-citation:/icon.png?validation=E11EEACD-DADF-4EAC-981C-097EA6BB49AF&citation=eyJlbmRJbmRleCI6ODg3NywibWV0YWRhdGEiOnsiaWNvblVybCI6Imh0dHBzOlwvXC93d3cuZ29vZ2xlLmNvbVwvczJcL2Zhdmljb25zP3N6PTY0JmRvbWFpbj16aWcubmV3cyIsInByZXZpZXdUaXRsZSI6IlppZyBQYWNrYWdlIE1hbmFnZXIgLSBXVEYgaXMgWm9uIiwic291cmNlIjoiemlnIiwidHlwZSI6ImdlbmVyaWNfbWV0YWRhdGEifSwic291cmNlcyI6W3siaWNvblVybCI6Imh0dHBzOlwvXC93d3cuZ29vZ2xlLmNvbVwvczJcL2Zhdmljb25zP3N6PTY0JmRvbWFpbj16aWcubmV3cyIsInNvdXJjZSI6InppZyIsInRpdGxlIjoiWmlnIFBhY2thZ2UgTWFuYWdlciAtIFdURiBpcyBab24iLCJ1cmwiOiJodHRwczpcL1wvemlnLm5ld3NcL2VkeXVcL3ppZy1wYWNrYWdlLW1hbmFnZXItd3RmLWlzLXpvbi01NThlIn1dLCJzdGFydEluZGV4Ijo4ODU5LCJ0aXRsZSI6InppZyIsInVybCI6Imh0dHBzOlwvXC96aWcubmV3c1wvZWR5dVwvemlnLXBhY2thZ2UtbWFuYWdlci13dGYtaXMtem9uLTU1OGUiLCJ1dWlkIjoiOTI0MmFhYTUtY2VhZi00NGI1LWEzNGYtMGU0NWU5MmU4ZmFkIn0%3D "Zig Package Manager - WTF is Zon")](https://zig.news/edyu/zig-package-manager-wtf-is-zon-558e)
$ rm -rf ~/.cache/zig/*

# Clear specific package
$ rm -rf ~/.cache/zig/p/1220abc*

# Pre-fetch all dependencies
$ zig build --fetch

# Use custom cache location
$ zig fetch --global-cache-dir /custom/path --save <url>
```

 

## Best practices

### Version management

- **Pin specific versions** using tagged releases
- **Record exact commits** for pre-release dependencies
- **Test updates thoroughly** before committing hash changes
- **Document version requirements** in your README

### Security considerations

- **Always verify hashes** match expected values
- **Use HTTPS URLs** for package sources
- **Review dependencies** before adding them
- **Monitor for security updates** in your dependencies

### Performance optimization

- **Pre-populate cache** in CI environments using `zig build --fetch`
- **Share cache** across CI runs when possible
- **Use specific releases** instead of branch snapshots
- **Minimize dependency count** to reduce download time

### Build reproducibility

Ensure reproducible builds across environments:  

```bash
# Specify exact target
$ zig build -Dtarget=x86_64-linux-gnu

# Use baseline CPU features
$ zig build -Dcpu=baseline

# Set optimization level
$ zig build -Doptimize=ReleaseFast
```

## Troubleshooting

### Common errors

#### Hash mismatch

```
error: hash mismatch: manifest declares 1220abc... but the fetched package has 1220def...
```

**Solution**: Update the hash in build.zig.zon with the provided correct hash. 

#### Network issues

```
error: unable to connect to server: ConnectionTimedOut
```

**Solution**: Check internet connection and proxy settings. Consider using git submodules for restricted environments. 

#### Cache corruption

```
error: unable to unpack tarball: EndOfStream
```

**Solution**: Clear the cache and try again:  

```bash
$ rm -rf ~/.cache/zig/*
$ zig build
```

#### Missing dependencies

```
error: no dependency named 'missing_dep' in the build.zig.zon manifest
```


**Solution**: Ensure the dependency name in build.zig matches the name in build.zig.zon.

### Proxy configuration

For environments behind proxies: 

```bash
# Set proxy environment variables
$ export HTTP_PROXY=http://proxy.example.com:8080
$ export HTTPS_PROXY=http://proxy.example.com:8080

# Or use git submodules as workaround
$ git submodule add https://github.com/user/repo deps/repo
```

Then reference as local path: 

```zig
.dependencies = .{
    .repo = .{
        .path = "deps/repo",
    },
},
```

### Debugging dependency resolution

Use verbose output to debug issues: 

```bash
$ zig build --verbose
```

This shows:

- Dependency resolution steps
- Cache lookups
- Download progress
- Hash calculations 

## Advanced topics

### Multi-platform dependencies

Handle platform-specific dependencies:

```zig
// build.zig
const target_os = target.result.os.tag;
const platform_dep = switch (target_os) {
    .windows => b.dependency("windows_lib", .{}),
    .linux => b.dependency("linux_lib", .{}),
    else => null,
};

if (platform_dep) |dep| {
    exe.linkLibrary(dep.artifact("platform_lib"));
}
```

### Creating composite packages

Combine multiple packages:

```zig
// build.zig
pub fn build(b: *std.Build) void {
    const json = b.dependency("json", .{});
    const http = b.dependency("http", .{});
    
    _ = b.addModule("web_toolkit", .{
        .root_source_file = b.path("src/main.zig"),
        .dependencies = &.{
            .{ .name = "json", .module = json.module("json") },
            .{ .name = "http", .module = http.module("http") },
        },
    });
}
```

### Migration from other systems

When migrating from git submodules: 

1. Identify current dependencies
1. Find or create package URLs
1. Add to build.zig.zon using `zig fetch --save`
1. Update build.zig to use `b.dependency()`
1. Remove git submodules
1. Test thoroughly

## Future developments

The Zig package manager continues to evolve.  Planned improvements include:

- SSH support for private repositories 
- Enhanced proxy configuration
- Signature verification for packages
- Improved error messages
- Performance optimizations 

Current limitations to be aware of:

- No central package registry 
- Manual hash management required
- Limited proxy support 
- No automatic version resolution

The package manager prioritizes security and reproducibility over convenience features found in other ecosystems. This design choice ensures builds remain deterministic across different environments and time. 
