[← Back to Learn](/learn/)

# Package Management

Zig uses a decentralized, content-based package manager. No central registry exists.

## Content-based vs Version-based

Traditional package managers:

```
"lodash": "^4.17.21"  # Might get 4.17.22 tomorrow
"react": "~18.2.0"    # Could be any 18.2.x
```

Zig package manager:

```zig
.lodash = .{
    .url = "...",
    .hash = "1220abc...",  # Always gets EXACTLY this code
},
```

In Zig, the hash IS the version. Change the code = change the hash = different package.

## Content-based addressing

In Zig, packages are identified by their content hash, not by version numbers or URLs:

- **Hash = Identity**: The SHA-256 hash of the package contents is its unique identifier
- **URLs are mirrors**: The same package (same hash) can be downloaded from multiple URLs
- **Immutable packages**: If content at a URL changes, the hash won’t match and Zig will reject it
- **Reproducible builds**: The same hash always produces the exact same code

Example: If a package author accidentally updates their “v1.0.0” release:

```
Your build.zig.zon: hash = "1220abc..."  (original v1.0.0)
Updated tarball:    hash = "1220def..."  (modified v1.0.0)
Result: Build fails! You're protected from surprise changes.
```

Benefits:

- **Security**: Tampering is automatically detected
- **Caching**: Packages with the same hash are safely shared across projects
- **No version conflicts**: Different versions have different hashes
- **Decentralized**: No central authority needed to manage packages

## Adding dependencies

Use `zig fetch` to add a dependency:

```bash
$ zig fetch --save https://github.com/user/package/archive/refs/tags/v1.0.0.tar.gz
```

This downloads the package and updates your `build.zig.zon` file.

**Important**: The hash that gets added identifies this exact code forever. Even if the maintainer updates the v1.0.0 tag (bad practice!), your build will fail rather than silently use different code.

## build.zig.zon

Every project with dependencies needs a `build.zig.zon` file:

```zig
.{
    .name = "my-project",
    .version = "0.1.0",
    .minimum_zig_version = "0.14.0",
    .fingerprint = "12345...", // Auto-generated, don't modify
    
    .dependencies = .{
        .zap = .{
            .url = "https://github.com/zigzap/zap/archive/v0.1.7-pre.tar.gz",
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

## Using dependencies

In `build.zig`:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    const zap = b.dependency("zap", .{
        .target = target,
        .optimize = optimize,
    });
    
    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    exe.root_module.addImport("zap", zap.module("zap"));
    
    b.installArtifact(exe);
}
```

In source code:

```zig
const std = @import("std");
const zap = @import("zap");

pub fn main() !void {
    // Use zap here
}
```

## Hash verification

When first adding a dependency, you’ll see:

```bash
$ zig build
error: hash mismatch: manifest declares
122053da05e0c9348d91218ef015c8307749ef39f8e90c208a186e5f444e818672da
but the fetched package has
122036b1948caa15c2c9054286b3057877f7b152a5102c9262511bf89554dc836ee5
```

This happens because:

1. You provided a dummy/incorrect hash in `build.zig.zon`
1. Zig fetched the package and computed its actual content hash
1. The hashes didn’t match (this is expected on first add)

Update `build.zig.zon` with the correct hash. This hash now permanently identifies this exact version of the code.

## zig fetch options

```bash
$ zig fetch [options] <url>
$ zig fetch [options] <path>
```

Options:

- `--save` - Add to build.zig.zon
- `--save=[name]` - Add with specific name
- `--save-exact` - Store URL without normalization
- `--global-cache-dir [path]` - Override cache location
- `--debug-hash` - Print hash information

### Sources

- **GitHub releases**: `https://github.com/user/repo/archive/refs/tags/v1.0.0.tar.gz`
- **Git repositories**: `git+https://github.com/user/repo#commit-hash`
- **Local paths**: `../path/to/local/package` or `file:///absolute/path`
- **Branch snapshots**: `https://github.com/user/repo/archive/refs/heads/main.tar.gz`

