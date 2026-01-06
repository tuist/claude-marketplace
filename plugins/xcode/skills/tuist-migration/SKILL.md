---
name: tuist-migrate
description: Guide for migrating existing Xcode projects, XcodeGen projects, or Swift Packages to Tuist. Use when helping with Tuist adoption, migration planning, or troubleshooting migration issues.
---

# Tuist Migration

A comprehensive guide for migrating projects to Tuist from various sources.

## Overview

Tuist provides a scalable solution for managing Apple platform projects. Migration complexity depends on your current setup:
- **Xcode Projects**: Manual but thorough, excellent cleanup opportunity
- **XcodeGen**: Swift over YAML, similar concepts, straightforward migration
- **Swift Package Manager**: Limited to .framework/.staticLibrary products, requires minimal .xcodeproj stub for .app bundles

## Migration Strategy

### Pre-Migration Setup

1. **Create project scaffold** with Tuist suffix to avoid conflicts:

```swift
// Tuist.swift
import ProjectDescription

let tuist = Tuist()
```

```swift
// Project.swift
import ProjectDescription

let project = Project(
    name: "MyApp-Tuist",
    targets: [
        /** Targets will go here **/
    ]
)
```

```swift
// Tuist/Package.swift
// swift-tools-version: 5.9
import PackageDescription

#if TUIST
    import ProjectDescription

    let packageSettings = PackageSettings(
        productTypes: [:] // Customize per package: ["Alamofire": .framework]
    )
#endif

let package = Package(
    name: "MyApp",
    dependencies: [
        // .package(url: "https://github.com/Alamofire/Alamofire", from: "5.0.0"),
    ]
)
```

2. **Set up CI validation** early:

```bash
tuist install
tuist generate
xcodebuild build {xcodebuild flags}
```

### Migration Order

**Always migrate from most depended-upon to least dependent targets.** Use this command to determine order:

```bash
tuist migration list-targets -p Project.xcodeproj
```

Start from the top of the list (targets with most dependencies on them).

## Step-by-Step Migration

### 1. Extract Project Build Settings

Extract project-level settings to `.xcconfig` files for cleaner management:

```bash
mkdir -p xcconfigs/
tuist migration settings-to-xcconfig -p MyApp.xcodeproj -x xcconfigs/MyApp-Project.xcconfig
```

Update `Project.swift`:

```swift
let project = Project(
    name: "MyApp",
    settings: .settings(configurations: [
        .debug(name: "Debug", xcconfig: "./xcconfigs/MyApp-Project.xcconfig"),
        .release(name: "Release", xcconfig: "./xcconfigs/MyApp-Project.xcconfig"),
    ]),
    targets: [...]
)
```

Add CI validation:

```bash
tuist migration check-empty-settings -p Project.xcodeproj
```

### 2. Extract Package Dependencies

Move all Swift Package dependencies to `Tuist/Package.swift`:

```swift
let package = Package(
    name: "MyApp",
    dependencies: [
        .package(url: "https://github.com/onevcat/Kingfisher", .upToNextMajor(from: "7.12.0"))
    ]
)
```

💡 **Tip**: Override product types when needed (default is `.staticFramework`):
```swift
let packageSettings = PackageSettings(
    productTypes: ["Alamofire": .framework]
)
```

### 3. Migrate Targets One-by-One

For each target (starting with most depended-upon):

#### a) Extract target build settings:

```bash
tuist migration settings-to-xcconfig -p MyApp.xcodeproj -t TargetX -x xcconfigs/TargetX.xcconfig
```

#### b) Define target in Project.swift:

```swift
.target(
    name: "TargetX",
    destinations: .iOS,
    product: .framework, // or .staticFramework, .staticLibrary, .app
    bundleId: "dev.tuist.targetX",
    sources: ["Sources/TargetX/**"],
    resources: ["Resources/**"], // if needed
    dependencies: [
        .external(name: "Kingfisher"), // SPM dependency
        .target(name: "OtherTarget"),  // Internal target
        .project(target: "SharedTarget", path: "../SharedModule") // Cross-project
    ],
    settings: .settings(configurations: [
        .debug(name: "Debug", xcconfig: "./xcconfigs/TargetX.xcconfig"),
        .release(name: "Release", xcconfig: "./xcconfigs/TargetX.xcconfig"),
    ])
)
```

