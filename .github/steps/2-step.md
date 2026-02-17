## Step 2: Setup Ktor and Coroutines

(replace-me: OPTIONAL Brief story or scenario to introduce the step)

### 📖 Theory: (replace-me: Theory title)

<!-- GitHub-styled notifications can be used outside of ordered lists. Available options are: NOTE, IMPORTANT, WARNING, TIP, CAUTION -->
<!--
> [!NOTE]
> (Important note or additional information relevant to this section)
 -->

(replace-me: Optional theory or background information relevant to this step)

(replace-me: OPTIONAL Reference images from the `.github/images/` directory to support any part of the content)

<img width="200" alt="descriptive alt text" src="../images/inflatocat.png" />


### ⌨️ Activity: (replace-me: Activity title)

1. Add Ktor and Coroutines dependencies to your project.

  ```toml
  # gradle/libs.versions.toml
  [versions]
  ktor = "3.4.0"
  kotlinx-coroutines = "1.10.2"
  dateTime = "0.7.1"

  [libraries]
  # Coroutine
  kotlinx-coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "kotlinx-coroutines" }
  kotlinx-coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "kotlinx-coroutines" }
  kotlinx-coroutines-test = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-test", version.ref = "kotlinx-coroutines" }

  # DateTime
  kotlinx-datetime = { module = "org.jetbrains.kotlinx:kotlinx-datetime", version.ref = "dateTime" }
  
  # Ktor
  ktor-bom = { module = "io.ktor:ktor-bom", version.ref = "ktor" }
  ktor-client-core = { module = "io.ktor:ktor-client-core" }
  ktor-client-okhttp = { module = "io.ktor:ktor-client-okhttp" }
  ktor-client-darwin = { module = "io.ktor:ktor-client-darwin" }
  # Ktor serialization
  ktor-client-content-negotiation = { module = "io.ktor:ktor-client-content-negotiation" }
  ktor-serialization-kotlinx-json = { module = "io.ktor:ktor-serialization-kotlinx-json" }
  ```

1. Add Ktor and Coroutines dependencies to your project modules.

  ```kotlin
  // shared/build.gradle.kts
  kotlin {
    sourceSets {
        androidMain.dependencies {
          implementation(libs.ktor.client.okhttp)
        }
        commonMain.dependencies {
          // ... other dependencies
          // Coroutine
          implementation(libs.kotlinx.coroutines.core)
          
          // DateTime
          implementation(libs.kotlinx.datetime)
          
          // Ktor
          implementation(project.dependencies.platform(libs.ktor.bom))
          implementation(libs.ktor.client.core)
          implementation(libs.ktor.client.content.negotiation)
          implementation(libs.ktor.serialization.kotlinx.json)
        }

        commonTest.dependencies {
          // ... other dependencies
          // Coroutine
          implementation(libs.kotlinx.coroutines.test)
        }

        iosMain.dependencies {
          implementation(libs.ktor.client.darwin)
        }
    }
  }
  ```

  ```kotlin
  // androidApp/build.gradle.kts
  kotlin {
    dependencies {
      // ... other dependencies
      // Coroutine
      implementation(libs.kotlinx.coroutines.android)
    }
  }
  ```

1. Add Ktor client dependency to your NetworkModule.

  ```kotlin
  // shared/src/commonMain/kotlin/compose/project/demo/composedemo/di/modules/NetworkModule.kt
  val networkModule = module {
    single {
      HttpClient {
        install(ContentNegotiation) {
          json(
              Json {
                ignoreUnknownKeys = true
                useAlternativeNames = false
              }
          )
        }
      }
    }
  }
  ```

1. Create a new file to use Ktor client in your data layer.

  ```kotlin
  // shared/src/commonMain/kotlin/compose/project/demo/composedemo/domain/entity/Entity.kt
  data class RocketLaunch(
      @SerialName("flight_number") val flightNumber: Int,
      @SerialName("name") val missionName: String,
      @SerialName("date_utc") val launchDateUTC: String,
      @SerialName("details") val details: String?,
      @SerialName("success") val launchSuccess: Boolean?,
      @SerialName("links") val links: Links,
  ) {
    var launchYear = 2025 // TODO: Default value, should be parsed from launchDateUTC
  }

  @Serializable
  data class Links(
      @SerialName("patch") val patch: Patch?,
      @SerialName("article") val article: String?,
  )

  @Serializable
  data class Patch(@SerialName("small") val small: String?, @SerialName("large") val large: String?)
  ```


  ```kotlin
  // shared/src/commonMain/kotlin/compose/project/demo/composedemo/data/remote/IRemoteRocketLaunchesDataSource.kt
  interface IRemoteRocketLaunchesDataSource {
    fun latestLaunches(): Flow<List<RocketLaunch>>
  }
  ```

  ```kotlin
  // shared/src/commonMain/kotlin/compose/project/demo/composedemo/data/remote/RemoteRocketLaunchesDataSource.kt
  class RemoteRocketLaunchesDataSource(
      private val httpClient: HttpClient,
      private val ioDispatcher: CoroutineDispatcher,
  ) : IRemoteRocketLaunchesDataSource {
    override fun latestLaunches(): Flow<List<RocketLaunch>> =
        flow {
              val latestLaunches =
                  httpClient.get("https://api.spacexdata.com/v5/launches").body<List<RocketLaunch>>()
              emit(latestLaunches)
            }
            .flowOn(ioDispatcher)
  }
  ```

1. Add RemoteRocketLaunchesDataSource to your DataModule.

  ```kotlin
  // shared/src/commonMain/kotlin/compose/project/demo/composedemo/di/modules/DataModule.kt
  val dataModule = module {
      single<IRemoteRocketLaunchesDataSource> { RemoteRocketLaunchesDataSource(get(), Dispatchers.IO) }
  }
  ```



<details>
<summary>Having trouble? 🤷</summary><br/>

- (replace-me: Troubleshooting tip or hint)
- (replace-me: Additional troubleshooting tips as needed)

</details>
