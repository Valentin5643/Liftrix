# Liftrix Dependencies & Integration Analysis

## Dependency Architecture Overview

Liftrix integrates 20+ carefully selected libraries following a strategic dependency management approach. Dependencies are categorized by functional domain and integrated through clean architecture boundaries to maintain modularity and testability.

## Core Framework Dependencies

### Android Architecture Components

**Jetpack Compose Ecosystem**:
```gradle
// UI Framework - Material 3 Design System
implementation 'androidx.compose.material3:material3:1.2.0-alpha10'
implementation 'androidx.compose.ui:ui:1.5.4'
implementation 'androidx.compose.ui:ui-tooling-preview:1.5.4'

// Navigation with Type Safety
implementation 'androidx.navigation:navigation-compose:2.7.5'

// Lifecycle & State Management
implementation 'androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0'
implementation 'androidx.lifecycle:lifecycle-runtime-compose:2.7.0'
```

**Integration Pattern**:
```kotlin
// Type-safe navigation with serializable routes
@Serializable
sealed class LiftrixRoute {
    @Serializable data object Home : LiftrixRoute()
    @Serializable 
    data class WorkoutDetail(val workoutId: String) : LiftrixRoute()
}

// Compose integration with lifecycle awareness
@Composable
fun WorkoutScreen(viewModel: WorkoutViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    LaunchedEffect(Unit) {
        viewModel.handleEvent(WorkoutEvent.LoadData)
    }
}
```

### Local Database Layer

**Room Database with Advanced Features**:
```gradle
// Core Room components
implementation 'androidx.room:room-runtime:2.6.0'
implementation 'androidx.room:room-ktx:2.6.0'
kapt 'androidx.room:room-compiler:2.6.0'

// Paging3 integration for social feeds
implementation 'androidx.room:room-paging:2.6.0'
```

**Integration Architecture**:
```kotlin
// User-scoped security pattern (mandatory)
@Dao
interface WorkoutDao {
    @Query("""
        SELECT * FROM workouts 
        WHERE user_id = :userId 
        ORDER BY created_at DESC
        LIMIT :limit OFFSET :offset
    """)
    fun getWorkoutsForUserPaged(
        userId: String, 
        limit: Int, 
        offset: Int
    ): PagingSource<Int, WorkoutEntity>
    
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

// 29 entities with sync metadata
@Entity(tableName = "workouts")
data class WorkoutEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "user_id") val userId: String,
    @ColumnInfo(name = "is_synced") val isSynced: Boolean = false,
    @ColumnInfo(name = "sync_version") val syncVersion: Long = 0,
    @ColumnInfo(name = "last_modified") val lastModified: Long = System.currentTimeMillis()
    // ... workout-specific fields
) : SyncableEntity
```

## Firebase Service Integrations

### Firebase Ecosystem (8 Services)

**Core Firebase Services**:
```gradle
// Firebase BOM for version alignment
implementation platform('com.google.firebase:firebase-bom:32.6.0')

// Authentication (Multi-provider)
implementation 'com.google.firebase:firebase-auth-ktx'

// Firestore (Offline-first database)
implementation 'com.google.firebase:firebase-firestore-ktx'

// Storage (Media upload with CDN)
implementation 'com.google.firebase:firebase-storage-ktx'

// Analytics & Performance
implementation 'com.google.firebase:firebase-analytics-ktx'
implementation 'com.google.firebase:firebase-perf-ktx'
implementation 'com.google.firebase:firebase-crashlytics-ktx'

// Remote Config (Feature flags)
implementation 'com.google.firebase:firebase-config-ktx'

// AI Integration (Gemini 2.5)
implementation 'com.google.firebase:firebase-vertexai:16.0.0-beta03'
```

**Firebase Integration Patterns**:

