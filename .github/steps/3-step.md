## Step 3: Setup SQLDelight

Fetching data from the network is great, but what happens when users lose internet connection or want to view previously loaded launches? You'll need local data persistence! SQLDelight provides a type-safe SQL interface that works seamlessly across all platforms in your KMP project.

### 📖 Theory: Local Data Persistence with SQLDelight

**SQLDelight** is a multiplatform database library that generates type-safe Kotlin APIs from your SQL statements. It's an excellent choice for KMP projects because:
- **Write SQL once, use everywhere**: The same database schema works on Android, iOS, and other platforms
- **Type-safe queries**: Compile-time verification of SQL queries prevents runtime errors
- **Platform-specific drivers**: Uses the best native database driver for each platform (SQLite on Android, SQLite.swift on iOS)
- **No ORM overhead**: Direct SQL means better performance and control

> [!NOTE]
> SQLDelight generates Kotlin code from `.sq` files containing SQL statements. When you build your project, it creates type-safe functions that match your queries, ensuring you can't accidentally use wrong types or column names.

**Key Concepts:**
- **Schema files (.sq)**: Define your database tables and queries using standard SQL
- **Driver Factory**: Platform-specific implementations that provide the appropriate SQLite driver
- **Generated code**: SQLDelight automatically creates Kotlin APIs from your SQL
- **Transactions**: Ensure data consistency when performing multiple operations

In this step, you'll:
- Set up SQLDelight with platform-specific drivers
- Create a database schema for caching rocket launches
- Implement a local data source for offline access
- Integrate the local database with Koin for dependency injection

### ⌨️ Activity: Implement Local Database with SQLDelight

1. Add SQLDelight dependencies to your project.

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

1. Apply the SQLDelight plugin and configure it in your project modules.

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

  Build your project and SQLDelight will generate the necessary database code based on your schema.

1. Create a Database Driver Factory to provide platform-specific database drivers.

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

1. Update your PlatformModule to include the DriverFactory and its dependencies.

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

1. Create LocalRocketLaunchesDataSource to interact with the database.

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
<summary>Having trouble? 🤷</summary><br/>

- **Build fails after adding SQLDelight plugin**: Make sure you've synced your Gradle files after adding the plugin. SQLDelight generates code during the build process, so a clean build might help: `./gradlew clean build`.
- **Generated code not found**: SQLDelight generates code based on your `.sq` files. Ensure the `.sq` file is in the correct location: `shared/src/commonMain/sqldelight/{packagePath}/AppDatabase.sq`. The package path should match your configured `packageName`.
- **SQL syntax errors**: SQLDelight validates SQL at compile time. If you see SQL errors, check that your syntax matches SQLite standards. The `import kotlin.Boolean;` statement is necessary to map SQLite's INTEGER to Kotlin's Boolean type.
- **Platform driver not found**: Verify that you've added the correct platform-specific driver dependencies in the appropriate source sets: `android-driver` for `androidMain` and `native-driver` for `iosMain`.
- **Context parameter missing on Android**: The Android DriverFactory requires a Context parameter. Make sure your Android app is providing the application context to Koin. This is typically done in your Application class or MainActivity.
- **Database queries return null or wrong data**: Check your mapper function (`mapLaunchSelecting`) - the parameter order must match the column order in your SELECT query. Type mismatches here can cause subtle bugs.
- **Transaction errors**: When using `dbQuery.transaction {}`, ensure all operations inside complete successfully. If one fails, the entire transaction rolls back. This is normal behavior to maintain data consistency.
- **linkSqlite configuration**: The `linkSqlite = true` setting in the SQLDelight configuration is crucial for iOS - it links the SQLite library into your iOS framework. Without it, you'll get runtime errors on iOS.

</details>
