# Liftrix System Architecture

## Architectural Overview

Liftrix implements a sophisticated multi-layered architecture based on Clean Architecture principles with MVVM presentation patterns and offline-first design. The system is designed for enterprise-scale reliability while maintaining modern Android development best practices.

## High-Level System Design

```
┌─────────────────────────────────────────────────────────────┐
│                    UI Layer (Jetpack Compose)              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │     Screens     │  │   ViewModels    │  │ Navigation  │ │
│  │   (30+ routes)  │  │ BaseViewModel   │  │ Type-Safe   │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              ↓ StateFlow<UiState<T>>
┌─────────────────────────────────────────────────────────────┐
│                  ViewModel Layer (MVI Pattern)             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │  State Mgmt     │  │  Event Handler  │  │ Coordinator │ │
│  │ UiState<T>      │  │ Sealed Events   │  │   Pattern   │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              ↓ LiftrixResult<T>
┌─────────────────────────────────────────────────────────────┐
│                Use Case Layer (50+ Operations)             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │ Business Logic  │  │ Error Handling  │  │ Validation  │ │
│  │  Single Resp.   │  │ LiftrixResult   │  │  Rules DSL  │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              ↓ LiftrixResult<T>
┌─────────────────────────────────────────────────────────────┐
│              Repository Layer (16 Interfaces)              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │ Data Contracts  │  │  Sync Mgmt      │  │ Cache Layer │ │
│  │ Domain Models   │  │ Conflict Res.   │  │ Performance │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              ↓ Flow<Entity>
┌─────────────────────────────────────────────────────────────┐
│                   DAO Layer (28 DAOs)                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │   User Scoped   │  │   Sync Status   │  │  Indexing   │ │
│  │ Security Layer  │  │   Metadata      │  │ Performance │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              ↓ SQL Queries
┌─────────────────────────────────────────────────────────────┐
│              Room Database (29 Entities, v1)               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │ Local Storage   │  │  Sync Metadata  │  │ Migrations  │ │
│  │ Source of Truth │  │ Conflict Track  │  │  Schema     │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              ↓ Background Sync
┌─────────────────────────────────────────────────────────────┐
│          Firebase Services (8 Integrated Services)         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │  Firestore DB   │  │   Storage CDN   │  │  AI/Auth    │ │
│  │ Real-time Sync  │  │  Media Upload   │  │ Analytics   │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Core Architectural Patterns

### 1. Clean Architecture Implementation

**Dependency Rule Enforcement**:
- **UI Layer**: Depends only on ViewModels
- **ViewModel Layer**: Depends only on Use Cases
- **Use Case Layer**: Depends only on Repository interfaces
- **Repository Layer**: Implements domain contracts, depends on DAOs
- **DAO Layer**: Depends only on database entities

**Layer Responsibilities**:

```kotlin
// UI Layer - Presentation Logic Only
@Composable
fun WorkoutScreen(viewModel: WorkoutViewModel) {
    val uiState by viewModel.uiState.collectAsState()
    
    when (uiState) {
        is UiState.Loading -> LoadingIndicator()
        is UiState.Success -> WorkoutContent(uiState.data)
        is UiState.Error -> ErrorMessage(uiState.error)
    }
}

// ViewModel Layer - State Management
class WorkoutViewModel @Inject constructor(
    private val getWorkoutsUseCase: GetWorkoutsUseCase
) : BaseViewModel<WorkoutUiState, WorkoutEvent>() {
    
    override fun handleEvent(event: WorkoutEvent) {
        when (event) {
            is WorkoutEvent.LoadWorkouts -> loadWorkouts()
        }
    }
}

// Use Case Layer - Business Logic
class GetWorkoutsUseCase @Inject constructor(
    private val workoutRepository: WorkoutRepository
) {
    suspend operator fun invoke(userId: String): LiftrixResult<List<Workout>> = 
        liftrixCatching(
            errorMapper = { LiftrixError.BusinessLogicError(/*...*/) }
        ) {
            workoutRepository.getWorkoutsForUser(userId)
        }
}
```

### 2. MVVM with MVI Elements

**State Management Pattern**:
```kotlin
// Base ViewModel with MVI Pattern
abstract class BaseViewModel<S : Any, E : Any> : ViewModel() {
    private val _uiState = MutableStateFlow<UiState<S>>(UiState.Loading)
    val uiState: StateFlow<UiState<S>> = _uiState.asStateFlow()
    
    abstract fun handleEvent(event: E)
    
    protected fun updateState(transform: (S) -> S) {
        val currentState = _uiState.value
        if (currentState is UiState.Success) {
            _uiState.value = UiState.Success(transform(currentState.data))
        }
    }
}

