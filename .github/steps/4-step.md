
## Paso 4: Agregar Repository y capa de presentación

Ahora tenés fuentes de datos tanto de red como locales, pero tu app necesita una forma limpia de coordinar entre ellas. El patrón Repository proporciona una única fuente de verdad para tus datos, mientras que la capa de presentación (ViewModel) gestiona el estado de UI y la lógica de negocio. ¡Vamos a juntarlo todo!

### 📖 Teoría: Arquitectura Limpia y el Patrón Repository

**Arquitectura Limpia** separa responsabilidades en capas distintas, haciendo tu código más mantenible y testeable. En KMP, esto típicamente incluye:
- **Capa de Datos**: Repositorios, fuentes de datos (remota/local), y DTOs
- **Capa de Dominio**: Lógica de negocio y casos de uso (opcional para apps más simples)
- **Capa de Presentación**: ViewModels y gestión de estado de UI

**El Patrón Repository** actúa como un mediador entre diferentes fuentes de datos:
- Proporciona una **única fuente de verdad** para los datos de la app
- **Abstrae el origen de datos** - la UI no necesita saber si los datos vienen de la red o el caché
- **Maneja la estrategia de caché** - cuándo obtener datos frescos vs. usar datos en caché
- **Gestiona el manejo de errores** - retrocede elegantemente a datos en caché cuando falla la red

> [!TIP]
> El Repository en esta implementación usa una estrategia de "red primero con retroceso a caché". Intenta obtener datos frescos, los cachea localmente, pero retrocede a datos en caché si la petición de red falla. ¡Esto proporciona la mejor experiencia de usuario!

**ViewModel y Gestión de Estado:**
- **ViewModel**: Sobrevive a cambios de configuración y gestiona datos relacionados con la UI
- **StateFlow**: Proporciona un stream reactivo de actualizaciones de estado de UI a la UI de Compose
- **viewModelScope**: Cancela automáticamente coroutines cuando el ViewModel se limpia
- **UiState**: Una sola data class que representa todo el estado de la pantalla

> [!IMPORTANT]
> Operadores de Flow como `flowOn()` y `catch()` son cruciales para el threading y manejo de errores apropiado. `flowOn()` afecta operaciones upstream (antes de él), mientras que `catch()` maneja excepciones y puede emitir valores de retroceso.

En este paso, vas a:
- Implementar el patrón Repository para coordinar fuentes de datos
- Crear un ViewModel para gestionar estado de UI con StateFlow
- Usar operadores de Flow para threading y manejo de errores
- Escribir tests para verificar el comportamiento de caché y retroceso

### ⌨️ Actividad: Construir Repository y ViewModel

1. Creá una interfaz Repository en el módulo shared para abstraer operaciones de datos.

   ```kotlin
   // shared/src/commonMain/kotlin/compose/project/demo/composedemo/data/repository/IRocketLaunchesRepository.kt
   interface IRocketLaunchesRepository {
      val latestLaunches: Flow<List<RocketLaunch>>
   }
   ```

1. Implementá la interfaz Repository usando fuentes de datos locales y remotas.

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

1. Actualizá tu Data Module para incluir el Repository y sus dependencias.

   ```diff
   // shared/src/commonMain/kotlin/compose/project/demo/composedemo/di/modules/DataModule.kt
   val dataModule = module {
       // Other data layer dependencies
   +    single<IRocketLaunchesRepository> { RocketLaunchesRepository(get(), get(), Dispatchers.Default) }
   }
   ```

1. Creá una clase UiState para representar el estado de tu UI.

   ```kotlin
   // shared/src/commonMain/kotlin/compose/project/demo/composedemo/presentation/rocketLaunch/RocketLaunchUiState.kt
   data class RocketLaunchUiState(
      val isLoading: Boolean = false,
      val launches: List<RocketLaunch> = emptyList(),
  )
   ```

1. Creá un ViewModel para gestionar el estado de UI e interactuar con el Repository.

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

1. Actualizá tu Presentation Module para incluir el ViewModel y sus dependencias.

  ```diff
  // shared/src/commonMain/kotlin/compose/project/demo/composedemo/di/modules/PresentationModule.kt
  val presentationModule = module {
      // Other presentation layer dependencies
  +    viewModel { RocketLaunchViewModel(get()) }
  }
  ```

1. Creá un test para el Repository para verificar su comportamiento.

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

1. Ejecutá tus tests para asegurar que todo funciona como se espera.

<details>
<summary>¿Tenés problemas? 🤷</summary><br/>

- **La colección de Flow nunca se completa**: Recordá que `Flow.collect()` es una función suspendida que se ejecuta continuamente hasta que el flow se completa o se cancela. Si tu flow del repository nunca emite una finalización, considerá usar `first()` para obtener solo la primera emisión, o asegurate de que el flow tenga una gestión de ciclo de vida apropiada.
- **El estado no se actualiza en la UI**: Asegurate de estar colectando el StateFlow `uiState` en tu Composable usando `collectAsState()`. También verificá que estés emitiendo nuevos objetos de estado (usando `copy()`) en lugar de mutar los existentes - StateFlow solo emite cuando la referencia del valor cambia.
- **Fallas en tests del ViewModel**: Cuando pruebes ViewModels con coroutines, usá `runTest` de kotlinx-coroutines-test y establecé `Dispatchers.Main` a un dispatcher de test. Además, recordá que `viewModelScope` lanza coroutines que pueden no completarse inmediatamente en tests.
- **El Repository devuelve datos obsoletos**: Verificá el orden de los operadores de Flow. `flowOn()` afecta operadores arriba de él (upstream), no abajo. Tu transformación de datos debe ocurrir antes de `flowOn()` si querés que se ejecute en ese dispatcher.
- **El bloque catch no se ejecuta**: El operador `catch()` solo captura excepciones desde upstream (antes de él). Si una excepción ocurre durante la colección (downstream), no será capturada. Además, `catch()` debe emitir un valor o relanzar para continuar el flow.
- **Múltiples llamadas a LoadLaunches**: Si `loadLaunches()` se llama múltiples veces rápidamente, podrías querer cancelar colecciones previas. Considerá usar `Flow.collect()` en una sola coroutine o usar `shareIn()`/`stateIn()` para compartir el flow.
- **Dependencias mock no funcionan**: Asegurate de haber agregado una biblioteca de mocking como MockK a tus dependencias de test. Para KMP, podrías necesitar agregarla al source set `commonTest`: `implementation("io.mockk:mockk:1.13.8")` o usar mocks manuales.
- **Dispatcher.Default vs Dispatcher.IO**: Usá `Dispatchers.Default` para trabajo intensivo de CPU y `Dispatchers.IO` para operaciones de I/O. En el repository, usamos Default para la operación de caché ya que es una escritura local rápida, mientras que la llamada de red en el data source usa IO.
- **Función copy() no disponible en data class**: Asegurate de que tu clase UiState esté declarada como `data class`, no una `class` regular. La función `copy()` se genera automáticamente para data classes.

</details>
