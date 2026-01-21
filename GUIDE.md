# How to Compile WebRTC for iOS: A Complete Guide

Building WebRTC from source for iOS is notoriously complex. Google's WebRTC codebase is massive, uses custom build tooling, and requires careful configuration to produce a usable framework. This guide walks through the entire process.

## Why Build From Source?

Most developers should use a precompiled binary. But you might need to build from source if you:

- Need a specific WebRTC version or commit
- Want to enable/disable specific features or codecs
- Need custom modifications to the source
- Require a specific iOS deployment target

## Prerequisites

**System Requirements:**

- macOS with Xcode installed
- ~25GB free disk space
- Stable internet connection
- Python 3.x

**Install Google's depot_tools:**

```bash
cd ~
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
echo 'export PATH="$HOME/depot_tools:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

Verify installation:

```bash
which gn && which ninja
```

## Step 1: Fetch the WebRTC Source

Create a working directory and fetch the iOS-specific WebRTC checkout:

```bash
mkdir ~/webrtc-build && cd ~/webrtc-build
fetch --nohooks webrtc_ios
gclient sync
```

This downloads ~15GB of source code and dependencies.

## Step 2: Select a Branch

WebRTC uses branch names like `branch-heads/XXXX`. Check [chromiumdash.appspot.com/branches](https://chromiumdash.appspot.com/branches) for stable releases.

```bash
cd src
git checkout branch-heads/6723  # Example: M131 release
gclient sync
```

## Step 3: Configure Build Targets

WebRTC uses `gn` (Generate Ninja) for build configuration. You need three builds:

### iOS Device (arm64)

```bash
gn gen out/ios_device --args='
target_os="ios"
target_cpu="arm64"
is_debug=false
is_component_build=false
rtc_include_tests=false
rtc_libvpx_build_vp9=true
ios_enable_code_signing=false
ios_deployment_target="16.0"
use_xcode_clang=true
'
```

### iOS Simulator (arm64 - Apple Silicon Macs)

```bash
gn gen out/ios_sim_arm64 --args='
target_os="ios"
target_cpu="arm64"
target_environment="simulator"
is_debug=false
is_component_build=false
rtc_include_tests=false
rtc_libvpx_build_vp9=true
ios_enable_code_signing=false
ios_deployment_target="16.0"
use_xcode_clang=true
'
```

### iOS Simulator (x86_64 - Intel Macs)

```bash
gn gen out/ios_sim_x64 --args='
target_os="ios"
target_cpu="x64"
target_environment="simulator"
is_debug=false
is_component_build=false
rtc_include_tests=false
rtc_libvpx_build_vp9=true
ios_enable_code_signing=false
ios_deployment_target="16.0"
use_xcode_clang=true
'
```

### Key Build Arguments Explained

| Argument | Purpose |
|----------|---------|
| `target_os="ios"` | Build for iOS platform |
| `target_cpu` | Architecture (arm64/x64) |
| `target_environment="simulator"` | Required for simulator builds |
| `is_debug=false` | Release build with optimizations |
| `rtc_include_tests=false` | Skip test code (faster build) |
| `rtc_libvpx_build_vp9=true` | Include VP9 codec support |
| `ios_deployment_target` | Minimum iOS version |
| `ios_enable_code_signing=false` | Skip signing for frameworks |

## Step 4: Build

Run ninja for each target:

```bash
ninja -C out/ios_device framework_objc
ninja -C out/ios_sim_arm64 framework_objc
ninja -C out/ios_sim_x64 framework_objc
```

Each build takes 15-30 minutes depending on your machine. The output is a `WebRTC.framework` in each output directory.

## Step 5: Create Universal Simulator Framework

Combine both simulator architectures into a fat binary:

```bash
# Create working copy
cp -R out/ios_sim_arm64/WebRTC.framework WebRTC_sim.framework

