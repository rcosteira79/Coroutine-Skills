---
name: android-data-layer
description: Use when implementing the data layer in Android — Repository pattern, Room local database, offline-first synchronization, and coordinating local and remote sources.
---

# Android Data Layer

The data layer coordinates data from multiple sources. Its public API to the rest of the app is repository interfaces; its internal implementation details (DAOs, API services, DTOs) never leak upward.

## Repository Pattern

The repository is the **single source of truth**. It decides whether to serve cached data or fetch fresh data, and maps raw data-layer types to domain models.

```kotlin
class NewsRepository @Inject constructor(
    private val newsDao: NewsDao,
    private val newsApi: NewsApi,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) {
    // Room DAO as the source of truth — UI always reads from local DB
    val newsStream: Flow<List<News>> = newsDao.getAllNews()

    // Triggered by UI or WorkManager to refresh data
    suspend fun refreshNews(): Result<Unit> = withContext(ioDispatcher) {
        runCatching {
            val remoteNews = newsApi.fetchLatest()
            newsDao.insertAll(remoteNews.map { it.toDomain() })
        }
    }
}
```

Bind the interface to its implementation in a Hilt module:

```kotlin
@Binds
abstract fun bindNewsRepository(impl: OfflineFirstNewsRepository): NewsRepository
```

---

## Room — Local Database

### Entity

```kotlin
@Entity(tableName = "articles")
data class ArticleEntity(
    @PrimaryKey val id: String,
    val title: String,
    val body: String,
    val publishedAt: Long
)
```

### DAO

Return `Flow<T>` for observable queries; `suspend fun` for one-shot reads and writes.

```kotlin
@Dao
interface ArticleDao {

    @Query("SELECT * FROM articles ORDER BY publishedAt DESC")
    fun observeAll(): Flow<List<ArticleEntity>>

    @Query("SELECT * FROM articles WHERE id = :id")
    suspend fun findById(id: String): ArticleEntity?

    @Upsert
    suspend fun upsertAll(articles: List<ArticleEntity>)

    @Query("DELETE FROM articles")
    suspend fun deleteAll()
}
```

### Database

```kotlin
@Database(entities = [ArticleEntity::class], version = 1, exportSchema = true)
abstract class AppDatabase : RoomDatabase() {
    abstract fun articleDao(): ArticleDao
}
```

Provide as a singleton via Hilt and export the schema for migration history tracking.

---

## Offline-First Strategies

### Read — Stale-While-Revalidate

Show local data immediately; trigger a background refresh in parallel.

```kotlin
// In ViewModel
fun loadNews() {
    viewModelScope.launch {
        // 1. Start observing the local DB immediately
        repository.newsStream
            .collect { articles -> _uiState.update { it.copy(articles = articles) } }
    }

    viewModelScope.launch {
        // 2. Trigger a network refresh in parallel
        repository.refreshNews().onFailure { error ->
            _uiState.update { it.copy(error = error.message) }
        }
    }
}
```

### Write — Outbox Pattern

Save changes locally first, then sync to the server. Use WorkManager to guarantee delivery even if the app is killed.

```kotlin
// 1. Mark item as unsynced in the DB immediately
suspend fun likeArticle(id: String) {
    articleDao.markAsUnsynced(id, action = "LIKE")
}

// 2. WorkManager job (runs when connected)
class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val articleDao: ArticleDao,
    private val newsApi: NewsApi
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val unsynced = articleDao.getUnsyncedActions()
        unsynced.forEach { action ->
            newsApi.postAction(action)
            articleDao.markAsSynced(action.id)
        }
        return Result.success()
    }
}
```

---

## Model Mapping

Keep three distinct model types and map between them at layer boundaries:

| Layer | Model Type | Purpose |
|-------|-----------|---------|
| Network | DTO (`ArticleDto`) | Matches API JSON structure |
| Database | Entity (`ArticleEntity`) | Matches Room table schema |
| Domain/UI | Domain model (`Article`) | What the rest of the app uses |

```kotlin
// DTO → Entity (in repository, before writing to DB)
fun ArticleDto.toEntity(): ArticleEntity = ArticleEntity(
    id = id,
    title = title,
    body = body,
    publishedAt = publishedAt
)

// Entity → Domain model (in repository, before returning to ViewModel)
fun ArticleEntity.toDomain(): Article = Article(
    id = id,
    title = title,
    body = body,
    publishedAt = Instant.ofEpochMilli(publishedAt)
)
```

---

## Checklist

- [ ] Repository exposes `Flow` for streams and `suspend fun` returning `Result<T>` for one-shot operations
- [ ] Raw DTOs and entities never reach the ViewModel — map at the repository boundary
- [ ] Room DAOs return `Flow` for observed queries; `suspend` for mutations
- [ ] Schema exported (`exportSchema = true`) and migration scripts provided for version bumps
- [ ] Offline-first: local DB is the source of truth; network writes go through an outbox if reliability matters
- [ ] WorkManager used for sync operations that must survive process death
