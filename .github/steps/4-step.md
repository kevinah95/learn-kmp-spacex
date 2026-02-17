
## Step 4: Add Repository and presentation layer

You now have both network and local data sources, but your app needs a clean way to coordinate between them. The Repository pattern provides a single source of truth for your data, while the presentation layer (ViewModel) manages UI state and business logic. Let's bring it all together!

### 📖 Theory: Clean Architecture and the Repository Pattern

**Clean Architecture** separates concerns into distinct layers, making your code more maintainable and testable. In KMP, this typically includes:
- **Data Layer**: Repositories, data sources (remote/local), and DTOs
- **Domain Layer**: Business logic and use cases (optional for simpler apps)
- **Presentation Layer**: ViewModels and UI state management

**The Repository Pattern** acts as a mediator between different data sources:
- Provides a **single source of truth** for the app's data
- **Abstracts data origin** - UI doesn't need to know if data comes from network or cache
- **Handles caching strategy** - when to fetch fresh data vs. use cached data
- **Manages error handling** - gracefully falls back to cached data when network fails

> [!TIP]
> The Repository in this implementation uses a "network-first with cache fallback" strategy. It tries to fetch fresh data, caches it locally, but falls back to cached data if the network request fails. This provides the best user experience!

**ViewModel and State Management:**
- **ViewModel**: Survives configuration changes and manages UI-related data
- **StateFlow**: Provides a reactive stream of UI state updates to the Compose UI
- **viewModelScope**: Automatically cancels coroutines when ViewModel is cleared
- **UiState**: A single data class representing the entire screen state

> [!IMPORTANT]
> Flow operators like `flowOn()` and `catch()` are crucial for proper threading and error handling. `flowOn()` affects upstream operations (before it), while `catch()` handles exceptions and can emit fallback values.

In this step, you'll:
- Implement the Repository pattern to coordinate data sources
- Create a ViewModel to manage UI state with StateFlow
- Use Flow operators for threading and error handling
- Write tests to verify the caching and fallback behavior

### ⌨️ Activity: Build Repository and ViewModel

1. Create a Repository interface in the shared module to abstract data operations.

   ```kotlin
   // shared/src/commonMain/kotlin/compose/project/demo/composedemo/data/repository/IRocketLaunchesRepository.kt
   interface IRocketLaunchesRepository {
      val latestLaunches: Flow<List<RocketLaunch>>
   }
   ```

1. Implement the Repository interface using both local and remote data sources.

   ```kotlin
   // shared/src/commonMain/kotlin/compose/project/demo/composedemo/data/repository/RocketLaunchesRepository.kt
   class RocketLaunchesRepository(
      private val localRocketLaunchesDataSource: ILocalRocketLaunchesDataSource,
      private val remoteRocketLaunchesDataSource: IRemoteRocketLaunchesDataSource,
      private val defaultDispatcher: CoroutineDispatcher,
  ) : IRocketLaunchesRepository {

    override val latestLaunches: Flow<List<RocketLaunch>> =
        remoteRocketLaunchesDataSource
            .latestLaunches()
            .onEach { launches -> // Executes on the default dispatcher
              localRocketLaunchesDataSource.clearAndCreateLaunches(launches)
            }
            // flowOn affects the upstream flow ↑
            .flowOn(defaultDispatcher)
            // the downstream flow ↓ is not affected
            // If an error happens, emit the last cached values
            .catch { exception -> // Executes in the consumer's context
              val cachedLaunches = localRocketLaunchesDataSource.getAllLaunches()
              if (cachedLaunches.isNotEmpty()) {
                emit(cachedLaunches)
              }
            }
  }
   ``` 

1. Update your Data Module to include the Repository and its dependencies.

   ```diff
   // shared/src/commonMain/kotlin/compose/project/demo/composedemo/di/modules/DataModule.kt
   val dataModule = module {
       // Other data layer dependencies
   +    single<IRocketLaunchesRepository> { RocketLaunchesRepository(get(), get(), Dispatchers.Default) }
   }
   ```

1. Create a UiState class to represent the state of your UI.

   ```kotlin
   // shared/src/commonMain/kotlin/compose/project/demo/composedemo/presentation/rocketLaunch/RocketLaunchUiState.kt
   data class RocketLaunchUiState(
      val isLoading: Boolean = false,
      val launches: List<RocketLaunch> = emptyList(),
  )
   ```

1. Create a ViewModel to manage the UI state and interact with the Repository.

   ```kotlin
   // shared/src/commonMain/kotlin/compose/project/demo/composedemo/presentation/rocketLaunch/RocketLaunchViewModel.kt
   class RocketLaunchViewModel(private val rocketLaunchesRepository: IRocketLaunchesRepository) :
      ViewModel() {
    private val _uiState = MutableStateFlow(RocketLaunchUiState())
    val uiState: StateFlow<RocketLaunchUiState> = _uiState.asStateFlow()

    init {
      loadLaunches()
    }

    fun loadLaunches() {
      viewModelScope.launch {
        _uiState.value = _uiState.value.copy(isLoading = true, launches = emptyList())
        try {
          rocketLaunchesRepository.latestLaunches.collect { launches ->
            _uiState.value = _uiState.value.copy(isLoading = false, launches = launches)
          }
        } catch (e: Exception) {
          _uiState.value = _uiState.value.copy(isLoading = false, launches = emptyList())
        }
      }
    }
  }
  ```

