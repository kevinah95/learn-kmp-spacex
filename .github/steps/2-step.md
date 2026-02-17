## Step 2: Setup Ktor and Coroutines

Now that you have Koin set up for dependency injection, it's time to connect your app to the real world! You'll fetch SpaceX launch data from their public API using Ktor, a modern HTTP client built for Kotlin Multiplatform, and handle asynchronous operations with Kotlin Coroutines.

### 📖 Theory: Networking and Asynchronous Programming in KMP

**Ktor** is a framework for building asynchronous clients and servers in Kotlin. The Ktor Client is perfect for KMP projects because:
- **Multiplatform**: Works on Android, iOS, and other platforms with platform-specific engines
- **Lightweight**: Only includes what you need through plugin-based architecture
- **Type-safe**: Leverages Kotlin's type system for safer HTTP requests
- **Coroutine-native**: Built from the ground up to work seamlessly with coroutines

**Kotlin Coroutines** provide a way to write asynchronous code that looks and behaves like synchronous code:
- **Structured Concurrency**: Ensures all async operations complete or cancel properly
- **Flow**: Reactive streams for handling multiple values over time
- **Dispatchers**: Control which thread your code runs on (IO, Main, Default)

> [!IMPORTANT]
> Ktor uses different HTTP engines for different platforms: OkHttp for Android and Darwin (NSURLSession) for iOS. This is why you'll see platform-specific dependencies in your configuration.

In this step, you'll:
- Configure Ktor with JSON serialization for the SpaceX API
- Create a data source that fetches rocket launch data
- Use Flow to emit data asynchronously
- Write comprehensive tests using Ktor's MockEngine


### ⌨️ Activity: Integrate Ktor and Coroutines

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
  ktor-client-mock = { module = "io.ktor:ktor-client-mock" }
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
          // Ktor
          implementation(libs.ktor.client.mock)
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

