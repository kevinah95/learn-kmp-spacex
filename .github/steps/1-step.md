## Paso 1: Configurar Koin

A medida que tu aplicación KMP de SpaceX crece, manejar dependencias manualmente se vuelve un reto. Koin, un framework pragmático y liviano de inyección de dependencias, te ayuda a organizar tu código proporcionando una forma limpia de gestionar la creación de objetos y dependencias en todas las plataformas de tu proyecto Kotlin Multiplatform.

### 📖 Teoría: Inyección de Dependencias con Koin

**Inyección de Dependencias (DI)** es un patrón de diseño donde los objetos reciben sus dependencias de fuentes externas en lugar de crearlas internamente. Esto hace que el código sea más modular, testeable y mantenible.

**Koin** es un framework de DI liviano diseñado específicamente para Kotlin que funciona perfectamente con Kotlin Multiplatform. A diferencia de otros frameworks de DI, Koin:
- Usa DSL puro de Kotlin (sin generación de código ni reflexión en producción)
- Proporciona excelente soporte KMP desde el principio
- Se integra suavemente con Jetpack Compose y ViewModels
- Ofrece módulos separados para dependencias específicas de cada plataforma

> [!TIP]
> Koin usa un Bill of Materials (BOM) para gestionar versiones de dependencias consistentemente en todos los módulos. Esto asegura compatibilidad y simplifica la gestión de versiones.

En este paso, configurarás Koin con una estructura modular:
- **Capa de Datos**: Dependencias de repositorios y fuentes de datos
- **Capa de Dominio**: Dependencias de casos de uso y lógica de negocio  
- **Capa de Presentación**: Dependencias relacionadas con ViewModel y UI
- **Módulo de Plataforma**: Implementaciones específicas de plataforma usando `expect`/`actual`


### ⌨️ Actividad: Agrega Koin a tu proyecto

1. Edita el Catálogo de Versiones para incluir las dependencias de Koin.

  ```toml
  # gradle/libs.versions.toml
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
  > Sincroniza tu proyecto de Gradle para descargar las nuevas dependencias.

1. Agrega las dependencias de Koin a los módulos de tu proyecto.

  ```kotlin
  // composeApp/build.gradle.kts
  sourceSets {
      commonMain.dependencies {
          // ... otras dependencias
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
        // ... otras dependencias
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
            // ... otras dependencias
            // Koin
            implementation(project.dependencies.platform(libs.koin.bom))
            implementation(libs.koin.core)
            implementation(libs.koin.compose.viewmodel)
        }
        commonTest.dependencies {
          // ... otras dependencias
          implementation(libs.koin.test)
        }
    }
  }
  ```

1. Crea marcadores de posición para los módulos de Koin en tu proyecto.

  ```kotlin
  // shared/src/commonMain/kotlin/di/modules/DataModule.kt
  val dataModule = module {
      // Define aquí las dependencias de tu capa de datos
  }
  ```

  ```kotlin
  // shared/src/commonMain/kotlin/di/modules/DomainModule.kt
  val domainModule = module {
      // Define aquí las dependencias de tu capa de dominio
  }
  ```

  ```kotlin

  // shared/src/commonMain/kotlin/di/modules/PresentationModule.kt
  val presentationModule = module {
      // Define aquí las dependencias de tu capa de presentación
  }
  ```

  ```kotlin
  // shared/src/commonMain/kotlin/di/modules/NetworkModule.kt
  val networkModule = module {
      // Define aquí tus dependencias relacionadas con red
  }
  ```

1. El caso especial es el módulo de plataforma, que será extendido por cada plataforma para incluir dependencias específicas.
  ```kotlin
  // shared/src/commonMain/kotlin/di/modules/PlatformModule.kt
  expect fun platformModule(): Module
  ```

  ```kotlin
  // shared/src/androidMain/kotlin/di/modules/PlatformModule.android.kt
  actual fun platformModule(): Module = module {
      // Define aquí tus dependencias específicas de Android
  }
  ```

  ```kotlin
  // shared/src/iosMain/kotlin/di/modules/PlatformModule.ios.kt
  actual fun platformModule(): Module = module {
      // Define aquí tus dependencias específicas de iOS
  }
  ```

1. Crea un Módulo Compartido para incluir todos los módulos de Koin.

  ```kotlin
  // shared/src/commonMain/kotlin/di/modules/SharedModule.kt
  val sharedModule = module {
      // Define aquí las dependencias compartidas
      // caso especial para dependencias específicas de plataforma llamándolo como función
      includes(dataModule, domainModule, presentationModule, networkModule, platformModule())
  }
  ```

1. Crea una función auxiliar para inicializar Koin en tu aplicación.

  ```kotlin
  // shared/src/commonMain/kotlin/di/KoinHelper.kt
  fun initKoin(config: KoinAppDeclaration? = null): KoinApplication {
      return startKoin {
          includes(config)  // Extensiones específicas de plataforma
          modules(sharedModule)
      }
  }
  ```

1. Inicializa Koin en tu aplicación Android.

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

  ```diff
  <!-- androidApp/src/main/AndroidManifest.xml -->
  <!-- ... -->
  <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
  +      android:name=".MainApplication" <!-- Agrega esta línea -->
        android:theme="@android:style/Theme.Material.Light.NoActionBar">
  <!-- ... -->
  ```

1. Inicializa Koin en tu aplicación iOS.

  ```diff
  // composeApp/src/iosMain/kotlin/compose/project/demo/composedemo/MainViewController.kt
  -  fun MainViewController() = ComposeUIViewController { App() }
  +  fun MainViewController() = ComposeUIViewController(configure = { initKoin() }) { App() }
  ```

<details>
<summary>¿Tenés problemas? 🤷</summary><br/>

- **Falla la sincronización de Gradle**: Asegurate de haber guardado el archivo `libs.versions.toml` y hacer clic en "Sync Now" en Android Studio. Si los problemas persisten, intentá invalidar el caché (File → Invalidate Caches → Invalidate and Restart).
- **Errores de módulo no encontrado**: Verificá que estés creando archivos en los source sets correctos (`commonMain`, `androidMain`, `iosMain`). La estructura de carpetas importa en proyectos KMP.
- **Errores de importación de Koin**: Asegurate de que los tres módulos (composeApp, androidApp, shared) tengan las dependencias de Koin agregadas. El BOM debe incluirse en cada módulo que use dependencias de Koin.
- **Desajuste expect/actual**: Asegurate de que la firma de la función `platformModule()` coincida exactamente en la declaración expect y ambas implementaciones actual (Android e iOS).

</details>