## Dependency types

### Remote dependencies

```zig
.dependencies = .{
    .json = .{
        .url = "https://github.com/user/json-lib/archive/v1.0.0.tar.gz",
        .hash = "1220abc...",  // This hash IS the package - URL is just where to get it
    },
}
```

### Local dependencies

```zig
.dependencies = .{
    .local_lib = .{
        .path = "./libs/local-lib",  // No hash - you control the content
    },
}
```

### Lazy dependencies

```zig
.dependencies = .{
    .optional_lib = .{
        .url = "https://github.com/user/optional/archive/main.tar.gz",
        .hash = "1220abc...",
        .lazy = true,  // Only fetch if actually used
    },
}
```

Even lazy dependencies are verified by hash when fetched.

## Common workflows

### Updating dependencies

```bash
# Fetch new version
$ zig fetch --save https://github.com/user/repo/archive/refs/tags/v2.0.0.tar.gz

# Clear cache if needed
$ rm -rf ~/.cache/zig/*

# Rebuild
$ zig build
```

**Note**: You’re not just updating a version number - you’re switching to completely different content with a different hash. The old version remains cached and available to any project still using its hash.

### Local development

Use local paths during development:

```zig
.dependencies = .{
    .my_lib = .{
        // .url = "https://github.com/user/repo/archive/v1.0.0.tar.gz",
        // .hash = "1220...",  // Hash for released version
        .path = "../my-local-library",  // Direct path for development
    },
},
```

Switch between stable (hash-verified) and development (local) versions by commenting/uncommenting.

### Creating packages

In `build.zig`:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    _ = b.addModule("my_library", .{
        .root_source_file = b.path("src/main.zig"),
    });
}
```

## Cache

Packages are stored in:

- Linux/macOS: `~/.cache/zig/`
- Windows: `%LOCALAPPDATA%\zig\cache\`

Content-based storage:

```
~/.cache/zig/p/
├── 1220abc.../  # One project uses "json v1.0"
├── 1220def.../  # Another uses "json v2.0"
└── 1220ghi.../  # Both use "http v1.5" (same hash = shared)
```

Each directory name is the content hash. Multiple projects using the same exact dependency share it automatically.

Clear cache:

```bash
$ rm -rf ~/.cache/zig/*
```

Pre-fetch dependencies:

```bash
$ zig build --fetch
```

## Troubleshooting

### Hash mismatch

Update the hash in build.zig.zon with the correct value from the error message.

**Why this happens**: Zig’s content-based system ensures you get exactly the code you expect. If the package author updates the release tarball (even with the same version tag), you’ll get a hash mismatch - protecting you from unexpected changes.

### Network issues

Check internet connection. For restricted environments, use git submodules:

```bash
$ git submodule add https://github.com/user/repo deps/repo
```

Then reference locally:

```zig
.dependencies = .{
    .repo = .{
        .path = "deps/repo",
    },
},
```

### Missing dependencies

Ensure dependency names match between build.zig and build.zig.zon.

### Debug issues

```bash
$ zig build --verbose
```

## Advanced usage

### Platform-specific dependencies

```zig
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

### Composite packages

```zig
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

## build.zig.zon reference

### Required fields

- `.name` - Package name (valid Zig identifier, max 32 bytes)
- `.paths` - Files and directories to include

### Optional fields

- `.version` - Version string (advisory only)
- `.minimum_zig_version` - Minimum compatible Zig version
- `.dependencies` - Package dependencies
- `.fingerprint` - Auto-generated unique identifier (don’t modify)

### Hash format

- Multihash format: `1220` prefix (SHA256) + 64 hex digits
- Computed from file contents after applying `.paths` inclusion rules
- Files sorted by path before hashing
- **Hash is the package identity** - URLs just tell where to find it
- Same hash = same exact code, always