1. Add a test for RemoteRocketLaunchesDataSource using Ktor MockEngine.

  ```kotlin
  // shared/src/commonTest/kotlin/compose/project/demo/composedemo/data/remote/RemoteRocketLaunchesDataSourceTest.kt
  class RemoteRocketLaunchesDataSourceTest {

    private val json = Json { ignoreUnknownKeys = true }

    @Test
    fun `latestLaunches should return list of rocket launches on success`() = runTest {
      // Arrange
      val mockResponse = """
        [
          {
            "flight_number": 1,
            "name": "FalconSat",
            "date_utc": "2006-03-24T22:30:00.000Z",
            "details": "Engine failure at 33 seconds and loss of vehicle",
            "success": false,
            "links": {
              "patch": {
                "small": "https://images2.imgbox.com/3c/0e/T8iJcSN3_o.png",
                "large": "https://images2.imgbox.com/40/e3/GypSkayF_o.png"
              },
              "article": "https://www.space.com/2196-spacex-inaugural-falcon-1-rocket-lost-launch.html"
            }
          },
          {
            "flight_number": 2,
            "name": "DemoSat",
            "date_utc": "2007-03-21T01:10:00.000Z",
            "details": "Successful first stage burn and transition to second stage",
            "success": true,
            "links": {
              "patch": {
                "small": "https://images2.imgbox.com/4f/e3/I0lkuJ2e_o.png",
                "large": "https://images2.imgbox.com/3d/86/cnu0pan8_o.png"
              },
              "article": null
            }
          }
        ]
      """.trimIndent()

      val mockEngine = MockEngine { request ->
        assertEquals("https://api.spacexdata.com/v5/launches", request.url.toString())
        respond(
            content = mockResponse,
            status = HttpStatusCode.OK,
            headers = headersOf(HttpHeaders.ContentType, "application/json")
        )
      }

      val httpClient = HttpClient(mockEngine) {
        install(ContentNegotiation) {
          json(json)
        }
      }

      val dataSource = RemoteRocketLaunchesDataSource(
          httpClient = httpClient,
          ioDispatcher = Dispatchers.Unconfined
      )

      // Act
      val result = dataSource.latestLaunches().first()

      // Assert
      assertEquals(2, result.size)
      
      assertEquals(1, result[0].flightNumber)
      assertEquals("FalconSat", result[0].missionName)
      assertEquals("2006-03-24T22:30:00.000Z", result[0].launchDateUTC)
      assertEquals("Engine failure at 33 seconds and loss of vehicle", result[0].details)
      assertEquals(false, result[0].launchSuccess)
      assertEquals("https://images2.imgbox.com/3c/0e/T8iJcSN3_o.png", result[0].links.patch?.small)
      assertEquals("https://images2.imgbox.com/40/e3/GypSkayF_o.png", result[0].links.patch?.large)
      assertEquals("https://www.space.com/2196-spacex-inaugural-falcon-1-rocket-lost-launch.html", result[0].links.article)

      assertEquals(2, result[1].flightNumber)
      assertEquals("DemoSat", result[1].missionName)
      assertEquals("2007-03-21T01:10:00.000Z", result[1].launchDateUTC)
      assertEquals("Successful first stage burn and transition to second stage", result[1].details)
      assertEquals(true, result[1].launchSuccess)
      assertEquals("https://images2.imgbox.com/4f/e3/I0lkuJ2e_o.png", result[1].links.patch?.small)
      assertEquals("https://images2.imgbox.com/3d/86/cnu0pan8_o.png", result[1].links.patch?.large)
      assertEquals(null, result[1].links.article)

      httpClient.close()
    }

    @Test
    fun `latestLaunches should return empty list when API returns empty array`() = runTest {
      // Arrange
      val mockResponse = "[]"

      val mockEngine = MockEngine { request ->
        respond(
            content = mockResponse,
            status = HttpStatusCode.OK,
            headers = headersOf(HttpHeaders.ContentType, "application/json")
        )
      }

      val httpClient = HttpClient(mockEngine) {
        install(ContentNegotiation) {
          json(json)
        }
      }

      val dataSource = RemoteRocketLaunchesDataSource(
          httpClient = httpClient,
          ioDispatcher = Dispatchers.Unconfined
      )

      // Act
      val result = dataSource.latestLaunches().first()

      // Assert
      assertEquals(0, result.size)

      httpClient.close()
    }

    @Test
    fun `latestLaunches should throw exception when API returns error`() = runTest {
      // Arrange
      val mockEngine = MockEngine { request ->
        respond(
            content = "Internal Server Error",
            status = HttpStatusCode.InternalServerError,
            headers = headersOf(HttpHeaders.ContentType, "text/plain")
        )
      }

      val httpClient = HttpClient(mockEngine) {
        install(ContentNegotiation) {
          json(json)
        }
      }

      val dataSource = RemoteRocketLaunchesDataSource(
          httpClient = httpClient,
          ioDispatcher = Dispatchers.Unconfined
      )

      // Act & Assert
      assertFailsWith<Exception> {
        dataSource.latestLaunches().first()
      }

      httpClient.close()
    }

    @Test
    fun `latestLaunches should handle nullable fields correctly`() = runTest {
      // Arrange
      val mockResponse = """
        [
          {
            "flight_number": 3,
            "name": "Trailblazer",
            "date_utc": "2008-08-03T03:34:00.000Z",
            "details": null,
            "success": null,
            "links": {
              "patch": null,
              "article": null
            }
          }
        ]
      """.trimIndent()

      val mockEngine = MockEngine { request ->
        respond(
            content = mockResponse,
            status = HttpStatusCode.OK,
            headers = headersOf(HttpHeaders.ContentType, "application/json")
        )
      }

      val httpClient = HttpClient(mockEngine) {
        install(ContentNegotiation) {
          json(json)
        }
      }

      val dataSource = RemoteRocketLaunchesDataSource(
          httpClient = httpClient,
          ioDispatcher = Dispatchers.Unconfined
      )

      // Act
      val result = dataSource.latestLaunches().first()

      // Assert
      assertEquals(1, result.size)
      assertEquals(3, result[0].flightNumber)
      assertEquals("Trailblazer", result[0].missionName)
      assertEquals("2008-08-03T03:34:00.000Z", result[0].launchDateUTC)
      assertEquals(null, result[0].details)
      assertEquals(null, result[0].launchSuccess)
      assertEquals(null, result[0].links.patch)
      assertEquals(null, result[0].links.article)

      httpClient.close()
    }
  }
  ```

1. Run your tests to ensure everything is working correctly.


<details>
<summary>Having trouble? 🤷</summary><br/>

- **Serialization errors**: Make sure you've added the `@Serializable` annotation to your data classes and imported `kotlinx.serialization.SerialName` for the `@SerialName` annotations. The Kotlin serialization plugin should be applied in your build.gradle.kts.
- **Network connection issues in tests**: The tests use MockEngine, which simulates network responses without making real HTTP calls. If tests are failing, verify that your JSON mock responses match the expected data structure exactly.
- **Flow collection errors**: Remember that Flows are cold streams - they don't execute until collected. Use `.first()` in tests to collect the first emitted value. For production code, collect in a coroutine scope.
- **Dispatcher issues**: In tests, use `Dispatchers.Unconfined` instead of `Dispatchers.IO` to execute coroutines immediately on the current thread. In production code, always use the appropriate dispatcher (IO for network calls).
- **Missing NetworkModule**: Don't forget to include `networkModule` in your `sharedModule` in SharedModule.kt. Without it, Koin won't be able to provide the HttpClient dependency.
- **Platform-specific engine not found**: Ensure you've added the correct platform-specific Ktor client dependencies: `ktor-client-okhttp` for Android and `ktor-client-darwin` for iOS in their respective source sets.

</details>
