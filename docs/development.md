# Liftrix Development Guide

## Code Organization & Architecture

### Project Structure Overview

Liftrix follows a feature-based modular architecture organized within Clean Architecture boundaries. The codebase is structured for scalability, maintainability, and clear separation of concerns.

```
app/src/main/java/com/example/liftrix/
├── core/                          # Cross-cutting concerns
│   ├── design/                   # Design system components
│   ├── error/                    # Error handling utilities
│   ├── network/                  # Network configuration
│   └── common/                   # Shared utilities
├── data/                         # Data layer implementation
│   ├── local/                    # Room database
│   │   ├── dao/                  # 28 Data Access Objects
│   │   ├── entity/               # 29 Database entities
│   │   └── converter/            # Type converters
│   ├── repository/               # 16 Repository implementations
│   ├── mapper/                   # Data transformation
│   ├── sync/                     # Synchronization logic
│   └── extensions/               # Data utilities
├── domain/                       # Business logic layer
│   ├── model/                    # Domain models
│   ├── repository/               # Repository interfaces
│   ├── usecase/                  # 85+ Use case implementations
│   └── service/                  # Domain services
├── di/                          # Dependency injection (22 modules)
├── sync/                        # Background sync workers
├── ui/                          # Presentation layer
│   ├── auth/                    # Authentication screens
│   ├── feed/                    # Social feed features
│   ├── home/                    # Home dashboard
│   ├── navigation/              # Navigation system
│   ├── profile/                 # User profile management
│   ├── progress/                # Analytics and charts
│   ├── settings/                # Application settings
│   ├── social/                  # Social features
│   ├── workout/                 # Workout management
│   └── components/              # Reusable UI components
└── LiftrixApp.kt               # Application entry point
```

### Package Organization Principles

**Layer-Based Organization**:
- **Core**: Framework-agnostic utilities and cross-cutting concerns
- **Data**: External dependencies and data access implementations
- **Domain**: Pure business logic without framework dependencies
- **UI**: Android-specific presentation components

**Feature Cohesion**:
```kotlin
// Example: Workout feature organization
ui/workout/
├── creation/                    # Workout creation flow
│   ├── WorkoutCreationScreen.kt
│   ├── WorkoutCreationViewModel.kt
│   └── components/
├── details/                     # Workout details and editing
│   ├── WorkoutDetailsScreen.kt
│   ├── WorkoutDetailsViewModel.kt
│   └── components/
├── selection/                   # Exercise selection
│   ├── ExerciseSelectionScreen.kt
│   └── ExerciseSelectionViewModel.kt
└── history/                     # Workout history
    ├── WorkoutHistoryScreen.kt
    └── WorkoutHistoryViewModel.kt
```

## Development Patterns & Best Practices

### 1. MVVM with MVI Elements

**BaseViewModel Pattern**:
```kotlin
// Abstract base for consistent ViewModel implementation
abstract class BaseViewModel<S : Any, E : Any> : ViewModel() {
    
    private val _uiState = MutableStateFlow<UiState<S>>(UiState.Loading)
    val uiState: StateFlow<UiState<S>> = _uiState.asStateFlow()
    
    private val _events = MutableSharedFlow<E>()
    
    init {
        // Collect and handle events
        viewModelScope.launch {
            _events.collect { event ->
                handleEvent(event)
            }
        }
    }
    
    abstract fun handleEvent(event: E)
    
    protected fun updateState(newState: UiState<S>) {
        _uiState.value = newState
    }
    
    protected fun updateState(transform: (S) -> S) {
        val currentState = _uiState.value
        if (currentState is UiState.Success) {
            _uiState.value = UiState.Success(transform(currentState.data))
        }
    }
    
    fun handleEvent(event: E) {
        viewModelScope.launch {
            _events.emit(event)
        }
    }
}

// Implementation example
class WorkoutDetailsViewModel @Inject constructor(
    private val getWorkoutUseCase: GetWorkoutUseCase,
    private val updateWorkoutUseCase: UpdateWorkoutUseCase
) : BaseViewModel<WorkoutDetailsState, WorkoutDetailsEvent>() {
    
    override fun handleEvent(event: WorkoutDetailsEvent) {
        when (event) {
            is WorkoutDetailsEvent.LoadWorkout -> loadWorkout(event.workoutId)
            is WorkoutDetailsEvent.UpdateExercise -> updateExercise(event.exerciseId, event.data)
            is WorkoutDetailsEvent.SaveWorkout -> saveWorkout()
        }
    }
    
    private fun loadWorkout(workoutId: String) {
        viewModelScope.launch {
            updateState(UiState.Loading)
            
            val result = getWorkoutUseCase(workoutId)
            result.fold(
                onSuccess = { workout ->
                    updateState(UiState.Success(WorkoutDetailsState(workout = workout)))
                },
                onFailure = { error ->
                    updateState(UiState.Error(error))
                }
            )
        }
    }
}
```