```kotlin
// Firestore with offline support and conflict resolution
class FirebaseWorkoutDataSource @Inject constructor(
    private val firestore: FirebaseFirestore
) {
    suspend fun syncWorkouts(userId: String, workouts: List<WorkoutEntity>): SyncResult {
        return try {
            val batch = firestore.batch()
            
            workouts.chunked(BATCH_SIZE).forEach { chunk ->
                chunk.forEach { workout ->
                    val docRef = firestore
                        .collection("users")
                        .document(userId)
                        .collection("workouts")
                        .document(workout.id)
                    
                    batch.set(docRef, workout.toFirestoreMap())
                }
                batch.commit().await()
            }
            
            SyncResult.Success
        } catch (exception: Exception) {
            SyncResult.Error(exception.toSyncError())
        }
    }
}

// Firebase AI integration with rate limiting
class AIChatService @Inject constructor(
    private val vertexAI: FirebaseVertexAI,
    private val rateLimitingService: RateLimitingService,
    private val abusePreventionService: AbusePreventionService
) {
    suspend fun sendMessage(
        message: String,
        conversationContext: List<ChatMessage>,
        userId: String
    ): LiftrixResult<ChatResponse> = liftrixCatching(
        errorMapper = { LiftrixError.AIError("Failed to process message") }
    ) {
        // Rate limiting (100 msg/day, 10k tokens/month)
        rateLimitingService.checkLimits(userId)
        
        // Abuse prevention with fitness context
        val abuseScore = abusePreventionService.analyzeMessage(message)
        if (abuseScore > THRESHOLD && !containsFitnessKeywords(message)) {
            throw AbuseDetectedException("Message flagged for policy violation")
        }
        
        val generativeModel = vertexAI.generativeModel("gemini-2.5")
        val response = generativeModel.generateContent(
            buildPrompt(message, conversationContext)
        ).await()
        
        ChatResponse(
            content = response.text ?: "",
            tokenCount = response.usageMetadata?.totalTokenCount ?: 0,
            language = detectLanguage(message)
        )
    }
}
```

### Authentication Integration

**Multi-Provider Authentication**:
```kotlin
// Google Sign-In integration
implementation 'com.google.android.gms:play-services-auth:20.7.0'

class AuthRepositoryImpl @Inject constructor(
    private val firebaseAuth: FirebaseAuth,
    private val googleSignInClient: GoogleSignInClient
) : AuthRepository {
    
    override suspend fun signInWithGoogle(): LiftrixResult<User> = liftrixCatching(
        errorMapper = { throwable ->
            when (throwable) {
                is GoogleSignInException -> LiftrixError.AuthenticationError(
                    reason = AuthFailureReason.GOOGLE_SIGN_IN_FAILED
                )
                else -> LiftrixError.NetworkError(
                    message = "Authentication service unavailable"
                )
            }
        }
    ) {
        val signInIntent = googleSignInClient.signInIntent
        // Intent handling via Activity Result API
        val result = handleGoogleSignInResult()
        val credential = GoogleAuthProvider.getCredential(result.idToken, null)
        val authResult = firebaseAuth.signInWithCredential(credential).await()
        
        authResult.user?.toUser() ?: throw UserNotFoundException()
    }
}
```

## Specialized Feature Dependencies

### Social Features & Media Processing

**Paging3 for Feed Performance**:
```gradle
// Paging3 with Compose integration
implementation 'androidx.paging:paging-runtime:3.2.1'
implementation 'androidx.paging:paging-compose:3.2.1'
```

**Implementation Pattern**:
```kotlin
@Composable
fun SocialFeedScreen(viewModel: FeedViewModel = hiltViewModel()) {
    val posts = viewModel.feedPosts.collectAsLazyPagingItems()
    
    LazyColumn {
        items(
            count = posts.itemCount,
            key = posts.itemKey { it.id }
        ) { index ->
            val post = posts[index]
            if (post != null) {
                WorkoutPostCard(
                    post = post,
                    onLikeClick = { viewModel.handleEvent(FeedEvent.ToggleLike(post.id)) },
                    onCommentClick = { /* Navigate to comments */ }
                )
            }
        }
    }
}

// Feed data source with RemoteMediator
class FeedRemoteMediator @Inject constructor(
    private val database: LiftrixDatabase,
    private val feedApi: FeedApiService,
    private val privacyService: PrivacyEnforcementService
) : RemoteMediator<Int, WorkoutPostEntity>() {
    
    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, WorkoutPostEntity>
    ): MediatorResult {
        // Complex privacy-aware loading logic
        val viewerId = getCurrentUserId()
        val posts = feedApi.getFeedPage(viewerId, loadType.toPageKey())
        
        // Privacy filtering before local storage
        val visiblePosts = posts.filter { post ->
            privacyService.canViewPost(viewerId, post)
        }
        
        database.withTransaction {
            if (loadType == LoadType.REFRESH) {
                database.feedDao().clearFeed(viewerId)
            }
            database.feedDao().insertPosts(visiblePosts)
        }
        
        return MediatorResult.Success(endOfPaginationReached = posts.isEmpty())
    }
}
```

### QR Code Integration

**ZXing for Gym Buddy Pairing**:
```gradle
// QR code generation and scanning
implementation 'com.google.zxing:core:3.5.2'
implementation 'com.journeyapps:zxing-android-embedded:4.3.0'
```

