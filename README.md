# Mobile Badge SDK

Secure BLE-based mobile authentication for Elatec devices. The Mobile Badge SDK enables Android applications to communicate with Elatec BLE readers using digital passes with biometric verification, providing a seamless and secure authentication experience.

## Features

- **Biometric Authentication** - Verify user identity with fingerprint, face recognition, or device PIN/pattern
- **BLE Device Discovery** - Automatic scanning and RSSI-based distance detection for nearby Elatec readers
- **Background Operation** - Continue scanning when app is backgrounded with foreground service support
- **Hands-Free Mode** - Automatic authentication when devices are in range, no user interaction needed
- **Deep Link Enrollment** - One-tap user enrollment via invitation URLs
- **Real-Time Status** - Monitor Bluetooth, location, pass validity, and authentication status
- **Connection Locks** - Granular control over when automatic authentication can occur
- **Offline Support** - Continue working with cached credentials when network is unavailable

## Installation

1. Add Jitpack Maven repository to `settings.gradle.kts`

```kotlin
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        maven { url = uri("https://jitpack.io") } // <-- Add Jitpack Maven repository
    }
}
```

2. Add the dependency to your app's `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.github.elatec-mobile-credentials:mcm-mobile-badge-sdk-android:1.0")
}
```

## Backup Configuration

The SDK stores sensitive data (database and encrypted preferences) that should be excluded from Android's auto-backup to prevent data corruption.

Add the following attributes to your app's `AndroidManifest.xml`:

```xml
<application
    android:allowBackup="true"
    android:fullBackupContent="@xml/backup_descriptor"
    android:dataExtractionRules="@xml/backup_rules">
```

Create `res/xml/backup_rules.xml` (Android 12+):

```xml
<data-extraction-rules>
    <cloud-backup>
        <exclude domain="database" path="mobile_badge.db" />
        <exclude domain="sharedpref" path="datastore/enrollment_auth.preferences_pb" />
        <exclude domain="sharedpref" path="datastore/enrollment_auth_no_biometric.preferences_pb" />
    </cloud-backup>
</data-extraction-rules>
```

Create `res/xml/backup_descriptor.xml` (Android 11 and below):

```xml
<full-backup-content>
    <exclude domain="database" path="mobile_badge.db" />
    <exclude domain="sharedpref" path="datastore/enrollment_auth.preferences_pb" />
    <exclude domain="sharedpref" path="datastore/enrollment_auth_no_biometric.preferences_pb" />
</full-backup-content>
```

## Runtime Permissions

Your app must request these runtime permissions for the SDK to function correctly:

#### Required for BLE Scanning

**`ACCESS_FINE_LOCATION`** (All API levels)
- Required by Android for BLE device discovery
- The SDK does not access device location - this is an Android platform requirement for BLE scanning

**`BLUETOOTH_SCAN`** (API 31+)
- Allows the app to discover BLE devices
- Required to scan for Elatec readers

**`BLUETOOTH_CONNECT`** (API 31+)
- Allows the app to connect to BLE devices
- Required to authenticate with Elatec readers

#### Optional for Background Mode

**`POST_NOTIFICATIONS`** (API 33+)
- Required to display the foreground service notification when background mode is enabled
- Only needed if you plan to use background scanning

## Getting Started

### 1. Initialize the SDK

Initialize the SDK in your `Application.onCreate()` method. This must be done before any other SDK functionality is used:

**Standard Edition**

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        MobileBadge.init(
            clientConfig = null,
            isDebugBuild = BuildConfig.DEBUG,
            loggers = setOf(AndroidSink()),
            startupConnectionLocks = emptySet(),
            bleInitialValues = BleInitialValues.Default,
            bleServiceNotificationFactory = MyNotificationFactory()
        )
    }
}
```