### 2. Error Handling Strategy

**LiftrixResult<T> Pattern**:
```kotlin
// Comprehensive error handling with recovery options
sealed class LiftrixResult<out T> {
    data class Success<T>(val data: T) : LiftrixResult<T>()
    data class Error(val error: LiftrixError) : LiftrixResult<Nothing>()
    
    inline fun <R> fold(
        onSuccess: (T) -> R,
        onFailure: (LiftrixError) -> R
    ): R = when (this) {
        is Success -> onSuccess(data)
        is Error -> onFailure(error)
    }
    
    inline fun <R> map(transform: (T) -> R): LiftrixResult<R> = when (this) {
        is Success -> Success(transform(data))
        is Error -> this
    }
    
    inline fun onSuccess(action: (T) -> Unit): LiftrixResult<T> {
        if (this is Success) action(data)
        return this
    }
    
    inline fun onFailure(action: (LiftrixError) -> Unit): LiftrixResult<T> {
        if (this is Error) action(error)
        return this
    }
}

// Use case implementation pattern
class CreateWorkoutUseCase @Inject constructor(
    private val workoutRepository: WorkoutRepository,
    private val getCurrentUserIdUseCase: GetCurrentUserIdUseCase,
    private val validationService: ValidationService
) {
    suspend operator fun invoke(request: CreateWorkoutRequest): LiftrixResult<Workout> = 
        liftrixCatching(
            errorMapper = { throwable ->
                when (throwable) {
                    is IllegalArgumentException -> LiftrixError.ValidationError(
                        field = "workout_data",
                        violations = listOf("Invalid workout structure"),
                        analyticsContext = mapOf("operation" to "CREATE_WORKOUT")
                    )
                    is SecurityException -> LiftrixError.AuthenticationError(
                        reason = AuthFailureReason.INSUFFICIENT_PERMISSIONS
                    )
                    else -> LiftrixError.BusinessLogicError(
                        code = "WORKOUT_CREATION_FAILED",
                        errorMessage = "Failed to create workout",
                        operation = "CREATE_WORKOUT",
                        analyticsContext = mapOf(
                            "user_id" to getCurrentUserIdUseCase().orEmpty(),
                            "exercise_count" to request.exercises.size.toString()
                        )
                    )
                }
            }
        ) {
            // Validate input
            validationService.validateWorkoutRequest(request)
            
            // Get current user
            val userId = getCurrentUserIdUseCase() 
                ?: throw SecurityException("User not authenticated")
            
            // Create domain model
            val workout = request.toDomain(userId)
            
            // Persist to repository
            workoutRepository.createWorkout(workout)
        }
}
```

### 3. Repository Pattern Implementation