1. Update your Presentation Module to include the ViewModel and its dependencies.

  ```diff
  // shared/src/commonMain/kotlin/compose/project/demo/composedemo/di/modules/PresentationModule.kt
  val presentationModule = module {
      // Other presentation layer dependencies
  +    viewModel { RocketLaunchViewModel(get()) }
  }
  ```

1. Create a test for the Repository to verify its behavior.

  ```kotlin
  // shared/src/commonTest/kotlin/compose/project/demo/composedemo/data/repository/RocketLaunchesRepositoryTest.kt
  class RocketLaunchesRepositoryTest {

    private val localDataSource = mock<ILocalRocketLaunchesDataSource>()
    private val remoteDataSource = mock<IRemoteRocketLaunchesDataSource>()

    @Test
    fun `latestLaunches should fetch from remote and save to local`() = runTest {
      // Arrange
      val remoteLaunches =
          listOf(
              RocketLaunch(
                  flightNumber = 1,
                  missionName = "Falcon 1",
                  launchDateUTC = "2006-03-24T22:30:00.000Z",
                  details = null,
                  launchSuccess = false,
                  links = Links(Patch(null, null), null),
              )
          )

      every { remoteDataSource.latestLaunches() } returns flowOf(remoteLaunches)
      every { localDataSource.clearAndCreateLaunches(remoteLaunches) } returns Unit

      val repository =
          RocketLaunchesRepository(
              localRocketLaunchesDataSource = localDataSource,
              remoteRocketLaunchesDataSource = remoteDataSource,
              defaultDispatcher = Dispatchers.Unconfined,
          )

      // Act
      val result = repository.latestLaunches.first()

      // Assert
      assertEquals(remoteLaunches, result)
      verify { localDataSource.clearAndCreateLaunches(remoteLaunches) }
    }

    @Test
    fun `latestLaunches should return cached data when remote fails`() = runTest {
      // Arrange
      val cachedLaunches =
          listOf(
              RocketLaunch(
                  flightNumber = 2,
                  missionName = "Cached Mission",
                  launchDateUTC = "2024-01-01T00:00:00Z",
                  details = null,
                  launchSuccess = true,
                  links = Links(Patch(null, null), null),
              )
          )
      every { remoteDataSource.latestLaunches() } returns flow { throw Exception("Remote error") }
      every { localDataSource.getAllLaunches() } returns cachedLaunches

      val repository =
          RocketLaunchesRepository(
              localRocketLaunchesDataSource = localDataSource,
              remoteRocketLaunchesDataSource = remoteDataSource,
              defaultDispatcher = Dispatchers.Unconfined,
          )

      // Act
      val result = repository.latestLaunches.first()

      // Assert
      assertEquals(cachedLaunches, result)
      verify { localDataSource.getAllLaunches() }
    }
  }
  ```

1. Run your tests to ensure everything is working as expected.

<details>
<summary>Having trouble? 🤷</summary><br/>

- **Flow collection never completes**: Remember that `Flow.collect()` is a suspending function that runs continuously until the flow completes or is cancelled. If your repository flow never emits a completion, consider using `first()` to get just the first emission, or ensure the flow has proper lifecycle management.
- **State not updating in UI**: Make sure you're collecting the `uiState` StateFlow in your Composable using `collectAsState()`. Also verify that you're emitting new state objects (using `copy()`) rather than mutating existing ones - StateFlow only emits when the value reference changes.
- **ViewModel test failures**: When testing ViewModels with coroutines, use `runTest` from kotlinx-coroutines-test and set `Dispatchers.Main` to a test dispatcher. Also, remember that `viewModelScope` launches coroutines that may not complete immediately in tests.
- **Repository returns stale data**: Check the order of Flow operators. `flowOn()` affects operators above it (upstream), not below. Your data transformation should happen before `flowOn()` if you want it to run on that dispatcher.
- **catch block not executing**: The `catch()` operator only catches exceptions from upstream (before it). If an exception happens during collection (downstream), it won't be caught. Also, `catch()` must emit a value or re-throw to continue the flow.
- **Multiple LoadLaunches calls**: If `loadLaunches()` is called multiple times rapidly, you might want to cancel previous collections. Consider using `Flow.collect()` in a single coroutine or using `shareIn()`/`stateIn()` to share the flow.
- **Mock dependencies not working**: Make sure you've added a mocking library like MockK to your test dependencies. For KMP, you might need to add it to `commonTest` source set: `implementation("io.mockk:mockk:1.13.8")` or use manual mocks.
- **Dispatcher.Default vs Dispatcher.IO**: Use `Dispatchers.Default` for CPU-intensive work and `Dispatchers.IO` for I/O operations. In the repository, we use Default for the cache operation since it's a quick local write, while the network call in the data source uses IO.
- **copy() function not available on data class**: Ensure your UiState class is declared as a `data class`, not a regular `class`. The `copy()` function is automatically generated for data classes.

</details>