# Merge binaries with lipo
lipo -create \
  out/ios_sim_arm64/WebRTC.framework/WebRTC \
  out/ios_sim_x64/WebRTC.framework/WebRTC \
  -output WebRTC_sim.framework/WebRTC

# Verify architectures
lipo -info WebRTC_sim.framework/WebRTC
# Output: arm64 x86_64
```

## Step 6: Create XCFramework

Apple's XCFramework format bundles multiple platforms into a single distributable:

```bash
xcodebuild -create-xcframework \
  -framework out/ios_device/WebRTC.framework \
  -framework WebRTC_sim.framework \
  -output WebRTC.xcframework
```

Your final `WebRTC.xcframework` (~30MB) is ready for distribution.

## Step 7: Distribute via Swift Package Manager

Create a repository with this structure:

```
your-webrtc-ios/
├── Package.swift
├── README.md
└── WebRTC.xcframework/
```

**Package.swift:**

```swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "WebRTC",
    platforms: [.iOS(.v16)],
    products: [
        .library(name: "WebRTC", targets: ["WebRTC"])
    ],
    targets: [
        .binaryTarget(
            name: "WebRTC",
            path: "WebRTC.xcframework"
        )
    ]
)
```

Push to GitHub and users can add it via:

```swift
.package(url: "https://github.com/you/webrtc-ios.git", from: "1.0.0")
```

## Troubleshooting

**"gn: command not found"**
Ensure depot_tools is in your PATH and you've opened a new terminal.

**Build fails with Xcode errors**
WebRTC requires specific Xcode versions. Check the [WebRTC release notes](https://webrtc.googlesource.com/src/+/refs/heads/main/docs/native-code/ios/index.md).

**Missing architectures**
Run `lipo -info` on your framework binary to verify all expected architectures are present.

**Simulator build crashes on Apple Silicon**
Make sure you built the arm64 simulator variant, not just x86_64.

## Automation Script

Here's a script that automates the full process:

```bash
#!/bin/bash
set -e

BRANCH="${1:-branch-heads/6723}"
IOS_TARGET="16.0"
OUTPUT_DIR="$(pwd)/build"

cd src
git fetch
git checkout $BRANCH
gclient sync

# Build all targets
for config in "ios_device arm64" "ios_sim_arm64 arm64 simulator" "ios_sim_x64 x64 simulator"; do
  read -r name cpu env <<< "$config"
  args="target_os=\"ios\" target_cpu=\"$cpu\" is_debug=false"
  args+=" rtc_include_tests=false ios_enable_code_signing=false"
  args+=" ios_deployment_target=\"$IOS_TARGET\""
  [[ -n "$env" ]] && args+=" target_environment=\"$env\""

  gn gen "out/$name" --args="$args"
  ninja -C "out/$name" framework_objc
done

# Create fat simulator binary
cp -R out/ios_sim_arm64/WebRTC.framework "$OUTPUT_DIR/WebRTC_sim.framework"
lipo -create out/ios_sim_*/WebRTC.framework/WebRTC \
  -output "$OUTPUT_DIR/WebRTC_sim.framework/WebRTC"

# Create xcframework
xcodebuild -create-xcframework \
  -framework out/ios_device/WebRTC.framework \
  -framework "$OUTPUT_DIR/WebRTC_sim.framework" \
  -output "$OUTPUT_DIR/WebRTC.xcframework"

echo "Done: $OUTPUT_DIR/WebRTC.xcframework"
```

## Additional Resources

- [Official WebRTC iOS Documentation](https://webrtc.googlesource.com/src/+/refs/heads/main/docs/native-code/ios/index.md)
- [WebRTC Source Code](https://webrtc.googlesource.com/src)
- [Chromium Branch Dashboard](https://chromiumdash.appspot.com/branches)

## About This Repository

This repository contains a precompiled WebRTC.xcframework built using the process described above. See the [README](README.md) for version info and usage instructions.