**Clean Architecture Repository Pattern**:
```kotlin
// Repository interface in domain layer
interface WorkoutRepository {
    suspend fun createWorkout(workout: Workout): LiftrixResult<Workout>
    suspend fun getWorkoutsForUser(userId: String): Flow<List<Workout>>
    suspend fun updateWorkout(workout: Workout): LiftrixResult<Workout>
    suspend fun deleteWorkout(workoutId: String, userId: String): LiftrixResult<Unit>
    suspend fun getWorkoutById(workoutId: String, userId: String): LiftrixResult<Workout>
}

// Repository implementation in data layer
@Singleton
class WorkoutRepositoryImpl @Inject constructor(
    private val localDataSource: WorkoutDao,
    private val remoteDataSource: FirebaseWorkoutDataSource,
    private val syncCoordinator: SyncCoordinator,
    private val workoutMapper: WorkoutMapper
) : WorkoutRepository {
    
    override suspend fun createWorkout(workout: Workout): LiftrixResult<Workout> = 
        liftrixCatching(
            errorMapper = { LiftrixError.DatabaseError("Failed to create workout") }
        ) {
            // Convert to entity
            val entity = workoutMapper.toEntity(workout)
            
            // Store locally first (offline-first)
            val savedEntity = localDataSource.insertWorkout(entity)
            
            // Queue for sync
            syncCoordinator.queueWorkoutSync(workout.userId, savedEntity.id)
            
            // Return domain model
            workoutMapper.toDomain(savedEntity)
        }
    
    override suspend fun getWorkoutsForUser(userId: String): Flow<List<Workout>> {
        return localDataSource.getWorkoutsForUser(userId)
            .map { entities -> 
                entities.map { workoutMapper.toDomain(it) }
            }
            .onStart {
                // Trigger background sync
                syncCoordinator.syncWorkouts(userId)
            }
            .catch { exception ->
                Timber.e(exception, "Failed to get workouts for user: $userId")
                emit(emptyList())
            }
    }
}
```

### 4. Database Access Pattern

**User-Scoped Security (Mandatory)**:
```kotlin
// All DAO methods MUST include user_id filtering
@Dao
interface WorkoutDao {
    
    @Query("""
        SELECT * FROM workouts 
        WHERE user_id = :userId 
        ORDER BY created_at DESC
    """)
    suspend fun getWorkoutsForUser(userId: String): List<WorkoutEntity>
    
    @Query("""
        SELECT * FROM workouts 
        WHERE user_id = :userId AND id = :workoutId
    """)
    suspend fun getWorkoutById(userId: String, workoutId: String): WorkoutEntity?
    
    @Query("""
        UPDATE workouts 
        SET name = :name, description = :description, last_modified = :timestamp
        WHERE user_id = :userId AND id = :workoutId
    """)
    suspend fun updateWorkout(
        userId: String, 
        workoutId: String, 
        name: String, 
        description: String,
        timestamp: Long = System.currentTimeMillis()
    )
    
    @Query("""
        DELETE FROM workouts 
        WHERE user_id = :userId AND id = :workoutId
    """)
    suspend fun deleteWorkout(userId: String, workoutId: String)
    
    // Sync-specific queries
    @Query("""
        SELECT * FROM workouts 
        WHERE user_id = :userId AND is_synced = 0
    """)
    suspend fun getUnsyncedWorkouts(userId: String): List<WorkoutEntity>
    
    @Query("""
        UPDATE workouts 
        SET is_synced = :isSynced, sync_version = :version
        WHERE user_id = :userId AND id IN (:workoutIds)
    """)
    suspend fun updateSyncStatus(
        userId: String, 
        workoutIds: List<String>, 
        isSynced: Boolean, 
        version: Long
    )
}

// Entity with sync metadata
@Entity(
    tableName = "workouts",
    indices = [
        Index(value = ["user_id"]),
        Index(value = ["user_id", "created_at"]),
        Index(value = ["user_id", "is_synced"])
    ]
)
data class WorkoutEntity(
    @PrimaryKey val id: String = UUID.randomUUID().toString(),
    @ColumnInfo(name = "user_id") val userId: String,
    val name: String,
    val description: String?,
    @ColumnInfo(name = "created_at") val createdAt: Long,
    @ColumnInfo(name = "last_modified") val lastModified: Long,
    
    // Sync metadata (required for all entities)
    @ColumnInfo(name = "is_synced") val isSynced: Boolean = false,
    @ColumnInfo(name = "sync_version") val syncVersion: Long = 0,
    @ColumnInfo(name = "conflict_resolution_timestamp") val conflictResolutionTimestamp: Long = 0
) : SyncableEntity {
    
    override fun withSyncStatus(isSynced: Boolean, version: Long): SyncableEntity {
        return copy(isSynced = isSynced, syncVersion = version)
    }
}
```

## Key Components & Classes

### 1. Navigation System