// UiState Sealed Class
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val error: LiftrixError) : UiState<Nothing>
    data object Empty : UiState<Nothing>
}
```

## Data Architecture

### 1. Offline-First Design

**Room as Source of Truth**:
```kotlin
// Repository Implementation Pattern
@Singleton
class WorkoutRepositoryImpl @Inject constructor(
    private val localDataSource: WorkoutDao,
    private val remoteDataSource: FirebaseWorkoutDataSource,
    private val syncCoordinator: SyncCoordinator
) : WorkoutRepository {
    
    override suspend fun getWorkoutsForUser(userId: String): Flow<List<Workout>> {
        // Always return local data first
        return localDataSource.getWorkoutsForUser(userId)
            .map { entities -> entities.map { it.toDomain() } }
            .onStart {
                // Trigger background sync
                syncCoordinator.syncWorkouts(userId)
            }
    }
}
```

### 2. User Scoping Security

**Mandatory User Filtering**:
```kotlin
// DAO Layer - All queries MUST include userId
@Dao
interface WorkoutDao {
    @Query("""
        SELECT * FROM workouts 
        WHERE user_id = :userId 
        ORDER BY created_at DESC
    """)
    suspend fun getWorkoutsForUser(userId: String): List<WorkoutEntity>
    
    @Query("""
        UPDATE workouts 
        SET is_synced = :isSynced 
        WHERE user_id = :userId AND id = :workoutId
    """)
    suspend fun updateSyncStatus(userId: String, workoutId: String, isSynced: Boolean)
}
```

### 3. Synchronization Architecture

**Multi-Layer Sync System**:
```
┌─────────────────────────────────────────────────────────────┐
│                  Sync Coordination Layer                   │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │ SyncCoordinator │  │ MasterSyncWorker│  │ Real-time   │ │
│  │   Orchestrator  │  │  Periodic Sync  │  │  Listeners  │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│              Specialized Sync Workers (10 Types)           │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │ WorkoutSyncWkr  │  │ SocialSyncWkr   │  │ ProfileSync │ │
│  │ TemplateSyncWkr │  │ EngagementSync  │  │ SettingSync │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                 Conflict Resolution Layer                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │ Last Write Wins │  │   Timestamp     │  │ Merge Logic │ │
│  │   Resolution    │  │  Comparison     │  │ Strategies  │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Sync Implementation Example**:
```kotlin
class WorkoutSyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val workoutRepository: SyncRepository,
    private val conflictResolver: ConflictResolver
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            val userId = inputData.getString(USER_ID_KEY) ?: return Result.failure()
            
            // Sync unsynced local changes to remote
            syncLocalChangesToRemote(userId)
            
            // Fetch remote changes and resolve conflicts
            syncRemoteChangesToLocal(userId)
            
            Result.success()
        } catch (exception: Exception) {
            if (runAttemptCount < MAX_RETRIES) {
                Result.retry()
            } else {
                Result.failure()
            }
        }
    }
    
    private suspend fun syncLocalChangesToRemote(userId: String) {
        val unsyncedWorkouts = workoutRepository.getUnsyncedWorkouts(userId)
        unsyncedWorkouts.chunked(BATCH_SIZE).forEach { batch ->
            remoteDataSource.batchUpdate(batch)
            localDataSource.markAsSynced(batch.map { it.id })
        }
    }
}
```

## Error Handling Architecture

### LiftrixResult<T> Pattern

**Comprehensive Error Handling**:
```kotlin
// Sealed Error Classes
sealed class LiftrixError {
    data class NetworkError(
        val httpCode: Int,
        val message: String,
        val isRetryable: Boolean = true
    ) : LiftrixError()
    
    data class ValidationError(
        val field: String,
        val violations: List<String>,
        val analyticsContext: Map<String, String>
    ) : LiftrixError()
    
    data class BusinessLogicError(
        val code: String,
        val errorMessage: String,
        val operation: String? = null,
        val analyticsContext: Map<String, String> = emptyMap()
    ) : LiftrixError()
    
    data class AuthenticationError(
        val reason: AuthFailureReason,
        val isRecoverable: Boolean = true
    ) : LiftrixError()
}

// Result Wrapper with Recovery
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
}

// Usage Pattern in Use Cases
suspend fun invoke(request: CreateWorkoutRequest): LiftrixResult<Workout> = 
    liftrixCatching(
        errorMapper = { throwable ->
            when (throwable) {
                is IllegalArgumentException -> LiftrixError.ValidationError(
                    field = "workout_data",
                    violations = listOf("Invalid workout structure")
                )
                else -> LiftrixError.BusinessLogicError(
                    code = "WORKOUT_CREATION_FAILED",
                    errorMessage = "Failed to create workout"
                )
            }
        }
    ) {
        // Business logic implementation
        val validatedWorkout = validateWorkout(request)
        workoutRepository.createWorkout(validatedWorkout)
    }
```