#### c) Validate migration:

```bash
tuist generate
xcodebuild build -scheme TargetX
tuist test
```

💡 **Pro tip**: Use [xcdiff](https://github.com/bloomberg/xcdiff) to compare generated vs original projects

#### d) Handle test targets:

Define test targets alongside main targets:

```swift
.target(
    name: "TargetXTests",
    destinations: .iOS,
    product: .unitTests,
    bundleId: "dev.tuist.targetX.tests",
    sources: ["Tests/TargetXTests/**"],
    dependencies: [
        .target(name: "TargetX")
    ]
)
```

### 4. Repeat Until Complete

Continue migrating targets in dependency order. Create a PR for each target to ensure proper review and testing.

## Best Practices

### Use buildableFolders for Performance

Avoid unnecessary project regeneration:

```swift
let target = Target(
    name: "MyApp",
    destinations: [.iPhone],
    product: .app,
    bundleId: "com.example.app",
    buildableFolders: ["Sources/", "Resources/"]
)
```

### Generate Without Opening Xcode

For CI or quick validation:

```bash
tuist generate --no-open
```

### Optimize Build Settings

Enable caching and improve build performance:

```bash
xcodebuild build \
  -scheme MyScheme \
  COMPILATION_CACHE_ENABLE_CACHING=YES \
  CLANG_ENABLE_COMPILE_CACHE=YES \
  SWIFT_ENABLE_EXPLICIT_MODULES=YES \
  SWIFT_USE_INTEGRATED_DRIVER=YES
```

### Disable Signing for CI

```bash
xcodebuild build \
  -scheme MyScheme \
  CODE_SIGN_IDENTITY=""
```

### Code Reusability

Use ProjectDescriptionHelpers for reusable code:

```swift
// Tuist/ProjectDescriptionHelpers/Target+Features.swift
import ProjectDescription

extension Target {
    static func featureTargets(name: String) -> [Target] {
        return [
            .target(
                name: name,
                destinations: .iOS,
                product: .framework,
                bundleId: "com.example.\(name.lowercased())",
                sources: ["Sources/\(name)/**"]
            ),
            .target(
                name: "\(name)Tests",
                destinations: .iOS,
                product: .unitTests,
                bundleId: "com.example.\(name.lowercased()).tests",
                sources: ["Tests/\(name)Tests/**"],
                dependencies: [.target(name: name)]
            )
        ]
    }
}
```

Then in `Project.swift`:

```swift
import ProjectDescriptionHelpers

let project = Project(
    name: "MyApp",
    targets: Target.featureTargets(name: "Authentication")
)
```

## Source-Specific Migration Tips

### From Xcode Projects

**Challenges:**
- Messy accumulated complexity (groups not matching directories, missing file references)
- Implicit configurations
- Manual work required

**Benefits:**
- Perfect opportunity to clean up project structure
- Align file system with target structure
- Remove dead code and references

**Key Command:**
```bash
tuist edit  # Edit manifests in Xcode with autocompletion
```

### From XcodeGen

**Advantages:**
- Similar concepts (both generate Xcode projects)
- Swift > YAML (better IDE support, type safety)
- ProjectDescriptionHelpers > Templates (native Swift reusability)

**Migration:**
- `project.yaml` → `Project.swift`
- Optionally add `Workspace.swift` for multi-project workspaces
- Replace XcodeGen templates with Swift extensions

### From Swift Package Manager

**Why migrate?**
- Performance: SPM can cause 15+ second file rename indexing
- Flexibility: Limited to framework/library products
- Scale: Designed as dependency manager, not project manager

**Key differences:**
- `PackageDescription` → `ProjectDescription`
- `package` → `project`
- Xcode language (targets, schemes, build phases)
- Need minimal .xcodeproj stub for .app bundles (or migrate fully)

**Performance data** (from Bumble article):
- SPM file operations: 10-30s waits on simple tasks
- Tuist: Equivalent to native .xcodeproj performance
- `tuist generate` cached: ~5-10s for large projects

## Troubleshooting

### Missing Files After Migration

If files aren't compiling:
1. Check that file structure matches glob patterns in `sources`
2. Verify all files are in correct directories
3. Use this as an opportunity to align filesystem with target structure

### Build Setting Conflicts

If builds fail with setting-related errors:
1. Check `.xcconfig` files were extracted correctly
2. Ensure no duplicate settings between `.xcconfig` and manifest
3. Use `tuist migration check-empty-settings` to verify

### Performance Issues with tuist generate

1. Exclude DerivedData and Tuist cache from security scanners
2. Check if internal security tools are slowing file operations
3. Consider using `tuist generate --no-open` for faster generation

### SPM Graph Resolution Slowness

If coming from SPM and experiencing slow graph resolution:
1. Reduce graph size with transitive reduction
2. Minimize redundant edges in dependency graph
3. Note: Tuist resolves graph only during `generate`, not continuously

## Performance Expectations

Based on real-world data (Bumble case study with 500-750 modules):

| Operation | Native Xcode | SPM | Tuist |
|-----------|--------------|-----|-------|
| Build time | Baseline | ~Same | ~Same |
| Indexing | Baseline | -20% worse | ~Same |
| File rename | <1s | 10-30s | <1s |
| Graph resolution | N/A | 5-40s (frequent!) | 5-10s (explicit only) |
| Project open | Baseline | Slow | ~2-5s added |

**Key insight**: Tuist graph resolution is **explicit** (`tuist generate`), not continuous like SPM.

## Migration Checklist

- [ ] Create Tuist project scaffold (Tuist.swift, Project.swift, Tuist/Package.swift)
- [ ] Add CI validation for Tuist project
- [ ] Extract project build settings to .xcconfig
- [ ] Add check-empty-settings validation to CI
- [ ] Move SPM dependencies to Tuist/Package.swift
- [ ] Determine target migration order with `list-targets`
- [ ] For each target:
  - [ ] Extract target build settings to .xcconfig
  - [ ] Define target in Project.swift
  - [ ] Validate with `tuist generate` and xcodebuild
  - [ ] Compare with xcdiff if needed
  - [ ] Create PR for review
- [ ] Update CI/CD to use `tuist generate`
- [ ] Remove -Tuist suffix from project name
- [ ] Archive or delete old .xcodeproj files

## Common Commands Reference

```bash
# Migration helpers
tuist migration list-targets -p Project.xcodeproj
tuist migration settings-to-xcconfig -p Project.xcodeproj -x output.xcconfig
tuist migration settings-to-xcconfig -p Project.xcodeproj -t TargetName -x target.xcconfig
tuist migration check-empty-settings -p Project.xcodeproj

# Core workflow
tuist edit                    # Edit manifests in Xcode
tuist generate                # Generate Xcode projects
tuist generate --no-open      # Generate without opening
tuist install                 # Install dependencies
tuist test                    # Run tests
tuist clean                   # Clean generated artifacts

# Validation
xcodebuild build -scheme MyScheme
tuist test
```

## Resources

- [Official Tuist Docs](https://docs.tuist.dev)
- [Tuist Migration Guides](https://docs.tuist.dev/guides/features/projects/adoption/migrate/)
- [Scaling iOS at Bumble](https://medium.com/bumble-tech/scaling-ios-at-bumble-239e0fa009f2) - Real-world SPM vs Tuist comparison
- [xcdiff](https://github.com/bloomberg/xcdiff) - Compare Xcode projects
- [XcodeGen](https://github.com/yonaskolb/XcodeGen) - Alternative project generator

## When to Ask for Help

Migration can be complex. Consult with the team or Tuist community when:
- Unusual project structures or edge cases
- Third-party dependencies with complex requirements
- Custom build phases or scripts
- Performance issues after migration
- Build setting conflicts that aren't resolved by .xcconfig extraction

Remember: **Migration is an opportunity to clean up and modernize your project structure!**