**Type-Safe Navigation with Compose**:
```kotlin
// Route definitions with serialization
@Serializable
sealed class LiftrixRoute {
    @Serializable data object Home : LiftrixRoute()
    @Serializable data object Profile : LiftrixRoute()
    @Serializable data object Feed : LiftrixRoute()
    
    @Serializable 
    data class WorkoutDetail(
        val workoutId: String,
        val isTemplate: Boolean = false
    ) : LiftrixRoute()
    
    @Serializable
    data class ExerciseSelection(
        val workoutId: String,
        val selectedCategory: String? = null
    ) : LiftrixRoute()
    
    @Serializable
    data class ProgressDetail(
        val widgetType: String,
        val timeRange: String,
        val exerciseIds: List<String> = emptyList()
    ) : LiftrixRoute()
    
    @Serializable
    data class PublicProfile(
        val profileUserId: String,
        val viewerId: String
    ) : LiftrixRoute()
}

// Navigation container implementation
@Composable
fun UnifiedNavigationContainer(
    startDestination: LiftrixRoute = LiftrixRoute.Home,
    modifier: Modifier = Modifier
) {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = startDestination,
        modifier = modifier
    ) {
        // Home navigation graph
        composable<LiftrixRoute.Home> {
            HomeScreen(
                onNavigateToWorkout = { workoutId ->
                    navController.navigate(LiftrixRoute.WorkoutDetail(workoutId))
                },
                onNavigateToProgress = { widgetType, timeRange ->
                    navController.navigate(
                        LiftrixRoute.ProgressDetail(widgetType, timeRange)
                    )
                }
            )
        }
        
        // Workout detail with parameters
        composable<LiftrixRoute.WorkoutDetail> { backStackEntry ->
            val route = backStackEntry.toRoute<LiftrixRoute.WorkoutDetail>()
            WorkoutDetailScreen(
                workoutId = route.workoutId,
                isTemplate = route.isTemplate,
                onNavigateBack = { navController.navigateUp() },
                onNavigateToExerciseSelection = { workoutId ->
                    navController.navigate(
                        LiftrixRoute.ExerciseSelection(workoutId)
                    )
                }
            )
        }
        
        // Progress analytics detail
        composable<LiftrixRoute.ProgressDetail> { backStackEntry ->
            val route = backStackEntry.toRoute<LiftrixRoute.ProgressDetail>()
            when (route.widgetType) {
                "volume_chart" -> VolumeAnalysisDetailScreen(
                    timeRange = TimeRange.fromString(route.timeRange),
                    onNavigateBack = { navController.navigateUp() }
                )
                "one_rm_progression" -> OneRmProgressionDetailScreen(
                    exerciseIds = route.exerciseIds,
                    timeRange = TimeRange.fromString(route.timeRange),
                    onNavigateBack = { navController.navigateUp() }
                )
                // Additional detail screens...
            }
        }
        
        // Social profile with viewer context
        composable<LiftrixRoute.PublicProfile> { backStackEntry ->
            val route = backStackEntry.toRoute<LiftrixRoute.PublicProfile>()
            PublicProfileScreen(
                profileUserId = route.profileUserId,
                viewerId = route.viewerId,
                onNavigateBack = { navController.navigateUp() }
            )
        }
    }
}
```

### 2. Sync System Architecture

