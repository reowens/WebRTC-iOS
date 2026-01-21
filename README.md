# WebRTC for iOS

Custom build of Google WebRTC for iOS, compatible with Xcode 26.

## Version Info

- **WebRTC Branch:** branch-heads/6723
- **Commit:** 28b793b4dd
- **Built:** 2025-12-16
- **iOS Deployment Target:** 16.0

## Architectures

- iOS Device: arm64
- iOS Simulator: arm64 + x86_64

## Usage

Add to your Package.swift:

```swift
dependencies: [
    .package(url: "https://github.com/beyond-labs-io/WebRTC-iOS.git", from: "1.0.0")
]
```

Or in Xcode: File → Add Package Dependencies → Enter repo URL

## Building From Source

Want to compile WebRTC yourself? See the full **[Build Guide](GUIDE.md)** for step-by-step instructions.

This package was built from `branch-heads/6723` using the process described in that guide.
