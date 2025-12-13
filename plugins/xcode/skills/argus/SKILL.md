---
name: argus-build-diagnostics
description: Diagnose xcodebuild builds using argus. Use this instead of parsing xcodebuild output to avoid filling the context window. Query build errors, warnings, slowest targets, bottlenecks, implicit/redundant dependencies, and dependency graph with linking information for optimization analysis.
---

# Argus Build Diagnostics

Use argus to analyze Xcode builds instead of parsing raw xcodebuild output.

## Installation

```bash
mise install github:tuist/argus@0.4.0
```

## Running a Build

Run xcodebuild with argus intercepting the build:

```bash
BUILD_TRACE_ID=$(uuidgen)
XCBBUILDSERVICE_PATH=$(which argus) BUILD_TRACE_ID=$BUILD_TRACE_ID xcodebuild build -scheme MyScheme
```

## Querying Build Results

After the build completes, use these commands to analyze results:

```bash
# Get build summary
argus trace summary --build $BUILD_TRACE_ID

# Get compilation errors
argus trace errors --build $BUILD_TRACE_ID

# Get slowest targets
argus trace slowest-targets --build $BUILD_TRACE_ID --limit 5

# Get build bottlenecks
argus trace bottlenecks --build $BUILD_TRACE_ID
```

## Dependency Analysis

Detect dependency issues in your project:

```bash
# Show what each target imports
argus trace imports --build $BUILD_TRACE_ID

# Find implicit dependencies (imports not declared as dependencies)
argus trace implicit-deps --build $BUILD_TRACE_ID

# Find redundant dependencies (declared but never imported)
argus trace redundant-deps --build $BUILD_TRACE_ID
```

Add `--json` flag for structured output.

## Dependency Graph and Linking Analysis

Query the build dependency graph with product type and linking information to identify optimization opportunities:

```bash
# Get the full dependency graph with linking info
argus trace graph --build $BUILD_TRACE_ID

# Query what a specific target depends on (transitive)
argus trace graph --source MyApp

# Query only direct dependencies
argus trace graph --source MyApp --direct

# Query what depends on a specific target
argus trace graph --sink CoreKit

# Find the path between two targets
argus trace graph --source MyApp --sink CoreKit

# Filter by dependency type
argus trace graph --source MyApp --label package
argus trace graph --source MyApp --label framework
```

### Available Labels

- `target`: Project targets
- `package`: Swift package dependencies
- `framework`: Framework dependencies
- `xcframework`: XCFramework dependencies
- `sdk`: System SDK frameworks
- `bundle`: Bundle dependencies

### Output Fields

The graph output includes:
- `name`: Target name
- `productType`: Xcode product type identifier (e.g., `com.apple.product-type.framework`)
- `artifactKind`: The kind of artifact (e.g., `framework`, `staticLibrary`, `dynamicLibrary`)
- `machOType`: Mach-O type (`staticlib`, `mh_dylib`, `mh_execute`)
- `dependencies`: Array of linked dependencies with:
  - `kind`: Link kind (`static`, `dynamic`, `framework`)
  - `mode`: Link mode
  - `isSystem`: Whether it's a system dependency

### Optimizing Launch Time and Binary Size

Use the graph information combined with linking types to make optimization decisions:

**To improve launch time (reduce dylib loading):**
1. Query the graph to find dynamically linked dependencies: `argus trace graph --source MyApp --json`
2. Look for dependencies with `kind: "dynamic"` or `kind: "framework"`
3. Consider converting frequently-used internal frameworks from dynamic to static linking
4. Targets with `machOType: "mh_dylib"` that are only used by one app are candidates for static linking
5. Fewer dynamic libraries means fewer `dlopen` calls at launch

**To reduce binary size:**
1. Query the graph to identify statically linked dependencies
2. Look for large targets with `kind: "static"` that are shared across multiple apps
3. Converting these to dynamic frameworks allows code sharing and reduces total install size
4. Targets with `machOType: "staticlib"` used by multiple apps benefit from being dynamic

**Analysis workflow:**
```bash
# 1. Get the full graph to understand the dependency structure
argus trace graph --build latest --json > graph.json

# 2. Find all targets that depend on a library you want to optimize
argus trace graph --sink MyLibrary --json

# 3. Check if a target is statically or dynamically linked
argus trace graph --source MyApp --direct --json | jq '.dependencies[] | select(.name == "MyLibrary")'
```

**Decision criteria:**
- If a library is used by only one target: prefer **static** linking (better launch time)
- If a library is used by multiple apps/extensions: prefer **dynamic** linking (smaller total size)
- For debug builds: prefer **dynamic** linking (faster incremental builds)
- For release builds with App Thinning: static linking may be preferred

## Listing Previous Builds

```bash
# List all tracked projects
argus trace projects

# List builds for a project
argus trace builds --project my-app --limit 10
```