**Managed Edition**

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        val clientConfig = ClientConfig(
            apiKey: <API Key>,
            apiKeyId: <API Key ID>,
        )

        MobileBadge.init(
            clientConfig = clientConfig,
            isDebugBuild = BuildConfig.DEBUG,
            loggers = setOf(AndroidSink()),
            startupConnectionLocks = emptySet(),
            bleInitialValues = BleInitialValues.Default,
            bleServiceNotificationFactory = MyNotificationFactory()
        )
    }
}
```


**Parameters:**
- `apiKey` - Your Mobile Badge API key provided by Elatec
- `isDebugBuild` - Set to `BuildConfig.DEBUG` to enable debug logging
- `loggers` - Set of logger implementations (use `AndroidSink()` for Logcat output)
- `startupConnectionLocks` - Initial connection locks to apply (usually empty set)
- `bleInitialValues` - Initial BLE configuration (see Persisting Settings below)
- `bleServiceNotificationFactory` - Factory for creating the background service notification

### 2. Request Runtime Permissions

Request the necessary runtime permissions before starting BLE operations:

```kotlin
// Build permission list based on API level
val permissions = buildList {
    add(Manifest.permission.ACCESS_FINE_LOCATION)
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        add(Manifest.permission.BLUETOOTH_SCAN)
        add(Manifest.permission.BLUETOOTH_CONNECT)
    }
    // Only needed for background mode
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        add(Manifest.permission.POST_NOTIFICATIONS)
    }
}
```

### 3. Start BLE Scanning

You need to manually start BLE scanning and to notify the SDK of changes in permissions status, you do both by calling
```kotlin
val mobileBadge = MobileBadge.getInstance()
mobileBadge.bleManager.startIfNotRunning()
```
You need to call `bleManager.startIfNotRunning()`:
- When required permissions are granted.
- In your main activity's `onResume`
- In your main activity's `onRequestPermissionsResult`

### 4. Handle Enrollment Invitations

Declare intent filters in your manifest for enrollment deep links:

```xml
<activity android:name=".MainActivity">
    // Other filters...
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" />
        <data android:host="*.portal.elatec.com" />
        <data android:host="*.dev.portal.elatec-integration.com" />
        <data android:host="*.playground.portal.elatec-integration.com" />
        <data android:host="*.qa.portal.elatec-integration.com" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="elatec-mb" />
        <data android:host="register" />
    </intent-filter>
</activity>
```

Handle the deep link in your Activity:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    val uri = intent.data
    if (uri != null) {
        mobileBadge.invitationManager.handleDeepLink(uri)
    }
}

override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    val uri = intent.data
    if (uri != null) {
        mobileBadge.invitationManager.handleDeepLink(uri)
    }
}

// Observe invitations in your ViewModel
viewModelScope.launch {
    mobileBadge.invitationManager.invitations.collect { consumable ->
        val invitation = consumable?.getIfNotConsumed()
        if (invitation != null) {
            // Show enrollment dialog to user
            // In your dialog, request biometric authentication if the user chooses to enroll
            // Call enrollmentRepository.register(invitation) to enroll once biometric authentication succeeds
        }
    }
}
```

### 5. Handle Biometric Authentication

The SDK automatically requests biometric authentication when needed. Observe requests and show the prompt:

```kotlin
viewModelScope.launch {
    instance.biometricAuthManager.request.collect { consumable ->
        val request = consumable?.getIfNotConsumed()
        if (request != null) {
            // Show biometric prompt using the extension function
            val result = mobileBadge.biometricAuthManager.authenticate(
                activity = activity,
                title = "Authenticate",
                description = "Verify your identity"
            )
            // Handle result as needed
        }
    }
}
```
The SDK will automatically resume performing the action that required the biometric authentication once authenticated successfully.

### 6. Persist BLE Settings

**Important:** The SDK does NOT persist BLE settings across app restarts. You must save and restore these values:

```kotlin
// Settings to persist:
- rssiCalibrationManager.rssiThreshold
- rssiCalibrationManager.rssiRange
- rssiCalibrationManager.lastCalibrationTimestamp
- bleManager.isBackgroundModeEnabled
- bleManager.isHandsFreeModeEnabled

// Observe each StateFlow and save changes to SharedPreferences or DataStore
// Restore values when creating BleInitialValues during SDK initialization
```

## Core Concepts

### User Enrollment

Users enroll by opening invitation URLs sent by administrators. The SDK supports deep links from:
- `https://*.portal.elatec.com/*`
- `https://*.dev.portal.elatec-integration.com/*`
- `https://*.playground.portal.elatec-integration.com/*`
- `https://*.qa.portal.elatec-integration.com/*`
- `elatec-mb://register`