## Navigation Architecture

### Type-Safe Navigation System

**Serializable Route Pattern**:
```kotlin
@Serializable
sealed class LiftrixRoute {
    @Serializable data object Home : LiftrixRoute()
    @Serializable data object Profile : LiftrixRoute()
    
    @Serializable 
    data class WorkoutDetail(
        val workoutId: String,
        val isTemplate: Boolean = false
    ) : LiftrixRoute()
    
    @Serializable
    data class ProgressDetail(
        val widgetType: String,
        val timeRange: String,
        val exerciseIds: List<String> = emptyList()
    ) : LiftrixRoute()
    
    @Serializable
    data class PublicProfile(
        val userId: String,
        val viewerId: String
    ) : LiftrixRoute()
}

// Navigation Container Implementation
@Composable
fun UnifiedNavigationContainer() {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = LiftrixRoute.Home
    ) {
        composable<LiftrixRoute.Home> {
            HomeScreen(
                onNavigateToWorkout = { workoutId ->
                    navController.navigate(
                        LiftrixRoute.WorkoutDetail(workoutId)
                    )
                }
            )
        }
        
        composable<LiftrixRoute.WorkoutDetail> { backStackEntry ->
            val route = backStackEntry.toRoute<LiftrixRoute.WorkoutDetail>()
            WorkoutDetailScreen(
                workoutId = route.workoutId,
                isTemplate = route.isTemplate
            )
        }
    }
}
```

## Dependency Injection Architecture

### Hilt Module Organization

**22-Module Structure**:
```kotlin
// Core Infrastructure Modules
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): LiftrixDatabase =
        Room.databaseBuilder(context, LiftrixDatabase::class.java, "liftrix.db")
            .addMigrations(MIGRATION_1_2)
            .build()
}

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideFirebaseFirestore(): FirebaseFirestore = Firebase.firestore
    
    @Provides
    @Singleton
    fun provideFirebaseAuth(): FirebaseAuth = Firebase.auth
}

// Domain Layer Modules
@Module
@InstallIn(ViewModelComponent::class)
abstract class UseCaseModule {
    @Binds
    abstract fun bindGetWorkoutsUseCase(
        impl: GetWorkoutsUseCaseImpl
    ): GetWorkoutsUseCase
    
    @Binds
    abstract fun bindCreateWorkoutUseCase(
        impl: CreateWorkoutUseCaseImpl
    ): CreateWorkoutUseCase
}

// Repository Layer Modules
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    @Singleton
    abstract fun bindWorkoutRepository(
        impl: WorkoutRepositoryImpl
    ): WorkoutRepository
    
    @Binds
    @Singleton
    abstract fun bindSyncRepository(
        impl: SyncRepositoryImpl
    ): SyncRepository
}
```

## Performance Architecture

### Memory-Aware Components

**Adaptive Widget System**:
```kotlin
@Composable
fun AdaptiveWidgetGrid(
    widgets: List<WidgetData>,
    modifier: Modifier = Modifier
) {
    val memoryInfo = LocalContext.current.getMemoryInfo()
    val maxWidgets = when {
        memoryInfo.lowMemory -> 6
        memoryInfo.availMem < 100.MB -> 8
        else -> 12
    }
    
    LazyVerticalGrid(
        columns = GridCells.Adaptive(minSize = 160.dp),
        modifier = modifier
    ) {
        items(widgets.take(maxWidgets)) { widget ->
            WidgetContainer(
                widget = widget,
                onInteraction = { /* Handle interaction */ }
            )
        }
    }
}
```

### Chart Performance Optimization

**60fps Rendering System**:
```kotlin
@Composable
fun ModernVolumeChart(
    data: List<DataPoint>,
    modifier: Modifier = Modifier
) {
    val processedData = remember(data) {
        // Expensive processing cached with remember
        ChartDataProcessor.processVolumeData(data)
    }
    
    Canvas(modifier = modifier) {
        // Optimized drawing operations
        drawBezierCurve(
            points = processedData.points,
            color = LiftrixColorsV2.primary,
            strokeWidth = 3.dp.toPx()
        )
        
        // Gradient fill with hardware acceleration
        drawGradientFill(
            path = processedData.fillPath,
            gradient = processedData.gradient
        )
    }
}
```

This architecture ensures scalability, maintainability, and performance while providing a robust foundation for enterprise-grade Android development.