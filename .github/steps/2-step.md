## Paso 2: Configurar Ktor y Coroutines

Ahora que tenés Koin configurado para inyección de dependencias, ¡es hora de conectar tu app al mundo real! Vas a obtener datos de lanzamientos de SpaceX desde su API pública usando Ktor, un cliente HTTP moderno construido para Kotlin Multiplatform, y manejar operaciones asíncronas con Kotlin Coroutines.

### 📖 Teoría: Networking y Programación Asíncrona en KMP

**Ktor** es un framework para construir clientes y servidores asíncronos en Kotlin. El Cliente Ktor es perfecto para proyectos KMP porque:
- **Multiplataforma**: Funciona en Android, iOS y otras plataformas con engines específicos de cada plataforma
- **Liviano**: Solo incluye lo que necesitás a través de una arquitectura basada en plugins
- **Type-safe**: Aprovecha el sistema de tipos de Kotlin para peticiones HTTP más seguras
- **Nativo de coroutines**: Construido desde cero para funcionar perfectamente con coroutines

**Kotlin Coroutines** proporciona una forma de escribir código asíncrono que se ve y comporta como código síncrono:
- **Structured Concurrency**: Asegura que todas las operaciones asíncronas se completen o cancelen correctamente
- **Flow**: Streams reactivos para manejar múltiples valores con el tiempo
- **Dispatchers**: Controlan en qué thread se ejecuta tu código (IO, Main, Default)

> [!IMPORTANT]
> Ktor usa diferentes engines HTTP para diferentes plataformas: OkHttp para Android y Darwin (NSURLSession) para iOS. Por eso verás dependencias específicas de plataforma en tu configuración.

En este paso, vas a:
- Configurar Ktor con serialización JSON para la API de SpaceX
- Crear una fuente de datos que obtiene información de lanzamientos de cohetes
- Usar Flow para emitir datos de forma asíncrona
- Escribir pruebas completas usando el MockEngine de Ktor


### ⌨️ Actividad: Integrar Ktor y Coroutines

1. Agregá las dependencias de Ktor y Coroutines a tu proyecto.

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

1. Agregá las dependencias de Ktor y Coroutines a los módulos de tu proyecto.

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

1. Agregá la dependencia del cliente Ktor a tu NetworkModule.

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

1. Creá un nuevo archivo para usar el cliente Ktor en tu capa de datos.

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

1. Agregá RemoteRocketLaunchesDataSource a tu DataModule.

  ```kotlin
  // shared/src/commonMain/kotlin/compose/project/demo/composedemo/di/modules/DataModule.kt
  val dataModule = module {
      single<IRemoteRocketLaunchesDataSource> { RemoteRocketLaunchesDataSource(get(), Dispatchers.IO) }
  }
  ```

1. Agregá un test para RemoteRocketLaunchesDataSource usando Ktor MockEngine.

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

1. Ejecutá tus pruebas para asegurar que todo funciona correctamente.


<details>
<summary>¿Tenés problemas? 🤷</summary><br/>

- **Errores de serialización**: Asegurate de haber agregado la anotación `@Serializable` a tus clases de datos e importado `kotlinx.serialization.SerialName` para las anotaciones `@SerialName`. El plugin de serialización de Kotlin debe estar aplicado en tu build.gradle.kts.
- **Problemas de conexión de red en tests**: Los tests usan MockEngine, que simula respuestas de red sin hacer llamadas HTTP reales. Si los tests fallan, verificá que tus respuestas JSON mock coincidan exactamente con la estructura de datos esperada.
- **Errores de colección de Flow**: Recordá que los Flows son streams fríos - no se ejecutan hasta que son colectados. Usá `.first()` en tests para colectar el primer valor emitido. Para código de producción, colectá en un coroutine scope.
- **Problemas con Dispatchers**: En tests, usá `Dispatchers.Unconfined` en lugar de `Dispatchers.IO` para ejecutar coroutines inmediatamente en el thread actual. En código de producción, siempre usá el dispatcher apropiado (IO para llamadas de red).
- **NetworkModule faltante**: No te olvidés de incluir `networkModule` en tu `sharedModule` en SharedModule.kt. Sin él, Koin no podrá proveer la dependencia HttpClient.
- **Engine específico de plataforma no encontrado**: Asegurate de haber agregado las dependencias correctas del cliente Ktor específicas de plataforma: `ktor-client-okhttp` para Android y `ktor-client-darwin` para iOS en sus respectivos source sets.

</details>