**Coordinated Background Sync**:
```kotlin
// Sync coordinator managing all sync operations
@Singleton
class SyncCoordinator @Inject constructor(
    private val workManager: WorkManager,
    private val syncRepository: SyncRepository,
    private val networkMonitor: NetworkMonitor
) {
    
    fun schedulePeriodicSync(userId: String) {
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .setRequiresBatteryNotLow(true)
            .build()
        
        val periodicSyncRequest = PeriodicWorkRequestBuilder<MasterSyncWorker>(
            repeatInterval = 15,
            repeatIntervalTimeUnit = TimeUnit.MINUTES
        )
            .setConstraints(constraints)
            .setInputData(workDataOf(MasterSyncWorker.USER_ID_KEY to userId))
            .build()
        
        workManager.enqueueUniquePeriodicWork(
            "periodic_sync_$userId",
            ExistingPeriodicWorkPolicy.UPDATE,
            periodicSyncRequest
        )
    }
    
    fun triggerImmediateSync(userId: String) {
        val immediateSyncRequest = OneTimeWorkRequestBuilder<MasterSyncWorker>()
            .setInputData(workDataOf(MasterSyncWorker.USER_ID_KEY to userId))
            .build()
        
        workManager.enqueue(immediateSyncRequest)
    }
    
    fun queueWorkoutSync(userId: String, workoutId: String) {
        val workoutSyncRequest = OneTimeWorkRequestBuilder<WorkoutSyncWorker>()
            .setInputData(workDataOf(
                WorkoutSyncWorker.USER_ID_KEY to userId,
                WorkoutSyncWorker.WORKOUT_ID_KEY to workoutId
            ))
            .build()
        
        workManager.enqueue(workoutSyncRequest)
    }
    
    suspend fun syncWorkouts(userId: String): SyncResult {
        return if (networkMonitor.isOnline) {
            syncRepository.syncWorkouts(userId)
        } else {
            SyncResult.NoNetwork
        }
    }
}

// Specialized sync worker implementation
class WorkoutSyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val workoutRepository: SyncRepository,
    private val conflictResolver: ConflictResolver
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            val userId = inputData.getString(USER_ID_KEY) ?: return Result.failure()
            
            // Sync local changes to remote
            val localChanges = workoutRepository.getUnsyncedWorkouts(userId)
            localChanges.chunked(BATCH_SIZE).forEach { batch ->
                syncBatchToRemote(batch)
            }
            
            // Fetch remote changes
            val remoteChanges = workoutRepository.getRemoteChanges(userId)
            remoteChanges.forEach { remoteWorkout ->
                val localWorkout = workoutRepository.getLocalWorkout(userId, remoteWorkout.id)
                
                if (localWorkout != null && localWorkout.lastModified != remoteWorkout.lastModified) {
                    // Conflict resolution required
                    val resolvedWorkout = conflictResolver.resolve(localWorkout, remoteWorkout)
                    workoutRepository.updateLocalWorkout(resolvedWorkout)
                } else if (localWorkout == null) {
                    // New remote workout
                    workoutRepository.insertLocalWorkout(remoteWorkout)
                }
            }
            
            Result.success()
        } catch (exception: Exception) {
            Timber.e(exception, "Workout sync failed")
            if (runAttemptCount < MAX_RETRIES) {
                Result.retry()
            } else {
                Result.failure()
            }
        }
    }
    
    companion object {
        const val USER_ID_KEY = "user_id"
        const val WORKOUT_ID_KEY = "workout_id"
        const val BATCH_SIZE = 20
        const val MAX_RETRIES = 3
    }
    
    @AssistedFactory
    interface Factory {
        fun create(context: Context, params: WorkerParameters): WorkoutSyncWorker
    }
}
```

### 3. Design System Implementation

**Material 3 with Custom Color System**:
```kotlin
// LiftrixColorsV2 - Production color system
object LiftrixColorsV2 {
    // Primary colors
    val primary = Color(0xFF20C9B7)        // Teal
    val onPrimary = Color(0xFFFFFFFF)
    val primaryContainer = Color(0xFFB2F5EF)
    val onPrimaryContainer = Color(0xFF002B26)
    
    // Secondary colors
    val secondary = Color(0xFF2A3B7D)      // Indigo
    val onSecondary = Color(0xFFFFFFFF)
    val secondaryContainer = Color(0xFFDDE1FF)
    val onSecondaryContainer = Color(0xFF0E1A40)
    
    // Surface colors
    val surface = Color(0xFFFAFDFC)
    val onSurface = Color(0xFF191C1B)
    val surfaceVariant = Color(0xFFDBE8E6)
    val onSurfaceVariant = Color(0xFF3F4946)
    
    // Additional semantic colors
    val success = Color(0xFF4CAF50)
    val warning = Color(0xFFFF9800)
    val error = Color(0xFFE57373)
}

// Theme implementation
@Composable
fun LiftrixTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) {
        darkColorScheme(
            primary = LiftrixColorsV2.primary,
            onPrimary = LiftrixColorsV2.onPrimary,
            secondary = LiftrixColorsV2.secondary,
            onSecondary = LiftrixColorsV2.onSecondary
            // ... complete color mapping
        )
    } else {
        lightColorScheme(
            primary = LiftrixColorsV2.primary,
            onPrimary = LiftrixColorsV2.onPrimary,
            secondary = LiftrixColorsV2.secondary,
            onSecondary = LiftrixColorsV2.onSecondary
            // ... complete color mapping
        )
    }
    
    MaterialTheme(
        colorScheme = colorScheme,
        typography = LiftrixTypography,
        shapes = LiftrixShapes,
        content = content
    )
}

// Semantic spacing system
object LiftrixSpacing {
    val xs = 4.dp      // Minimal spacing
    val small = 8.dp   // Component internal spacing
    val medium = 12.dp // Standard component spacing
    val large = 16.dp  // Screen content spacing
    val xl = 24.dp     // Section spacing
    val xxl = 32.dp    // Major layout spacing
}

// Reusable component implementation
@Composable
fun UnifiedWorkoutCard(
    title: String,
    subtitle: String?,
    modifier: Modifier = Modifier,
    onClick: (() -> Unit)? = null,
    content: @Composable ColumnScope.() -> Unit = {}
) {
    Card(
        modifier = modifier
            .fillMaxWidth()
            .let { mod ->
                if (onClick != null) {
                    mod.clickable { onClick() }
                } else {
                    mod
                }
            },
        shape = RoundedCornerShape(12.dp),
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.surface
        ),
        elevation = CardDefaults.cardElevation(defaultElevation = 2.dp)
    ) {
        Column(
            modifier = Modifier.padding(LiftrixSpacing.large)
        ) {
            Text(
                text = title,
                style = MaterialTheme.typography.titleMedium,
                color = MaterialTheme.colorScheme.onSurface
            )
            
            if (subtitle != null) {
                Text(
                    text = subtitle,
                    style = MaterialTheme.typography.bodyMedium,
                    color = MaterialTheme.colorScheme.onSurfaceVariant,
                    modifier = Modifier.padding(top = LiftrixSpacing.xs)
                )
            }
            
            content()
        }
    }
}
```

