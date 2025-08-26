---
title: 🎁 Presentation
sidebar_position: 5
---

The presentation phase allows a user to present a credential (such as an mDL) to a verifier, typically using BLE, NFC, or QR code. This section covers runtime permissions, setting up presentment flows, and generating engagement QR codes.

### Runtime Permissions

Multipaz provides composable functions for requesting runtime permissions in your app. Typical permissions include Bluetooth, Camera, and Notifications.

* **Bluetooth Permission:** Use `rememberBluetoothPermissionState`
* **Camera Permission:** Use `rememberCameraPermissionState`
* **Notification Permission:** Use `rememberNotificationPermissionState`

**Example: Requesting BLE Permission**

```kotlin
val blePermissionState = rememberBluetoothPermissionState()

if (!blePermissionState.isGranted) {
   Button(
       onClick = {
           coroutineScope.launch {
               blePermissionState.launchPermissionRequest()
           }
       }
   ) {
       Text("Request BLE permissions")
   }
```

**AndroidManifest.xml: Required BLE Permissions**

```xml
<!-- For BLE -->
<uses-feature
   android:name="android.hardware.bluetooth_le"
   android:required="true" />
<uses-permission
   android:name="android.permission.BLUETOOTH_SCAN"
   android:usesPermissionFlags="neverForLocation"
   tools:targetApi="s" />
<uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<!-- Request legacy Bluetooth permissions on older devices. -->
<uses-permission
   android:name="android.permission.BLUETOOTH"
   android:maxSdkVersion="30" />
<uses-permission
   android:name="android.permission.BLUETOOTH_ADMIN"
   android:maxSdkVersion="30" />
<uses-permission
   android:name="android.permission.ACCESS_COARSE_LOCATION"
   android:maxSdkVersion="30" />
<uses-permission
   android:name="android.permission.ACCESS_FINE_LOCATION"
   android:maxSdkVersion="30" />
```

