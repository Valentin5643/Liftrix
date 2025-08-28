# Liftrix Setup Guide

This repository does **not** contain the full Liftrix source code.  
Instead, it provides:  
- The official Liftrix APK (for testing/preview)  
- Example code snippets and utilities  
- Documentation for contributors  

You cannot build the entire Liftrix app from this repo, but you can still:  
- Install and test the APK  
- Explore and run the provided example code  
- Contribute improvements to documentation and code snippets  

---

## Prerequisites

### For Running the APK
- **Device**: Android 7.0 (API 24) or later  
- **Storage**: ~200MB free  
- **Emulator (optional)**: Android Studio emulator (API 30+)  

### For Running Example Code
- **Android Studio**: Latest stable version (e.g. Jellyfish 2023.3.1 or later)  
- **JDK**: 17+  
- **Gradle**: Wrapper included in examples  
- **Kotlin**: 1.9+  

---

## Setup Instructions

### 1. Install the APK

```bash
# On your device (USB debugging enabled):
adb install liftrix.apk
```

Or simply copy the APK to your device and install it manually.

### 2. Run Example Code

The `docs/` folder contains small snippets.

To try one out:

```bash
# Clone repo
git clone Valentin5643/Liftrix.git
cd liftrix

# Open in Android Studio
# Run on emulator or device
```

Each example demonstrates a concept used in Liftrix.

---

## Contribution Workflow

### Reporting Issues
- Test the APK and open an issue if you find bugs
- Run examples and report crashes or incorrect behavior
- Always include clear reproduction steps

### Suggesting Improvements
- Propose new examples (e.g., Firebase usage, Room database, Compose UI patterns)
- Improve documentation for beginners
- Suggest better packaging/distribution methods for the APK

### Pull Requests
- Keep changes focused and small
- Document all new example code with comments
- Ensure examples compile and run independently
- Update README.md if adding new examples

---

## Testing Contributions

### APK Testing
- Install on emulator (API 30+) and at least one real device if possible
- Report differences between environments

### Code Example Testing
- Open example in Android Studio
- Run on emulator/device
- Confirm expected output/behavior

---

## Documentation Standards
- Keep instructions beginner-friendly
- Add comments to all example code
- Use Markdown for formatting
- Link back to official Android/Firebase docs when useful

---

## Common Issues

### 1. APK Installation Fails
- Ensure "Install from unknown sources" is enabled
- Use:
```bash
adb install -r liftrix.apk
```
to update an existing version

### 2. Example Project Won't Sync
```bash
./gradlew --stop
./gradlew clean build --refresh-dependencies
```

### 3. Emulator Too Slow
- Allocate at least 4GB RAM to the emulator
- Use x86_64 images with hardware acceleration enabled

---

## Verification Checklist

After setup, verify:
- ✓ APK installs and runs on device/emulator
- ✓ Example projects compile in Android Studio
- ✓ Example code runs as expected
- ✓ Documentation matches actual behavior
