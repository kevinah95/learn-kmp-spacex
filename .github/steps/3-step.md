## Paso 3: Configurar SQLDelight

Obtener datos de la red es genial, pero ¿qué pasa cuando los usuarios pierden la conexión a internet o quieren ver lanzamientos cargados previamente? ¡Necesitás persistencia de datos local! SQLDelight proporciona una interfaz SQL type-safe que funciona perfectamente en todas las plataformas de tu proyecto KMP.

### 📖 Teoría: Persistencia de Datos Local con SQLDelight

**SQLDelight** es una biblioteca de bases de datos multiplataforma que genera APIs de Kotlin type-safe desde tus sentencias SQL. Es una excelente opción para proyectos KMP porque:
- **Escribí SQL una vez, usálo en todas partes**: El mismo esquema de base de datos funciona en Android, iOS y otras plataformas
- **Consultas type-safe**: La verificación de consultas SQL en tiempo de compilación previene errores en tiempo de ejecución
- **Drivers específicos de plataforma**: Usa el mejor driver de base de datos nativo para cada plataforma (SQLite en Android, SQLite.swift en iOS)
- **Sin overhead de ORM**: SQL directo significa mejor rendimiento y control

> [!NOTE]
> SQLDelight genera código Kotlin desde archivos `.sq` que contienen sentencias SQL. Cuando construyés tu proyecto, crea funciones type-safe que coinciden con tus consultas, asegurando que no podés usar accidentalmente tipos o nombres de columnas incorrectos.

**Conceptos Clave:**
- **Archivos de esquema (.sq)**: Definí tus tablas y consultas de base de datos usando SQL estándar
- **Driver Factory**: Implementaciones específicas de plataforma que proveen el driver SQLite apropiado
- **Código generado**: SQLDelight crea automáticamente APIs de Kotlin desde tu SQL
- **Transacciones**: Aseguran consistencia de datos al realizar múltiples operaciones

En este paso, vas a:
- Configurar SQLDelight con drivers específicos de plataforma
- Crear un esquema de base de datos para cachear lanzamientos de cohetes
- Implementar una fuente de datos local para acceso offline
- Integrar la base de datos local con Koin para inyección de dependencias

### ⌨️ Actividad: Implementar Base de Datos Local con SQLDelight

1. Agregá las dependencias de SQLDelight a tu proyecto.

  ```kotlin
  // gradle/libs.versions.toml
  [versions]
  sqldelight = "2.2.1"

  [libraries]
  # SQLDelight
  sqldelight-driver-android = { module = "app.cash.sqldelight:android-driver", version.ref = "sqldelight" }
  sqldelight-driver-native = { module = "app.cash.sqldelight:native-driver", version.ref = "sqldelight" }

  [plugins]
  # SQLDelight
  sqlDelight = { id = "app.cash.sqldelight", version.ref = "sqldelight" }
  ```

1. Aplicá el plugin de SQLDelight y configurálo en los módulos de tu proyecto.

  ```kotlin
  // shared/build.gradle.kts
  plugins {
      // ... other plugins
      alias(libs.plugins.sqlDelight)
  }

  kotlin {
    sourceSets {
        androidMain.dependencies {
            // ... other dependencies
            // SQLDelight
            implementation(libs.sqldelight.driver.android)
        }
        iosMain.dependencies {
            // ... other dependencies
            // SQLDelight
            implementation(libs.sqldelight.driver.native)
        }
    }
  }
  // ...
  // at the end of the file
  sqldelight {
    databases { create("AppDatabase") { packageName.set("compose.project.demo.composedemo.data.local") } }
    linkSqlite = true
  }
  ```

1. Create your SQLDelight database schema and generate the database code.

  ```sql
  -- shared/src/commonMain/sqldelight/compose/project/demo/composedemo/data/local/AppDatabase.sq
  import kotlin.Boolean;

  CREATE TABLE Launch (
      flightNumber INTEGER NOT NULL,
      missionName TEXT NOT NULL,
      details TEXT,
      launchSuccess INTEGER AS Boolean DEFAULT NULL,
      launchDateUTC TEXT NOT NULL,
      patchUrlSmall TEXT,
      patchUrlLarge TEXT,
      articleUrl TEXT
  );

  insertLaunch:
  INSERT INTO Launch(flightNumber, missionName, details, launchSuccess, launchDateUTC, patchUrlSmall, patchUrlLarge, articleUrl)
  VALUES(?, ?, ?, ?, ?, ?, ?, ?);

  removeAllLaunches:
  DELETE FROM Launch;

  selectAllLaunchesInfo:
  SELECT Launch.*
  FROM Launch;
  ```

  Construí tu proyecto y SQLDelight generará el código de base de datos necesario basado en tu esquema.

1. Creá una Database Driver Factory para proveer drivers de base de datos específicos de plataforma.

  ```kotlin
  // shared/src/commonMain/kotlin/compose/project/demo/composedemo/data/local/DriverFactory.kt
  expect class DriverFactory {
      fun createDriver(): SqlDriver
  }
  ```

  ```kotlin
  // shared/src/androidMain/kotlin/compose/project/demo/composedemo/data/local/DriverFactory.android.kt
  actual class DriverFactory(private val context: Context) {
      actual fun createDriver(): SqlDriver {
          return AndroidSqliteDriver(AppDatabase.Schema, context, "launch.db")
      }
  }
  ```

  ```kotlin
  // shared/src/iosMain/kotlin/compose/project/demo/composedemo/data/local/DriverFactory.ios.kt
  actual class DriverFactory {
      actual fun createDriver(): SqlDriver {
          return NativeSqliteDriver(AppDatabase.Schema, "launch.db")
      }
  }
  ```

