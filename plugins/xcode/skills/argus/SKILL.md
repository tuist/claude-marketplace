---
name: argus-build-diagnostics
description: Diagnose xcodebuild builds using argus. Use this instead of parsing xcodebuild output to avoid filling the context window. Query build errors, warnings, slowest targets, bottlenecks, and implicit/redundant dependencies.
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

## Listing Previous Builds

```bash
# List all tracked projects
argus trace projects

# List builds for a project
argus trace builds --project my-app --limit 10
```