**Enrollment Flow:**
1. User clicks invitation URL
2. App receives deep link intent
3. Pass URI to `invitationManager.handleDeepLink(uri)`
4. SDK parses invitation and emits it via `invitationManager.invitations`
5. App shows confirmation dialog
6. Call `enrollmentRepository.register(invitation)` to complete enrollment
7. SDK downloads and caches the user's digital pass

### Digital Passes

Digital passes are credentials that authenticate users with Elatec readers. Passes have a lifecycle:

**Pass Validity:**
- Valid for **24 hours** after last successful refresh
- SDK automatically refreshes when:
  - User completes enrollment
  - Biometric authentication succeeds
  - Internet connection is restored after being lost

**Pass States:**
- **Active** - Valid and ready to use
- **NotYetActive** - Start date is in the future
- **Stale** - Needs refresh (24+ hours since last refresh)
- **Expired** - Expiration date has passed
- **Suspended** - Temporarily disabled by organization
- **Revoked** - Permanently revoked, re-enrollment required

Monitor pass state via:
```kotlin
enrollmentRepository.findCurrentPass() // Flow<ElatecPass?>
enrollmentRepository.isEnrolled() // Flow<Boolean>
```

### Biometric Authentication

The SDK triggers biometric authentication at critical moments (pass refresh, first use after restart, etc.):

**Authentication Flow:**
1. SDK determines authentication is needed
2. Emits request via `biometricAuthManager.request`
3. App observes request and shows biometric prompt using `biometricAuthManager.authenticate`
4. On success, SDK automatically continues the task that required the authentication and refreshes the pass.

### BLE Device Discovery

The SDK continuously scans for Elatec BLE readers and tracks them with RSSI (signal strength):

**Device Tracking:**
- Devices discovered within RSSI threshold appear in `deviceManager.discoveredDevices`
- RSSI history maintained (last 20 values)
- Devices marked as stale after **5 seconds** without rediscovery
- Average, latest, nearest, and furthest RSSI values available

**Key StateFlows:**
```kotlin
deviceManager.discoveredDevices // Map of all discovered devices
deviceManager.closestDeviceInRange // Closest device within RSSI threshold
deviceManager.devicesInRange // List of all devices in range
deviceManager.canAuthenticate // Whether authentication is possible
```

### Connection Status

`ConnectionStatusManager` provides a unified view of system state with hierarchical priority:

**Status Priority (highest to lowest):**
1. **Service Availability** - Bluetooth/Location enabled/disabled
2. **Pass State** - Active, Expired, Revoked, Suspended, etc.
3. **Authentication State** - Auth started/succeeded/failed (3-second timeout)
4. **Device Discovery** - Devices discovered, devices in vicinity
5. **Scanning** - Currently scanning

Observe the overall status via:
```kotlin
connectionStatusManager.status // StateFlow<ConnectionStatus>
```

Or observe individual components:
```kotlin
connectionStatusManager.servicesAvailabilityStatus
connectionStatusManager.passStateStatus
connectionStatusManager.authStatus
connectionStatusManager.devicesDiscoveryStatus
connectionStatusManager.scanningStatus
```

### Background Mode & Hands-Free

**Background Mode:**
- Keeps BLE scanning active when app is in background
- Runs as a foreground service with persistent notification
- Requires `BleServiceNotificationFactory` implementation
- Always available for non-enrolled users
- For enrolled users, availability depends on the organization's policy (disabled when biometric authentication is required)

**Hands-Free Mode:**
- Enables automatic authentication when devices are in range
- No user interaction required
- Automatically enabled when background mode is enabled
- Cannot be disabled while background mode is enabled

**Enable/Disable:**
```kotlin
bleManager.setBackgroundModeEnabled(true)
bleManager.setHandsFreeModeEnabled(true)
```

**Check Constraints:**
```kotlin
bleManager.canEnableBackgroundMode // Available if not enrolled or if the organization's policy allows it
bleManager.canDisableHandsFreeMode // Can only disable when background mode is off
```

### RSSI Calibration

RSSI (Received Signal Strength Indicator) determines when devices are "in range". Calibration improves accuracy:

**Calibration Process:**
1. Position phone **30cm (1ft)** from Elatec reader
2. Call `rssiCalibrationManager.calibrate(device)` or `calibrate(rssiValue)`
3. SDK records baseline signal strength
4. Future RSSI readings compared against this baseline