**Integration Implementation**:
```kotlin
class QRCodeService @Inject constructor() {
    
    fun generateGymBuddyQR(
        userId: String,
        size: Int = 300
    ): LiftrixResult<Bitmap> = liftrixCatching(
        errorMapper = { LiftrixError.QRGenerationError("Failed to generate QR code") }
    ) {
        val expiryTime = System.currentTimeMillis() + TimeUnit.MINUTES.toMillis(5)
        val pairingData = "liftrix://gym-buddy/$userId?expires=$expiryTime"
        
        val writer = QRCodeWriter()
        val bitMatrix = writer.encode(
            pairingData,
            BarcodeFormat.QR_CODE,
            size,
            size
        )
        
        val bitmap = Bitmap.createBitmap(size, size, Bitmap.Config.RGB_565)
        for (x in 0 until size) {
            for (y in 0 until size) {
                bitmap.setPixel(
                    x, y, 
                    if (bitMatrix[x, y]) Color.BLACK else Color.WHITE
                )
            }
        }
        bitmap
    }
    
    suspend fun validateQRCode(qrData: String): LiftrixResult<GymBuddyPairingData> {
        // Validation logic with expiry checking
        val uri = Uri.parse(qrData)
        val userId = uri.pathSegments.lastOrNull() 
            ?: return LiftrixResult.Error(LiftrixError.QRValidationError("Invalid QR format"))
        
        val expiryTime = uri.getQueryParameter("expires")?.toLongOrNull()
            ?: return LiftrixResult.Error(LiftrixError.QRValidationError("Missing expiry"))
        
        if (System.currentTimeMillis() > expiryTime) {
            return LiftrixResult.Error(LiftrixError.QRExpiredError("QR code expired"))
        }
        
        return LiftrixResult.Success(GymBuddyPairingData(userId, expiryTime))
    }
}
```

### Chart & Data Visualization

**Custom Chart Implementation** (No External Chart Library):
```kotlin
// Pure Compose implementation for 60fps performance
@Composable
fun ModernVolumeChart(
    data: List<VolumeDataPoint>,
    modifier: Modifier = Modifier,
    showPersonalRecords: Boolean = true
) {
    val animatedData by animateFloatAsState(
        targetValue = if (data.isNotEmpty()) 1f else 0f,
        animationSpec = tween(durationMillis = 300)
    )
    
    Canvas(modifier = modifier.fillMaxSize()) {
        val path = Path()
        val controlPoints = calculateBezierControlPoints(data)
        
        // Draw bezier curve with gradient fill
        drawPath(
            path = createBezierPath(data, controlPoints, size),
            color = LiftrixColorsV2.primary,
            style = Stroke(width = 3.dp.toPx())
        )
        
        // Draw gradient fill
        drawPath(
            path = createFillPath(data, controlPoints, size),
            brush = Brush.verticalGradient(
                colors = listOf(
                    LiftrixColorsV2.primary.copy(alpha = 0.3f),
                    Color.Transparent
                )
            )
        )
        
        // Highlight personal records
        if (showPersonalRecords) {
            data.filter { it.isPersonalRecord }.forEach { pr ->
                drawCircle(
                    color = LiftrixColorsV2.success,
                    radius = 8.dp.toPx(),
                    center = pr.position
                )
            }
        }
    }
}
```

## Background Processing Dependencies

### WorkManager for Reliable Sync

**WorkManager with Hilt Integration**:
```gradle
// WorkManager for background sync
implementation 'androidx.work:work-runtime-ktx:2.9.0'
implementation 'androidx.hilt:hilt-work:1.1.0'
kapt 'androidx.hilt:hilt-compiler:1.1.0'
```

**Integration Pattern**:
```kotlin
// MasterSyncWorker coordinating 10 specialized workers
class MasterSyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val syncCoordinator: SyncCoordinator,
    private val getCurrentUserIdUseCase: GetCurrentUserIdUseCase
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            val userId = getCurrentUserIdUseCase() 
                ?: return Result.failure()
            
            // Parallel sync execution for performance
            val syncResults = listOf(
                async { syncCoordinator.syncWorkouts(userId) },
                async { syncCoordinator.syncSocialProfiles(userId) },
                async { syncCoordinator.syncFollowRelationships(userId) },
                async { syncCoordinator.syncEngagements(userId) },
                async { syncCoordinator.syncAchievements(userId) }
                // ... additional sync operations
            ).awaitAll()
            
            if (syncResults.all { it.isSuccess }) {
                Result.success()
            } else {
                Result.retry()
            }
        } catch (exception: Exception) {
            Timber.e(exception, "Master sync failed")
            if (runAttemptCount < MAX_RETRIES) {
                Result.retry()
            } else {
                Result.failure()
            }
        }
    }
    
    @AssistedFactory
    interface Factory {
        fun create(context: Context, params: WorkerParameters): MasterSyncWorker
    }
}

// Hilt WorkManager integration
@Module
@InstallIn(SingletonComponent::class)
object WorkManagerModule {
    
    @Provides
    @Singleton
    fun provideWorkManager(@ApplicationContext context: Context): WorkManager =
        WorkManager.getInstance(context)
    
    @Binds
    @IntoMap
    @StringKey("MasterSyncWorker")
    abstract fun bindMasterSyncWorker(factory: MasterSyncWorker.Factory): ChildWorkerFactory
}
```