Refer to [this](https://github.com/openmobilehub/multipaz-getting-started-sample/blob/7500a92ead53cdeca3c6131000c3f7ec07284349/composeApp/src/commonMain/kotlin/org/multipaz/get_started/App.kt#L186-L196) part for the implementation of the permissions section of this guide.

### PresentmentModel

`PresentmentModel` manages the entire UX/UI flow for credential presentation, providing a `state` variable to track the presentation process. Multipaz also offers a `Presentment` composable for embedding credential presentment UI.

You can generate QR codes using `org.multipaz.compose.qrcode:generateQrCode`.

```kotlin
lateinit var presentmentModel: PresentmentModel
lateinit var presentmentSource: PresentmentSource
// . . .
presentmentModel = PresentmentModel().apply { setPromptModel(promptModel) }
presentmentSource = SimplePresentmentSource(
   documentStore = documentStore,
   documentTypeRepository = documentTypeRepository,
   readerTrustManager = readerTrustManager,
   preferSignatureToKeyAgreement = true,
   domainMdocSignature = "mdoc",
)

val deviceEngagement = remember { mutableStateOf<ByteString?>(null) }
val state = presentmentModel.state.collectAsState()
when (state.value) {
   PresentmentModel.State.IDLE -> {
       showQrButton(deviceEngagement)
   }

   PresentmentModel.State.CONNECTING -> {
       showQrCode(deviceEngagement)
   }

   PresentmentModel.State.WAITING_FOR_SOURCE,
   PresentmentModel.State.PROCESSING,
   PresentmentModel.State.WAITING_FOR_DOCUMENT_SELECTION,
   PresentmentModel.State.WAITING_FOR_CONSENT,
   PresentmentModel.State.COMPLETED -> {
       Presentment(
           appName = "Multipaz Getting Started Sample",
           appIconPainter = painterResource(Res.drawable.compose_multiplatform),
           presentmentModel = presentmentModel,
           presentmentSource = presentmentSource,
           documentTypeRepository = documentTypeRepository,
           onPresentmentComplete = {
               presentmentModel.reset()
           },
       )
   }
}
```

### Starting Device Engagement

To start engagement for presentment (e.g., via BLE), use a connection method that extends `MdocConnectionMethod` (such as `MdocConnectionMethodBle` or `MdocConnectionMethodNfc`). The following example uses BLE:

**Example: BLE Engagement and QR Code**

```kotlin
@Composable
private fun showQrButton(showQrCode: MutableState<ByteString?>) {
   Column(
       modifier = Modifier.fillMaxSize(),
       verticalArrangement = Arrangement.Center,
       horizontalAlignment = Alignment.CenterHorizontally
   ) {
       Button(onClick = {
           presentmentModel.reset()
           presentmentModel.setConnecting()
           presentmentModel.presentmentScope.launch() {
               val connectionMethods = listOf(
                   MdocConnectionMethodBle(
                       supportsPeripheralServerMode = false,
                       supportsCentralClientMode = true,
                       peripheralServerModeUuid = null,
                       centralClientModeUuid = UUID.randomUUID(),
                   )
               )
               val eDeviceKey = Crypto.createEcPrivateKey(EcCurve.P256)
               val advertisedTransports = connectionMethods.advertise(
                   role = MdocRole.MDOC,
                   transportFactory = MdocTransportFactory.Default,
                   options = MdocTransportOptions(bleUseL2CAP = true),
               )
               val engagementGenerator = EngagementGenerator(
                   eSenderKey = eDeviceKey.publicKey,
                   version = "1.0"
               )
               engagementGenerator.addConnectionMethods(advertisedTransports.map {
                   it.connectionMethod
               })
               val encodedDeviceEngagement = ByteString(engagementGenerator.generate())
               showQrCode.value = encodedDeviceEngagement
               val transport = advertisedTransports.waitForConnection(
                   eSenderKey = eDeviceKey.publicKey,
                   coroutineScope = presentmentModel.presentmentScope
               )
               presentmentModel.setMechanism(
                   MdocPresentmentMechanism(
                       transport = transport,
                       eDeviceKey = eDeviceKey,
                       encodedDeviceEngagement = encodedDeviceEngagement,
                       handover = Simple.NULL,
                       engagementDuration = null,
                       allowMultipleRequests = false
                   )
               )
               showQrCode.value = null
           }
       }) {
           Text("Present mDL via QR")
       }
   }
}
```

Refer to [this](https://github.com/openmobilehub/multipaz-getting-started-sample/blob/7500a92ead53cdeca3c6131000c3f7ec07284349/composeApp/src/commonMain/kotlin/org/multipaz/get_started/App.kt#L230-L284) part for the implementation of this section in this guide.

### Displaying the QR Code

Use the following composable to display the QR code generated for presentment.

**Example: QR Code Display**

```kotlin
@Composable
private fun showQrCode(deviceEngagement: MutableState<ByteString?>) {
   Column(
       modifier = Modifier.fillMaxSize().padding(16.dp),
       verticalArrangement = Arrangement.Center,
       horizontalAlignment = Alignment.CenterHorizontally,
   ) {
       if (deviceEngagement.value != null) {
           val mdocUrl = "mdoc:" + deviceEngagement.value!!.toByteArray().toBase64Url()
           val qrCodeBitmap = remember { generateQrCode(mdocUrl) }
           Text(text = "Present QR code to mdoc reader")
           Image(
               modifier = Modifier.fillMaxWidth(),
               bitmap = qrCodeBitmap,
               contentDescription = null,
               contentScale = ContentScale.FillWidth
           )
           Button(
               onClick = {
                   presentmentModel.reset()
               }
           ) {
               Text("Cancel")
           }
       }
   }
}
```

Refer to [this](https://github.com/openmobilehub/multipaz-getting-started-sample/blob/7500a92ead53cdeca3c6131000c3f7ec07284349/composeApp/src/commonMain/kotlin/org/multipaz/get_started/App.kt#L286-L312) part for the implementation of this section in this guide.

### NFC Engagement (Android Only)

NFC (Near Field Communication) provides a contactless mechanism for credential sharing, allowing users to simply "tap" their phone to a verifier device to present credentials. This is particularly useful for Android devices, offering fast and secure sharing without requiring manual UI interaction.

The NFC implementation consists of three main components:

* **`NfcActivity`** - Handles the credential presentation lifecycle triggered by NFC tap
* **`NdefService`** - System-level service that binds the NFC engagement mechanism  
* **AndroidManifest.xml** - Declares NFC capabilities and configures the app's NFC role

#### NFC Permissions and Requirements

First, add the required NFC permissions to your `AndroidManifest.xml`:

```xml
<!-- NFC permissions -->
<uses-permission android:name="android.permission.NFC" />
<uses-feature
    android:name="android.hardware.nfc"
    android:required="true" />
```

#### NfcActivity Implementation

Create an `NfcActivity` that extends `MdocNfcPresentmentActivity` for ISO/IEC 18013-5:2021 presentment when using NFC engagement:

**NfcActivity.kt**

```kotlin
import org.multipaz.android.mdoc.presentment.MdocNfcPresentmentActivity
import org.multipaz.android.mdoc.presentment.Settings
import androidx.compose.runtime.Composable

class NfcActivity : MdocNfcPresentmentActivity() {
    
    override fun ApplicationTheme(content: @Composable (() -> Unit)) {
        content()
    }

    override suspend fun getSettings(): Settings {
        val app = App.getInstance()
        app.init()
        return Settings(
            appName = app.appName,
            appIcon = app.appIcon,
            promptModel = App.promptModel,
            documentTypeRepository = app.documentTypeRepository,
            presentmentSource = app.presentmentSource
        )
    }
}
```

This activity launches when the device is tapped against a verifier. It initializes the SDK and returns the appropriate settings, including app name, app icon, prompt model, and presentment source.

#### NdefService Implementation

Create an `NdefService` that extends `MdocNdefService` for NFC engagement according to ISO/IEC 18013-5:2021:

**NdefService.kt**

```kotlin
import org.multipaz.android.mdoc.nfc.MdocNdefService
import org.multipaz.android.mdoc.nfc.Settings
import org.multipaz.android.mdoc.transport.MdocTransportOptions
import org.multipaz.crypto.EcCurve

class NdefService : MdocNdefService() {
    override suspend fun getSettings(): Settings {
        return Settings(
            sessionEncryptionCurve = EcCurve.P256,
            allowMultipleRequests = false,
            useNegotiatedHandover = true,
            negotiatedHandoverPreferredOrder = listOf(
                "ble:central_client_mode:",
                "ble:peripheral_server_mode:"
            ),
            transportOptions = MdocTransportOptions(bleUseL2CAP = true),
            promptModel = App.promptModel,
            presentmentActivityClass = NfcActivity::class.java
        )
    }
}
```

The `negotiatedHandoverPreferredOrder` is set to select BLE. In this configuration, NFC establishes the initial connection but no credential data is transferred at this stage. The NFC connection is used to negotiate which transport method to use, and since BLE is selected, a BLE connection is established for actual credential sharing.

#### AndroidManifest.xml Configuration

Add the NFC activity and service declarations to your `AndroidManifest.xml`:

```xml
<!-- NFC Activity for credential presentation -->
<activity
    android:name=".NfcActivity"
    android:showWhenLocked="true"
    android:turnScreenOn="true"
    android:exported="true"
    android:launchMode="singleInstance"
    android:theme="@android:style/Theme.Translucent.NoTitleBar.Fullscreen" />

<!-- NFC Service for APDU communication -->
<service
    android:name=".NdefService"
    android:exported="true"
    android:permission="android.permission.BIND_NFC_SERVICE">
    <intent-filter>
        <action android:name="android.nfc.cardemulation.action.HOST_APDU_SERVICE" />
    </intent-filter>
    <meta-data
        android:name="android.nfc.cardemulation.host_apdu_service"
        android:resource="@xml/nfc_ndef_service" />
</service>
```

#### NFC AID Filter Configuration

Create a file `res/xml/nfc_ndef_service.xml` to configure the AID (Application Identifier) filter. This allows your Android device to act as an NFC Type 4 Tag and share credentials securely with a verifier:

**res/xml/nfc_ndef_service.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<host-apdu-service xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:description="@string/nfc_ndef_service_description"
    android:requireDeviceUnlock="false"
    android:requireDeviceScreenOn="false"
    tools:ignore="UnusedAttribute">

    <aid-group android:description="@string/nfc_ndef_service_aid_group_description"
        android:category="other">
        <!-- NFC Type 4 Tag - matches ISO 18013-5 mDL standard -->
        <aid-filter android:name="D2760000850101"/>
    </aid-group>
</host-apdu-service>
```

**Configuration Attributes:**
- `android:requireDeviceUnlock`: `false` — app can respond even when device is locked
- `android:requireDeviceScreenOn`: `false` — screen can be off during NFC interaction
- `aid-filter`: Identifies the NFC Type 4 Tag according to ISO 18013-5 mDL standard

#### String Resources

Add the required string resources to your `res/values/strings.xml`:

```xml
<string name="nfc_ndef_service_description">Multipaz NFC Service</string>
<string name="nfc_ndef_service_aid_group_description">Multipaz mDL NFC</string>
```

#### NFC with Connection Methods

To support NFC alongside other connection methods, include `MdocConnectionMethodNfc` in your connection methods list:

```kotlin
val connectionMethods = listOf(
    MdocConnectionMethodBle(
        supportsPeripheralServerMode = false,
        supportsCentralClientMode = true,
        peripheralServerModeUuid = null,
        centralClientModeUuid = UUID.randomUUID(),
    ),
    MdocConnectionMethodNfc() // Add NFC support
)
```

#### Error Handling and Timeouts

When implementing NFC functionality, consider these error scenarios:

**NFC Not Available:**
```kotlin
private fun isNfcAvailable(): Boolean {
    val nfcManager = getSystemService(Context.NFC_SERVICE) as NfcManager
    return nfcManager.defaultAdapter?.isEnabled == true
}

// Check before attempting NFC operations
if (!isNfcAvailable()) {
    // Show user guidance to enable NFC
    // Fallback to QR code or BLE presentation
}
```

**NFC Timeout Handling:**
```kotlin
// Configure timeout settings in NdefService
override suspend fun getSettings(): Settings {
    return Settings(
        // ... other settings
        sessionEncryptionCurve = EcCurve.P256,
        allowMultipleRequests = false,
        // Implement timeout logic in your presentation flow
    )
}
```

#### Testing NFC Functionality

To test NFC credential sharing:

1. **Enable NFC** on both holder and verifier devices
2. **Install your app** with the NFC configuration on the holder device
3. **Use a compatible verifier** (such as Multipaz Identity Reader)
4. **Tap devices together** - the holder device should automatically wake and present the credential selection UI
5. **Verify credential transfer** - check that credentials are successfully shared via the negotiated transport (typically BLE)

**Testing Checklist:**
- [ ] NFC is enabled on the device
- [ ] App has NFC permissions
- [ ] Device wakes up when tapped to verifier
- [ ] Credential selection UI appears
- [ ] Credentials are successfully shared
- [ ] Error handling works for NFC unavailable scenarios

#### Verifying Credential Authenticity

The NFC implementation automatically handles credential verification through the Multipaz SDK:

- **Device Authentication**: Ensures the credential comes from a legitimate device
- **Issuer Authentication**: Verifies the credential was issued by a trusted authority  
- **Certificate Chain Validation**: Checks the complete trust chain
- **Temporal Validity**: Confirms the credential is currently valid

These verifications are handled automatically by the `presentmentSource` and related trust managers configured in your app.

By following these steps, you can implement comprehensive NFC support for credential sharing in your Multipaz holder application, providing users with a seamless "tap to share" experience while maintaining security and compliance with ISO/IEC 18013-5:2021 standards.