1. Actualizá tu PlatformModule para incluir el DriverFactory y sus dependencias.

  ```diff
  // shared/src/androidMain/kotlin/compose/project/demo/composedemo/di/modules/PlatformModule.android.kt
  actual fun platformModule(): Module = module {
  +    single { DriverFactory(get()) }
  }
  ```

  ```diff
  // shared/src/iosMain/kotlin/compose/project/demo/composedemo/di/modules/PlatformModule.ios.kt
  actual fun platformModule(): Module = module {
  +    single { DriverFactory() }
  }
  ```

1. Creá LocalRocketLaunchesDataSource para interactuar con la base de datos.

  ```kotlin
  // shared/src/commonMain/kotlin/compose/project/demo/composedemo/data/local/ILocalRocketLaunchesDataSource.kt
  interface ILocalRocketLaunchesDataSource {
    fun getAllLaunches(): List<RocketLaunch>

    fun clearAndCreateLaunches(launches: List<RocketLaunch>)
  }
  ```

  ```kotlin
  // shared/src/commonMain/kotlin/compose/project/demo/composedemo/data/local/LocalRocketLaunchesDataSource.kt
  class LocalRocketLaunchesDataSource(database: AppDatabase) : ILocalRocketLaunchesDataSource {
    private val dbQuery = database.appDatabaseQueries

    override fun getAllLaunches(): List<RocketLaunch> {
      return dbQuery.selectAllLaunchesInfo(::mapLaunchSelecting).executeAsList()
    }

    private fun mapLaunchSelecting(
        flightNumber: Long,
        missionName: String,
        details: String?,
        launchSuccess: Boolean?,
        launchDateUTC: String,
        patchUrlSmall: String?,
        patchUrlLarge: String?,
        articleUrl: String?,
    ): RocketLaunch {
      return RocketLaunch(
          flightNumber = flightNumber.toInt(),
          missionName = missionName,
          details = details,
          launchDateUTC = launchDateUTC,
          launchSuccess = launchSuccess,
          links =
              Links(
                  patch = Patch(small = patchUrlSmall, large = patchUrlLarge),
                  article = articleUrl,
              ),
      )
    }

    override fun clearAndCreateLaunches(launches: List<RocketLaunch>) {
      dbQuery.transaction {
        dbQuery.removeAllLaunches()
        launches.forEach { launch ->
          dbQuery.insertLaunch(
              flightNumber = launch.flightNumber.toLong(),
              missionName = launch.missionName,
              details = launch.details,
              launchSuccess = launch.launchSuccess ?: false,
              launchDateUTC = launch.launchDateUTC,
              patchUrlSmall = launch.links.patch?.small,
              patchUrlLarge = launch.links.patch?.large,
              articleUrl = launch.links.article,
          )
        }
      }
    }
  }
  ```

1. Update DataModule to include the LocalRocketLaunchesDataSource and its dependencies.

  ```diff
  // shared/src/commonMain/kotlin/compose/project/demo/composedemo/di/modules/DataModule.kt
  val dataModule = module {
      single<IRemoteRocketLaunchesDataSource> { RemoteRocketLaunchesDataSource(get(), Dispatchers.IO) }
  +    single { get<DriverFactory>().createDriver() }
  +    single { AppDatabase(get()) }
  +    single { get<AppDatabase>().appDatabaseQueries }
  +    single<ILocalRocketLaunchesDataSource> { LocalRocketLaunchesDataSource(get()) }
  }
  ```

<details>
<summary>¿Tenés problemas? 🤷</summary><br/>

- **Falla la construcción después de agregar el plugin SQLDelight**: Asegurate de haber sincronizado tus archivos Gradle después de agregar el plugin. SQLDelight genera código durante el proceso de construcción, así que una construcción limpia podría ayudar: `./gradlew clean build`.
- **Código generado no encontrado**: SQLDelight genera código basado en tus archivos `.sq`. Asegurate de que el archivo `.sq` esté en la ubicación correcta: `shared/src/commonMain/sqldelight/{packagePath}/AppDatabase.sq`. El package path debe coincidir con tu `packageName` configurado.
- **Errores de sintaxis SQL**: SQLDelight valida SQL en tiempo de compilación. Si ves errores SQL, verificá que tu sintaxis coincida con los estándares de SQLite. La sentencia `import kotlin.Boolean;` es necesaria para mapear el INTEGER de SQLite al tipo Boolean de Kotlin.
- **Driver de plataforma no encontrado**: Verificá que hayas agregado las dependencias correctas de driver específicas de plataforma en los source sets apropiados: `android-driver` para `androidMain` y `native-driver` para `iosMain`.
- **Parámetro Context faltante en Android**: El DriverFactory de Android requiere un parámetro Context. Asegurate de que tu app Android esté proveyendo el contexto de aplicación a Koin. Esto típicamente se hace en tu clase Application o MainActivity.
- **Las consultas de base de datos retornan null o datos incorrectos**: Verificá tu función mapper (`mapLaunchSelecting`) - el orden de los parámetros debe coincidir con el orden de las columnas en tu consulta SELECT. Desajustes de tipos aquí pueden causar bugs sutiles.
- **Errores de transacción**: Cuando uses `dbQuery.transaction {}`, asegurate de que todas las operaciones dentro se completen exitosamente. Si una falla, toda la transacción se revierte. Esto es comportamiento normal para mantener la consistencia de datos.
- **Configuración linkSqlite**: La configuración `linkSqlite = true` en la configuración de SQLDelight es crucial para iOS - enlaza la biblioteca SQLite en tu framework iOS. Sin ella, obtendrás errores en tiempo de ejecución en iOS.

</details>