## Testing Strategies

### Unit Testing Patterns

**ViewModel Testing with MockK**:
```kotlin
@ExtendWith(MockKExtension::class)
class WorkoutDetailsViewModelTest {
    
    @MockK private lateinit var getWorkoutUseCase: GetWorkoutUseCase
    @MockK private lateinit var updateWorkoutUseCase: UpdateWorkoutUseCase
    @MockK private lateinit var getCurrentUserIdUseCase: GetCurrentUserIdUseCase
    
    private lateinit var viewModel: WorkoutDetailsViewModel
    
    @BeforeEach
    fun setup() {
        viewModel = WorkoutDetailsViewModel(
            getWorkoutUseCase,
            updateWorkoutUseCase,
            getCurrentUserIdUseCase
        )
    }
    
    @Test
    fun `when loading workout, should emit loading then success state`() = runTest {
        // Given
        val workoutId = "workout123"
        val workout = createTestWorkout(id = workoutId)
        coEvery { getCurrentUserIdUseCase() } returns "user123"
        coEvery { getWorkoutUseCase(workoutId) } returns LiftrixResult.Success(workout)
        
        // When
        viewModel.handleEvent(WorkoutDetailsEvent.LoadWorkout(workoutId))
        
        // Then
        viewModel.uiState.test {
            assertEquals(UiState.Loading, awaitItem())
            
            val successState = awaitItem()
            assertTrue(successState is UiState.Success)
            assertEquals(workout, (successState as UiState.Success).data.workout)
        }
    }
    
    @Test
    fun `when workout update fails, should emit error state`() = runTest {
        // Given
        val workoutId = "workout123"
        val error = LiftrixError.NetworkError(message = "Network error")
        coEvery { getCurrentUserIdUseCase() } returns "user123"
        coEvery { updateWorkoutUseCase(any()) } returns LiftrixResult.Error(error)
        
        // When
        viewModel.handleEvent(WorkoutDetailsEvent.UpdateWorkout(workoutId))
        
        // Then
        viewModel.uiState.test {
            skipItems(1) // Skip loading state
            
            val errorState = awaitItem()
            assertTrue(errorState is UiState.Error)
            assertEquals(error, (errorState as UiState.Error).error)
        }
    }
}
```

### Repository Testing with Room

