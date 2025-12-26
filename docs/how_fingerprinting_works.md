# How Device Fingerprinting and Device ID Work

This document provides a detailed technical explanation of how FingerprintJS Android derives device fingerprints and device IDs, and the key differences between these two identification methods.

## Table of Contents

1. [Overview](#overview)
2. [Device Fingerprint Derivation](#device-fingerprint-derivation)
3. [Device ID Derivation](#device-id-derivation)
4. [Key Differences](#key-differences)
5. [Choosing Between Fingerprint and Device ID](#choosing-between-fingerprint-and-device-id)

## Overview

FingerprintJS Android provides two complementary approaches to device identification:

- **Device Fingerprint**: A computed hash derived from multiple device signals/attributes
- **Device ID**: A single stable identifier retrieved from the Android system

Both serve different purposes and have different stability characteristics.

## Device Fingerprint Derivation

### What is a Device Fingerprint?

A device fingerprint is a hash value computed from a combination of various device signals and attributes. Unlike a device ID, which is a single value retrieved from the system, a fingerprint is **calculated** by collecting multiple device characteristics and hashing them together.

### How Fingerprints are Derived

The fingerprint derivation process follows these steps:

#### 1. Signal Collection

The library collects various signals (attributes) from the device across multiple categories:

**Hardware Signals:**
- Manufacturer name
- Model name
- Total RAM
- Total internal storage space
- CPU information (`/proc/cpuinfo`)
- Number of CPU cores
- ABI type (Application Binary Interface)
- Available sensors
- Input devices
- Battery health and capacity
- Camera list
- GPU information (OpenGL ES version)

**OS Build Signals:**
- Android version
- SDK version
- Kernel version
- Build fingerprint (OS build signature)
- Encryption status
- Codec list (audio/video codecs)
- Security providers

**Device State Signals:**
- Developer options enabled
- ADB (Android Debug Bridge) enabled
- HTTP proxy settings
- Animation scale settings
- Data roaming enabled
- Accessibility settings
- Default input method
- RTT calling mode
- Touch exploration enabled
- System sound settings (alarm, ringtone)
- Date format
- Font scale
- Screen timeout
- Text auto-correction settings
- 12/24 hour format
- PIN security status
- Fingerprint sensor status
- Available locales
- Region/country
- Default language
- Timezone

**Installed Apps Signals:**
- List of installed applications
- List of system applications

#### 2. Signal Versioning and Filtering

Not all signals are used in every version. The library uses a versioning system (V_1 through V_6) to control which signals are included:

- Each signal has metadata indicating when it was added (`addedInVersion`) and optionally when it was removed (`removedInVersion`)
- When you request a fingerprint with a specific version, only signals appropriate for that version are included
- This ensures fingerprint stability across library updates

#### 3. Stability Level Filtering

Signals are also categorized by stability level:

- **OPTIMAL**: Balanced between stability and uniqueness (default)
- **STABLE**: Prioritizes stability, excludes signals that change frequently
- **UNIQUE**: Prioritizes uniqueness, includes more signals even if they may change

The stability level you choose filters which signals are included in the fingerprint calculation.

#### 4. String Concatenation

Once the appropriate signals are collected based on version and stability level:

1. Each signal's value is converted to a "hashable string" representation
2. All signal strings are concatenated together in a consistent order
3. The order is determined by the signal's declaration order in the code to ensure consistency

#### 5. Hashing

The concatenated string is then hashed using a hashing algorithm:

**Default Hash Algorithm: MurmurHash3 (x64_128)**
- Fast, non-cryptographic hash function
- Produces a 128-bit hash value
- Converts the long concatenated string into a shorter, fixed-length hash
- Output is represented as a hexadecimal string

**Example simplified process:**
```
Signals: ["Samsung", "SM-G991B", "8GB RAM", "128GB Storage", ...]
         ↓
Concatenated: "SamsungSM-G991B8GB128GB..."
         ↓
Hashed (MurmurHash3): "a7f2c9e3b1d8f4a6c2e9d7b5f3a1c8e4"
         ↓
Device Fingerprint: "a7f2c9e3b1d8f4a6c2e9d7b5f3a1c8e4"
```

### Legacy vs Modern Fingerprinting (Versions)

**Legacy Approach (V_1 to V_4):**
- Signals were grouped into categories (hardware, OS, device state, installed apps)
- Each group was hashed separately
- Group hashes were concatenated and hashed again
- This created a two-level hashing structure

**Modern Approach (V_5 and later):**
- All signals are treated as a flat list
- Signal strings are concatenated directly
- Single hash operation is performed
- Simpler and more efficient

### Custom Fingerprinting

You can also create custom fingerprints by:

1. Getting the `FingerprintingSignalsProvider`
2. Selecting specific signals you want to use
3. Calling `getFingerprint(fingerprintingSignals, hasher)` with your custom signal list

This allows you to:
- Use only specific signals relevant to your use case
- Implement custom hashing algorithms
- Fine-tune the stability/uniqueness tradeoff for your needs

## Device ID Derivation

### What is a Device ID?

A device ID is a single, stable identifier that is retrieved directly from the Android system. Unlike a fingerprint (which is computed), a device ID is **retrieved** from system APIs.

### How Device IDs are Derived

The device ID derivation follows a priority-based fallback approach:

#### 1. ID Source Priority

The library attempts to retrieve IDs from multiple sources in priority order:

**Priority Order (Version V_3 and later):**
1. **GSF ID (Google Services Framework ID)** - First choice
2. **Media DRM ID** - Second choice (fallback)
3. **Android ID** - Last resort (always available)

**Priority Order (Version V_1 and V_2):**
1. **GSF ID** - First choice
2. **Android ID** - Fallback (Media DRM ID not used)

#### 2. ID Retrieval Methods

**GSF ID (Google Services Framework ID):**
```kotlin
// Retrieved from Google Services Framework content provider
URI: "content://com.google.android.gsf.gservices"
Key: "android_id"
```
- Requires Google Play Services to be installed
- Unique per Google account on the device
- Returns hexadecimal string representation of a 64-bit number
- May be empty if Google Play Services is not available
- Remains stable across app reinstalls
- Changes after factory reset or when Google account is removed

**Media DRM ID:**
```kotlin
// Retrieved from Widevine Media DRM
UUID: Widevine UUID (edef8ba9-79d6-4ace-a3c8-27dcd51d21ed)
```
- Uses Widevine DRM's device unique ID
- Processed through SHA-256 hashing
- Represented as hexadecimal string
- Available on most modern Android devices
- More stable than GSF ID (survives factory reset on some devices)
- May be empty if DRM is not available or accessible

**Android ID:**
```kotlin
// Retrieved from Android system settings
Settings.Secure.getString(contentResolver, Settings.Secure.ANDROID_ID)
```
- Always available on Android devices
- Unique per app installation (on Android 8.0+) or per device (on older versions)
- Returns hexadecimal string representation of a 64-bit number
- Changes after factory reset
- Behavior varies significantly across Android versions and manufacturers

#### 3. Fallback Logic

The library implements a waterfall approach:

```
1. Try GSF ID
   ↓ (if empty)
2. Try Media DRM ID (V_3+)
   ↓ (if empty or not applicable)
3. Use Android ID (always available)
```

This ensures that:
- You always get a device ID (Android ID is always available)
- You get the most stable ID available on the device
- The system gracefully handles cases where some IDs are unavailable

#### 4. Result Structure

When you call `getDeviceId()`, you receive a `DeviceIdResult` object containing:

```kotlin
data class DeviceIdResult(
    val deviceId: String,      // The selected ID based on priority
    val gsfId: String,          // GSF ID (may be empty)
    val androidId: String,      // Android ID (always present)
    val mediaDrmId: String,     // Media DRM ID (may be empty)
)
```

This allows you to:
- Use the automatically selected `deviceId`
- Access individual ID sources if needed
- Implement your own selection logic

## Key Differences

### Conceptual Differences

| Aspect | Device Fingerprint | Device ID |
|--------|-------------------|-----------|
| **Nature** | Computed hash from multiple signals | Retrieved identifier from system |
| **Source** | Derived from 50+ device attributes | Single system identifier |
| **Computation** | Requires signal collection + hashing | Direct API call |
| **Uniqueness** | High (but not guaranteed unique) | Unique per device/account |
| **Randomness** | Not random (deterministic from signals) | Random system-generated ID |

### Derivation Process Differences

**Device Fingerprint:**
```
Collect Signals → Filter by Version/Stability → Concatenate → Hash → Fingerprint
(Multi-step computation)
```

**Device ID:**
```
Try GSF ID → Try Media DRM ID → Use Android ID → Device ID
(Priority-based retrieval)
```

### Stability Differences

| Event | Device Fingerprint | Device ID |
|-------|-------------------|-----------|
| **App reinstall** | ✅ Same | ✅ Same (usually) |
| **Factory reset** | 🟡 May change (some hardware remains) | ❌ Changes |
| **System update** | 🟡 May change (OS signals change) | ✅ Same |
| **Settings changes** | ❌ Changes (if settings are in signals) | ✅ Same |
| **Hardware change** | ❌ Changes | ✅ Same (usually) |
| **Google account change** | ✅ Same | 🟡 GSF ID changes |

See the [stability documentation](stability.md) for detailed stability tables.

### Use Case Differences

**Device Fingerprint is better for:**
- Fraud detection and security
- Scenarios where IDs might be spoofed
- Cross-account device tracking
- Identifying device characteristics
- Hardware-based identification

**Device ID is better for:**
- User session management
- Personalization within an app
- Analytics and attribution
- Simple device identification
- Cases where stability is critical

### Technical Implementation Differences

**Fingerprint:**
- Requires Android permissions for some signals (camera, sensors, etc.)
- More computationally intensive (signal collection + hashing)
- Cached after first computation
- Can be customized with different signals and hash functions
- Version and stability level affect the result

**Device ID:**
- Minimal permissions required
- Very fast (just system API calls)
- Cached after first retrieval
- No customization options
- Only version affects the result (not stability level)

### Collision Probability

**Device Fingerprint:**
- Low but non-zero collision probability
- Two identical devices (same model, same OS version, similar apps) might have the same fingerprint
- Probability decreases with more unique signals
- Stability level affects collision probability (UNIQUE has lower collision risk)

**Device ID:**
- Virtually zero collision probability
- Each device/account combination has a unique ID
- Cryptographically random values
- Guaranteed unique by the system

### Spoofing Resistance

**Device Fingerprint:**
- Harder to spoof (requires changing many device attributes)
- No single value to manipulate
- Requires root access or sophisticated tools to fake multiple signals
- Detects emulators and modified devices better

**Device ID:**
- Easier to spoof (single value to manipulate)
- Root access can allow ID modification
- Emulators can easily fake these IDs
- Less secure for fraud detection

## Choosing Between Fingerprint and Device ID

### Use Device ID when:
- ✅ You need a simple, stable identifier
- ✅ User experience matters (faster, less resource-intensive)
- ✅ You're doing basic analytics or personalization
- ✅ You trust the environment (legitimate app users)
- ✅ You need guaranteed uniqueness

### Use Device Fingerprint when:
- ✅ Security is a priority
- ✅ You need to detect fraudulent activities
- ✅ You want to track devices across factory resets
- ✅ You need to identify hardware characteristics
- ✅ You want to detect rooted devices or emulators
- ✅ You need cross-account device tracking

### Use Both when:
- ✅ You need a comprehensive identification solution
- ✅ You want both stability (device ID) and security (fingerprint)
- ✅ You're building a fraud detection system
- ✅ You want to correlate multiple identification methods

## Code Examples

### Getting a Device Fingerprint

```kotlin
val fingerprinter = FingerprinterFactory.create(context)

// Get fingerprint with default settings (V_5, OPTIMAL stability)
fingerprinter.getFingerprint(version = Fingerprinter.Version.V_5) { fingerprint ->
    println("Device fingerprint: $fingerprint")
    // Example output: "a7f2c9e3b1d8f4a6c2e9d7b5f3a1c8e4"
}

// Get fingerprint with high uniqueness
fingerprinter.getFingerprint(
    version = Fingerprinter.Version.V_5,
    stabilityLevel = StabilityLevel.UNIQUE
) { fingerprint ->
    println("Unique fingerprint: $fingerprint")
}
```

### Getting a Device ID

```kotlin
val fingerprinter = FingerprinterFactory.create(context)

// Get device ID
fingerprinter.getDeviceId(version = Fingerprinter.Version.V_5) { result ->
    println("Selected device ID: ${result.deviceId}")
    println("GSF ID: ${result.gsfId}")
    println("Android ID: ${result.androidId}")
    println("Media DRM ID: ${result.mediaDrmId}")
}
```

### Custom Fingerprinting

```kotlin
@WorkerThread
fun buildCustomFingerprint(fingerprinter: Fingerprinter): String {
    val signalsProvider = fingerprinter.getFingerprintingSignalsProvider()
    
    // Select only hardware signals for a hardware-based fingerprint
    val hardwareSignals = listOf(
        signalsProvider.manufacturerNameSignal,
        signalsProvider.modelNameSignal,
        signalsProvider.totalRamSignal,
        signalsProvider.coresCountSignal,
        signalsProvider.abiTypeSignal,
    )
    
    return fingerprinter.getFingerprint(
        fingerprintingSignals = hardwareSignals
    )
}
```

## Summary

- **Device Fingerprint**: A computed hash from multiple device signals, offering higher security and spoofing resistance but with possible collisions
- **Device ID**: A retrieved system identifier, offering guaranteed uniqueness and stability but easier to spoof
- The choice between them depends on your specific use case, security requirements, and stability needs
- Both methods are cached internally for performance
- Both support versioning to maintain stability across library updates