## Dependency Injection Framework

### Hilt with 22 Modules

**Core Hilt Configuration**:
```gradle
// Hilt dependency injection
implementation 'com.google.dagger:hilt-android:2.48'
kapt 'com.google.dagger:hilt-compiler:2.48'
implementation 'androidx.hilt:hilt-navigation-compose:1.1.0'
```

**Module Organization Strategy**:
```kotlin
// Database Module
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): LiftrixDatabase
    
    @Provides
    fun provideWorkoutDao(database: LiftrixDatabase): WorkoutDao = database.workoutDao()
    
    // ... 28 DAO providers
}

// Repository Module (16 repositories)
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    @Singleton
    abstract fun bindWorkoutRepository(impl: WorkoutRepositoryImpl): WorkoutRepository
    
    @Binds
    @Singleton
    abstract fun bindSyncRepository(impl: SyncRepositoryImpl): SyncRepository
    
    // ... 14 additional repository bindings
}

// Use Case Module (50+ use cases)
@Module
@InstallIn(ViewModelComponent::class)
abstract class UseCaseModule {
    @Binds
    abstract fun bindCreateWorkoutUseCase(impl: CreateWorkoutUseCaseImpl): CreateWorkoutUseCase
    
    @Binds
    abstract fun bindGetWorkoutsUseCase(impl: GetWorkoutsUseCaseImpl): GetWorkoutsUseCase
    
    // ... 48 additional use case bindings
}
```

## Testing Dependencies

### Comprehensive Testing Framework

**Unit Testing Stack**:
```gradle
// Core testing
testImplementation 'junit:junit:4.13.2'
testImplementation 'org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3'

// Mocking framework
testImplementation 'io.mockk:mockk:1.13.8'
testImplementation 'org.mockito:mockito-core:4.11.0'

// Flow testing
testImplementation 'app.cash.turbine:turbine:1.0.0'

// Room testing
testImplementation 'androidx.room:room-testing:2.6.0'
```

**Instrumentation Testing**:
```gradle
// Android testing
androidTestImplementation 'androidx.test.ext:junit:1.1.5'
androidTestImplementation 'androidx.test.espresso:espresso-core:3.5.1'

// Compose testing
androidTestImplementation 'androidx.compose.ui:ui-test-junit4:1.5.4'
androidTestImplementation 'androidx.compose.ui:ui-test-manifest:1.5.4'

// Hilt testing
androidTestImplementation 'com.google.dagger:hilt-android-testing:2.48'
kaptAndroidTest 'com.google.dagger:hilt-compiler:2.48'
```

## Version Management Strategy

### BOM Usage for Version Alignment

```gradle
// Platform BOMs for consistent versioning
implementation platform('androidx.compose:compose-bom:2023.10.01')
implementation platform('com.google.firebase:firebase-bom:32.6.0')

// Individual dependencies without version numbers (managed by BOM)
implementation 'androidx.compose.ui:ui'
implementation 'androidx.compose.material3:material3'
implementation 'com.google.firebase:firebase-firestore-ktx'
implementation 'com.google.firebase:firebase-auth-ktx'
```

### Dependency Conflict Resolution

```gradle
// Force specific versions for conflict resolution
configurations.all {
    resolutionStrategy {
        force 'org.jetbrains.kotlin:kotlin-stdlib:1.9.0'
        force 'androidx.lifecycle:lifecycle-viewmodel:2.7.0'
    }
}
```

## Security Considerations

### Dependency Security Analysis

**Vulnerability Scanning Integration**:
- **Regular dependency audits** using Gradle dependency-check plugin
- **Firebase Security Rules** enforcing user-level access controls
- **ProGuard/R8 obfuscation** for release builds protecting against reverse engineering
- **Certificate pinning** for secure network communications

**Critical Security Dependencies**:
```gradle
// Security-focused dependencies
implementation 'androidx.security:security-crypto:1.1.0-alpha06'  // Encrypted SharedPreferences
implementation 'com.google.crypto.tink:tink-android:1.7.0'        // Additional cryptography
```


This comprehensive dependency strategy ensures Liftrix maintains enterprise-grade reliability while leveraging best-in-class Android development tools and frameworks.