**Repository Integration Testing**:
```kotlin
@RunWith(AndroidJUnit4::class)
class WorkoutRepositoryTest {
    
    @get:Rule
    val instantTaskExecutorRule = InstantTaskExecutorRule()
    
    private lateinit var database: LiftrixDatabase
    private lateinit var workoutDao: WorkoutDao
    private lateinit var repository: WorkoutRepositoryImpl
    
    @Before
    fun setup() {
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            LiftrixDatabase::class.java
        ).allowMainThreadQueries().build()
        
        workoutDao = database.workoutDao()
        repository = WorkoutRepositoryImpl(
            localDataSource = workoutDao,
            remoteDataSource = mockk(),
            syncCoordinator = mockk(),
            workoutMapper = WorkoutMapper()
        )
    }
    
    @After
    fun teardown() {
        database.close()
    }
    
    @Test
    fun createWorkout_shouldSaveToLocalDatabase() = runTest {
        // Given
        val workout = createTestWorkout(userId = "user123")
        
        // When
        val result = repository.createWorkout(workout)
        
        // Then
        assertTrue(result is LiftrixResult.Success)
        
        val savedWorkouts = workoutDao.getWorkoutsForUser("user123")
        assertEquals(1, savedWorkouts.size)
        assertEquals(workout.name, savedWorkouts.first().name)
    }
    
    @Test
    fun getWorkoutsForUser_shouldReturnUserSpecificData() = runTest {
        // Given
        val user1Id = "user1"
        val user2Id = "user2"
        
        workoutDao.insertWorkout(createTestWorkoutEntity(userId = user1Id, name = "User 1 Workout"))
        workoutDao.insertWorkout(createTestWorkoutEntity(userId = user2Id, name = "User 2 Workout"))
        
        // When
        val user1Workouts = repository.getWorkoutsForUser(user1Id).first()
        val user2Workouts = repository.getWorkoutsForUser(user2Id).first()
        
        // Then
        assertEquals(1, user1Workouts.size)
        assertEquals(1, user2Workouts.size)
        assertEquals("User 1 Workout", user1Workouts.first().name)
        assertEquals("User 2 Workout", user2Workouts.first().name)
    }
}
```

## Code Quality Standards

### 1. Function Complexity Limits

**Maximum Function Size**: 20 instructions
```kotlin
// ✅ GOOD - Concise function with single responsibility
suspend fun createWorkout(request: CreateWorkoutRequest): LiftrixResult<Workout> = 
    liftrixCatching(errorMapper = ::mapCreateWorkoutError) {
        validateRequest(request)
        val userId = getCurrentUserId()
        val workout = request.toDomain(userId)
        workoutRepository.createWorkout(workout)
    }

// ❌ AVOID - Function too complex, needs refactoring
suspend fun processComplexWorkoutData(
    rawData: String,
    userId: String,
    options: ProcessingOptions
): LiftrixResult<ProcessedWorkout> {
    // 50+ lines of complex processing logic
    // Should be broken into smaller functions
}
```

### 2. Class Complexity Limits

**Maximum Class Size**: 200 instructions
```kotlin
// ✅ GOOD - Focused class with clear responsibility
class WorkoutValidator @Inject constructor() {
    
    fun validateWorkout(workout: Workout): ValidationResult {
        return ValidationResult.Builder()
            .validate { workout.name.isNotBlank() }
            .validate { workout.exercises.isNotEmpty() }
            .validate { workout.exercises.all { it.sets.isNotEmpty() } }
            .build()
    }
}

// ❌ AVOID - Class handling too many responsibilities
class MegaWorkoutManager {
    // 500+ lines handling validation, creation, sync, UI state, etc.
    // Should be decomposed into specialized classes
}
```

### 3. Documentation Standards

**KDoc Documentation Pattern**:
```kotlin
/**
 * Creates a new workout for the specified user with comprehensive validation and sync support.
 * 
 * This function implements offline-first architecture by immediately storing the workout
 * locally and queuing it for background synchronization with Firebase.
 * 
 * @param request The workout creation request containing exercises and metadata
 * @return LiftrixResult containing the created workout or error details
 * 
 * @throws SecurityException if user is not authenticated
 * @throws ValidationException if workout data is invalid
 * 
 * @see CreateWorkoutRequest for request structure requirements
 * @see WorkoutRepository.createWorkout for persistence implementation
 * 
 * @since 1.0.0
 */
suspend fun createWorkout(request: CreateWorkoutRequest): LiftrixResult<Workout>
```

This comprehensive development guide ensures consistent, maintainable, and high-quality code across the entire Liftrix codebase while supporting the application's sophisticated feature set and enterprise-grade architecture requirements.