**Connection Lock:**
- SDK creates a calibration lock on first launch
- Prevents automatic authentication until calibration is performed
- Lock released when `wasCalibratedBefore` becomes true

**Configuration:**
Both Rssi Threshold and Rssi Range are updated on calibration, you can also manually set them using:
```kotlin
rssiCalibrationManager.setRssiThreshold(-70) // Devices above this are "in range"
rssiCalibrationManager.setRssiRange(-90..-10) // Valid range for user settings
```

### Connection Locks

Connection locks prevent automatic authentication during specific operations:

**Use Cases:**
- Showing dialogs or bottom sheets
- User is manually selecting which device to authenticate with
- Performing configuration or settings changes
- Any operation where automatic auth would be disruptive

**Creating Locks:**
```kotlin
val lock = connectionLocksManager.lockConnection(
    name = "Settings Dialog",
    foregroundOnly = true // Only active when app is in foreground
)

// Perform your operation

connectionLocksManager.releaseLock(lock)
```

**Observing Locks:**
```kotlin
connectionLocksManager.isLocked // Whether any locks are active
connectionLocksManager.activeLocks // Set of currently active locks
```

## Key Managers

Access all managers via `MobileBadge.getInstance()`:

| Manager                        | Purpose                                                                  |
|--------------------------------|--------------------------------------------------------------------------|
| `bleManager`                   | Start/stop BLE scanning, configure background and hands-free modes       |
| `enrollmentRepository`         | Register users, refresh passes, check enrollment status                  |
| `biometricAuthManager`         | Handle biometric authentication requests and results                     |
| `deviceManager`                | Monitor discovered BLE devices and determine authentication readiness    |
| `invitationManager`            | Parse enrollment invitation deep links                                   |
| `connectionStatusManager`      | Monitor overall system status (Bluetooth, location, pass, auth, devices) |
| `rssiCalibrationManager`       | Configure RSSI thresholds and perform calibration                        |
| `connectionLocksManager`       | Create and manage connection locks to prevent automatic authentication   |
| `bluetoothAvailabilityManager` | Check Bluetooth state and permissions                                    |
| `locationAvailabilityManager`  | Check location service state                                             |
| `credentialIdProvider`         | Resolve credential ID for current user or device                         |

## Important Behaviors

### Device Lifecycle
- Devices rediscovered continuously during scanning
- RSSI history maintains last **20 values**
- Devices marked stale after **5 seconds** without rediscovery
- **5-second cooldown** between authentication attempts to same device

### Pass Refresh Timing
- Passes valid for **24 hours** after last refresh
- Auto-refresh triggers:
  - After successful enrollment
  - After successful biometric authentication
  - When internet restored after being disconnected
- Manual refresh via `enrollmentRepository.refreshPasses()`

### Authentication Timing
- Auth status (success/failure) available for **3 seconds** via `connectionStatusManager.authStatus`
- After timeout, status cleared from hierarchy
- Prevents stale status from affecting UI

### Background Service
- Service survives app termination
- Automatically restarts after device reboot
- Requires notification channel (API 26+)

## Typical Workflow

1. **App Launch**
   - Initialize SDK in `Application.onCreate()`
   - Restore persisted BLE settings via `BleInitialValues`

2. **Permission Request**
   - Check permissions on activity resume
   - Request missing permissions

3. **Start Scanning**
   - Call `bleManager.startIfNotRunning()`
   - Observe scanning state

4. **User Enrollment** (first time)
   - User clicks invitation URL
   - Handle deep link, show confirmation
   - Call `enrollmentRepository.register(invitation)`
   - SDK downloads pass and triggers biometric auth

5. **Authentication**
   - Observe `biometricAuthManager.request`
   - Show biometric prompt when requested
   - SDK auto-refreshes pass on success

6. **Device Detection**
   - Monitor `deviceManager.closestDeviceInRange`
   - Wait for device to enter RSSI threshold
   - SDK automatically authenticates (if hands-free enabled)

7. **Status Updates**
   - Observe `connectionStatusManager.status`
   - Update UI based on current state
   - Handle errors (Bluetooth off, pass expired, etc.)

## Additional Resources

For detailed API documentation, refer to the SDK's KDoc documentation.
