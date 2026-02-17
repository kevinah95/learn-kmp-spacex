## Step 1: Setup Koin

As your KMP SpaceX application grows, managing dependencies manually becomes challenging. Koin, a pragmatic lightweight dependency injection framework, helps organize your code by providing a clean way to manage object creation and dependencies across all platforms in your Kotlin Multiplatform project.

### 📖 Theory: Dependency Injection with Koin

**Dependency Injection (DI)** is a design pattern where objects receive their dependencies from external sources rather than creating them internally. This makes code more modular, testable, and maintainable.

**Koin** is a lightweight DI framework designed specifically for Kotlin that works seamlessly with Kotlin Multiplatform. Unlike other DI frameworks, Koin:
- Uses pure Kotlin DSL (no code generation or reflection in production)
- Provides excellent KMP support out of the box
- Integrates smoothly with Jetpack Compose and ViewModels
- Offers separate modules for platform-specific dependencies

> [!TIP]
> Koin uses a Bill of Materials (BOM) to manage dependency versions consistently across all modules. This ensures compatibility and simplifies version management.

In this step, you'll set up Koin with a modular structure:
- **Data Layer**: Repository and data source dependencies
- **Domain Layer**: Use case and business logic dependencies  
- **Presentation Layer**: ViewModel and UI-related dependencies
- **Platform Module**: Platform-specific implementations using `expect`/`actual`


### ⌨️ Activity: Add Koin to your project

1. Edit Version Catalog to include Koin dependencies.

  ```toml
  // gradle/libs.versions.toml
  [versions]
  koin-bom = "4.1.1"

  [libraries]
  # Koin
  koin-bom = { module = "io.insert-koin:koin-bom", version.ref = "koin-bom" }
  koin-core = { module = "io.insert-koin:koin-core" }
  koin-android = { module = "io.insert-koin:koin-android" }
  koin-compose = { module = "io.insert-koin:koin-compose" }
  koin-compose-viewmodel = { module = "io.insert-koin:koin-compose-viewmodel" }
  koin-compose-viewmodel-navigation = { module = "io.insert-koin:koin-compose-viewmodel-navigation" }
  koin-test = { module = "io.insert-koin:koin-test" }
  ```

  > [!NOTE]  
  > Sync your Gradle project to download the new dependencies.

1. Add Koin dependencies to your project modules.

  ```kotlin
  // composeApp/build.gradle.kts
  sourceSets {
      commonMain.dependencies {
          // ... other dependencies
          // Koin
          implementation(project.dependencies.platform(libs.koin.bom))
          implementation(libs.koin.compose)
          implementation(libs.koin.compose.viewmodel)
          implementation(libs.koin.compose.viewmodel.navigation)
      }
  }
  ```

  ```kotlin
  // androidApp/build.gradle.kts
  kotlin {
    dependencies {
        // ... other dependencies
        // Koin
        implementation(project.dependencies.platform(libs.koin.bom))
        implementation(libs.koin.android)
    }
  }
  ```

  ```kotlin
  // shared/build.gradle.kts
  kotlin {
    sourceSets {
        commonMain.dependencies {
            // ... other dependencies
            // Koin
            implementation(project.dependencies.platform(libs.koin.bom))
            implementation(libs.koin.core)
            implementation(libs.koin.compose.viewmodel)
        }
        commonTest.dependencies {
          // ... other dependencies
          implementation(libs.koin.test)
        }
    }
  }
  ```

1. Create placeholders for Koin modules in your project.

  ```kotlin
  // shared/src/commonMain/kotlin/di/modules/DataModule.kt
  val dataModule = module {
      // Define your data layer dependencies here
  }
  ```

  ```kotlin
  // shared/src/commonMain/kotlin/di/modules/DomainModule.kt
  val domainModule = module {
      // Define your domain layer dependencies here
  }
  ```

  ```kotlin
  // shared/src/commonMain/kotlin/di/modules/PresentationModule.kt
  val presentationModule = module {
      // Define your presentation layer dependencies here
  }
  ```

  ```kotlin
  // shared/src/commonMain/kotlin/di/modules/NetworkModule.kt
  val networkModule = module {
      // Define your network-related dependencies here
  }
  ```

1. The special case is the platform module, which will be extended by each platform to include platform-specific dependencies.
  ```kotlin
  // shared/src/commonMain/kotlin/di/modules/PlatformModule.kt
  expect fun platformModule(): Module
  ```

  ```kotlin
  // shared/src/androidMain/kotlin/di/modules/PlatformModule.android.kt
  actual fun platformModule(): Module = module {
      // Define your Android-specific dependencies here
  }
  ```

  ```kotlin
  // shared/src/iosMain/kotlin/di/modules/PlatformModule.ios.kt
  actual fun platformModule(): Module = module {
      // Define your iOS-specific dependencies here
  }
  ```

1. Create a Shared Module to include all Koin modules.

  ```kotlin
  // shared/src/commonMain/kotlin/di/modules/SharedModule.kt
  val sharedModule = module {
      // Define shared dependencies here
      // special case for platform-specific dependencies calling as a function.
      includes(dataModule, domainModule, presentationModule, networkModule, platformModule())
  }
  ```

1. Create a Helper function to initialize Koin in your application.

  ```kotlin
  // shared/src/commonMain/kotlin/di/KoinHelper.kt
  fun initKoin(config: KoinAppDeclaration? = null): KoinApplication {
      return startKoin {
          includes(config)  // Platform-specific extensions
          modules(sharedModule)
      }
  }
  ```

1. Initialize Koin in your Android application.

  ```kotlin
  // androidApp/src/main/kotlin/compose/project/demo/composedemo/MainApplication.kt
  class MainApplication : Application() {

    override fun onCreate() {
      super.onCreate()

      initKoin {
        androidContext(this@MainApplication)
        androidLogger()
      }
    }
  }
  ```

  ```xml
  <!-- androidApp/src/main/AndroidManifest.xml -->
  <!-- ... -->
  <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
  +      android:name=".MainApplication" <!-- Add this line -->
        android:theme="@android:style/Theme.Material.Light.NoActionBar">
  <!-- ... -->
  ```

1. Initialize Koin in your iOS application.

  ```diff
  // composeApp/src/iosMain/kotlin/compose/project/demo/composedemo/MainViewController.kt
  -  fun MainViewController() = ComposeUIViewController { App() }
  +  fun MainViewController() = ComposeUIViewController(configure = { initKoin() }) { App() }
  ```

<details>
<summary>Having trouble? 🤷</summary><br/>

- **Gradle sync fails**: Make sure you've saved the `libs.versions.toml` file and clicked "Sync Now" in Android Studio. If issues persist, try invalidating caches (File → Invalidate Caches → Invalidate and Restart).
- **Module not found errors**: Verify that you're creating files in the correct source sets (`commonMain`, `androidMain`, `iosMain`). The path structure matters in KMP projects.
- **Import errors for Koin**: Ensure all three modules (composeApp, androidApp, shared) have the Koin dependencies added. The BOM needs to be included in each module that uses Koin dependencies.
- **expect/actual mismatch**: Make sure the `platformModule()` function signature matches exactly in the expect declaration and both actual implementations (Android and iOS).

</details>
