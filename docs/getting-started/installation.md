---
title: Installation
sidebar_position: 1
---


## Prerequisites

The steps below assume you’ve already set up a **Kotlin Multiplatform (KMP)** project. Multipaz supports both Android and iOS targets, and these instructions focus on configuring KMP to share business logic across platforms.

> 💡 You can quickly create a KMP project using the official [JetBrains wizard](https://kmp.jetbrains.com/?android=true&ios=true&iosui=compose&includeTests=true)

## Installation of Dependencies[​](https://openmobilehub.github.io/developer-multipaz-website/overview/getting-started/#installation-of-dependencies)

To get started with Multipaz, you need to add the necessary dependencies to your project. This guide assumes you are using Gradle as your build system.

* Add the google repository to `settings.gradle.kts` file

```kotlin
pluginManagement {
   repositories {
       // ... other repositories
       google()
   }
}

dependencyResolutionManagement {
   repositories {
       // ... other repositories
       google()
   }
}
```

* Add the following dependencies to `libs.versions.toml`

```yml
[versions]
# update this line
android-minSdk = "26" # Multipaz requires minSdk >= 26 due to usage of Android 8.0+ APIs

multipaz = "0.93.0" # latest version of Multipaz to use

androidx-fragment = "1.8.6"

[libraries]
multipaz = { group = "org.multipaz", name = "multipaz", version.ref = "multipaz" }
multipaz-models = { group = "org.multipaz", name = "multipaz-models", version.ref = "multipaz" }
multipaz-compose = { group = "org.multipaz", name = "multipaz-compose", version.ref = "multipaz" }
multipaz-doctypes = { group = "org.multipaz", name = "multipaz-doctypes", version.ref = "multipaz" }

androidx-fragment = { group = "androidx.fragment", name = "fragment", version.ref = "androidx-fragment" }
```

* Add the following to your module level `build.gradle.kts` file (usually `app/build.gradle.kts`):

```kotlin
kotlin {
   sourceSets {
       androidMain.dependencies {
           // ... other dependencies
           implementation(libs.androidx.fragment)
       }
       commonMain.dependencies {
           // ... other dependencies
           implementation(libs.multipaz)
           implementation(libs.multipaz.models)
           implementation(libs.multipaz.compose)
           implementation(libs.multipaz.doctypes)
       }
   }
}
```

You might also want to check out other libraries in the Multipaz ecosystem, from Multipaz [here](https://mvnrepository.com/search?q=multipaz).

### Initialize `App.kt`

App Class is the main class that holds all the core logic and state for the app.

We are splitting `App.kt` into multiple sections for ease of use wit multiple Multipaz components.

- **Properties**: Variables for storage, document management, trust management, and presentment.
- **Initialization**: Sets up storage, document types, creates a sample document, and configures trusted certificates.
    - `suspend fun init()`
- **UI**: A Composable function that builds the app’s user interface using Jepack Compose components. It shows initialization status, handles permissions, and displays buttons or QR codes based on the app state.
    - `@Composable fun Content()`
- **Companion Object**: Provides a singleton instance of App and holds shared models.
    - `fun getInstance(): App`

* To support secure prompts such as **biometric authentication**, **passphrases**, and **NFC dialogs** in a consistent and platform-specific way, we now initialize `PromptDialogs` by passing a `PromptModel`.
* Multipaz provides a pre-initialized `promptModel` object that can be imported from `org.multipaz.util.Platform.promptModel`.

```kotlin
// commonMain/App.kt
class App {

    val appName = "Multipaz Getting Started Sample"
    val appIcon = Res.drawable.compose_multiplatform

    var isAppInitialized = false

    suspend fun init() {
        if (!isAppInitialized) {
            isAppInitialized = true
        }
    }

    @Composable
    @Preview
    fun Content() {

        val isUIInitialized = remember { mutableStateOf(false) }

        if (!isUIInitialized.value) {
            CoroutineScope(Dispatchers.Main).launch {
                init()
                isUIInitialized.value = true
            }

            Column(
                modifier = Modifier.fillMaxSize(),
                verticalArrangement = Arrangement.Center,
                horizontalAlignment = Alignment.CenterHorizontally
            ) {
                Text(text = "Initializing...")
            }
            return
        }

        MaterialTheme {
            // This ensures all prompts inherit the app's main style
            PromptDialogs(promptModel)

            Column(
                modifier = Modifier.fillMaxSize(),
                verticalArrangement = Arrangement.Center,
                horizontalAlignment = Alignment.CenterHorizontally
            ) {
                Text(text = "Welcome to Multipaz Getting Started Sample")
                // ... rest of your UI
            }
        }
    }

    companion object {
        val promptModel = org.multipaz.util.Platform.promptModel

        private var app: App? = null
        fun getInstance(): App {
            if (app == null) {
                app = App()
            }
            return app!!
        }
    }
}
```

### Update `MainActivity.kt`

Update `MainActivity` to reflect the changes from `App.kt`, along with the following additions for the Multipaz library.

* Inside the `onCreate()` method in `kotlin/MainActivity` class, call the `initializeApplication(applicationContext)` function provided by the Multipaz library.
    * This ensures the SDK has access to a valid application-level context, which is required for internal operations like secure storage and credential handling. Make sure this is done only once in the app lifecycle, ideally during app startup.
* Modify update `MainActivity` to extend `FragmentActivity`.
    * Multipaz's `PromptDialogs` require the activity to be a `FragmentActivity` to support the `BiometricPrompt` and other platform features.

```kotlin
// kotlin/MainActivity.kt
// IMPORTANT: Multipaz's PromptDialogs require the activity to be a FragmentActivity
// to support the BiometricPrompt and other platform features.
class MainActivity : FragmentActivity() { // use FragmentActivity
    override fun onCreate(savedInstanceState: Bundle?) {
        enableEdgeToEdge()
        super.onCreate(savedInstanceState)
        initializeApplication(this.applicationContext) // initialize Multipaz

        lifecycle.coroutineScope.launch {
            val app = App.getInstance()
            app.init()
            setContent {
                app.Content()
            }
        }
    }
}
```

### Update `iOSMain/MainViewController.kt`

Update `MainViewController` to reflect the changes from `App.kt`.

```kotlin
private val app = App.getInstance()

fun MainViewController() = ComposeUIViewController {
    app.Content()
}
```

<!-- TODO: code to be updated -->
> 📌 Refer to [this](https://github.com/openmobilehub/multipaz-getting-started-sample/commit/f467118149b55080edd2e4f8606a7cd7ad82c2cb) commit for a sample project initialization setup.

#### ⚠️ Some gotchas to be aware of (iOS only):

For iOS, there are these required fixes:

1. In `iosApp/iosApp.xcodeproj/project.pbxproj`

Add the following flags to the `buildSettings` of each `XCBuildConfiguration` under the `iosApp` target in your `project.pbxproj` file:

```C
OTHER_LDFLAGS = (
   "$(inherited)",
   "-lsqlite3",
);
```

Refer to [this](https://github.com/openmobilehub/multipaz-getting-started-sample/commit/6fd2be0b1e039903c837f3d8894aca54bc7d6adf) commit for the changes to be done for the iOS builds.

