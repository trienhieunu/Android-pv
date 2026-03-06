---
ID: 001
Date: 2026-03-06
Tags: clean-architecture, repository, data-sources, remote-local, cache-strategy, single-responsibility
---

# Repository Pattern - Remote/Local Data Strategy

## Câu hỏi

Trong Clean Architecture, Repository thường abstract data sources (remote + local). Nếu implement như: `repository.getData()` first checks local DB, if empty then calls API and saves to DB. Vấn đề gì với cách này khi user làm refresh gesture mà API đang failed?

## Câu trả lời

### Giải đáp gốc rễ

- **Repository là abstraction layer**, không nên có business logic phức tạp
- "Check local then fetch remote" tạo ra **implicit dependency** giữa data sources
- Khi API fail, stale data trong local DB vẫn được return
- User không biết data đã cũ, dẫn đến **UX kém và confused**
- **Bẫy:** Không có cách để distinguish giữa "no cached data" và "API failed"
- Violate **single responsibility principle** của Repository

### Trả lời theo kiểu phỏng vấn

> "Vấn đề này là Repository không nên decide data strategy - đó là responsibility của use case hoặc domain layer. Repository chỉ provide raw data access: getAll, insert, delete. Use case mới là nơi orchestrate logic: nếu cache stale thì fetch remote, nếu API fail thì show error hoặc dùng stale data. Với refresh gesture, use case có thể bypass cache và force API call, hoặc invalidate cache trước. Tách biệt này giúp test dễ hơn, và use case có thể change strategy mà không sửa Repository. Tôi thường implement Repository chỉ là wrapper cho data sources, và đặt business logic trong use case hoặc ViewModel."

### Giải pháp

1. **Repository chỉ là wrapper** - provide raw data access (getAll, insert, delete)
2. **Use case/ViewModel orchestrate data strategy** - quyết định cache/remote logic
3. **Result object với status** - distinguish giữa success, error, stale data
4. **Force refresh parameter** - use case có thể bypass cache khi cần
5. **Separate concerns** - dễ test và maintain

## Tình huống ví dụ

### Code có vấn đề

```kotlin
// Repository ❌ BAD - có business logic
class UserRepository @Inject constructor(
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource: UserLocalDataSource
) {
    suspend fun getUsers(): List<User> {
        // ⚠️ Business logic trong Repository
        val cached = localDataSource.getUsers()
        if (cached.isNotEmpty()) {
            return cached // Return stale data!
        }

        try {
            val remote = remoteDataSource.getUsers()
            localDataSource.saveUsers(remote)
            return remote
        } catch (e: Exception) {
            return cached // Vấn đề: user không biết API failed
        }
    }
}

// Use case
class GetUsersUseCase @Inject constructor(
    private val repository: UserRepository
) {
    suspend operator fun invoke(): List<User> {
        return repository.getUsers() // ❌ Không control được strategy
    }
}

// ViewModel
class UserViewModel @Inject constructor(
    private val getUsersUseCase: GetUsersUseCase
) : ViewModel() {
    fun loadUsers() {
        viewModelScope.launch {
            val users = getUsersUseCase()
            _users.value = users // ❌ User không biết data stale hay fresh
        }
    }

    fun refreshUsers() {
        viewModelScope.launch {
            // ⚠️ Refresh vẫn dùng cache nếu có!
            val users = getUsersUseCase()
            _users.value = users
        }
    }
}
```

### Vấn đề trong kịch bản refresh

```
User mở app → load data từ local DB (cache)
User làm refresh gesture
  → API đang failed (network error, server down)
  → Repository return stale data từ local DB
  → ViewModel update UI với data cũ
  → User nghĩ refresh đã complete, nhưng thực tế vẫn là data cũ!
```

**Vấn đề:**
- User không biết refresh thành công hay thất bại
- UI không hiển thị error state
- Data có thể rất cũ (từ lần load đầu tiên)
- User confused và mất niềm tin vào app

### Code giải pháp

#### Cách 1: Repository chỉ wrapper, Use case orchestrate

```kotlin
// Repository ✅ GOOD - chỉ wrapper
class UserRepository @Inject constructor(
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource: UserLocalDataSource
) {
    suspend fun getRemoteUsers(): List<User> = remoteDataSource.getUsers()
    suspend fun getLocalUsers(): List<User> = localDataSource.getUsers()
    suspend fun saveUsers(users: List<User>) = localDataSource.saveUsers(users)
}

// Result object để track status
sealed class Result<out T> {
    data class Success<T>(val data: T, val source: DataSource) : Result<T>()
    data class Error(val exception: Exception, val cachedData: T? = null) : Result<Nothing>()
}

enum class DataSource {
    REMOTE,
    LOCAL
}

// Use case ✅ GOOD - orchestrate strategy
class GetUsersUseCase @Inject constructor(
    private val repository: UserRepository
) {
    suspend operator fun invoke(forceRefresh: Boolean = false): Result<List<User>> {
        return try {
            if (forceRefresh) {
                // Bypass cache, force fetch remote
                val remote = repository.getRemoteUsers()
                repository.saveUsers(remote)
                Result.Success(remote, DataSource.REMOTE)
            } else {
                // Check cache first
                val local = repository.getLocalUsers()
                if (local.isNotEmpty()) {
                    Result.Success(local, DataSource.LOCAL)
                } else {
                    // Cache empty, fetch remote
                    val remote = repository.getRemoteUsers()
                    repository.saveUsers(remote)
                    Result.Success(remote, DataSource.REMOTE)
                }
            }
        } catch (e: Exception) {
            // Return error với cached data nếu có
            val cached = repository.getLocalUsers()
            Result.Error(e, cached.takeIf { it.isNotEmpty() })
        }
    }
}

// ViewModel ✅ GOOD - handle UI state
class UserViewModel @Inject constructor(
    private val getUsersUseCase: GetUsersUseCase
) : ViewModel() {
    private val _users = mutableStateOf<Result<List<User>>>(Result.Success(emptyList(), DataSource.LOCAL))
    val users: StateFlow<Result<List<User>>> = _users

    fun loadUsers() {
        viewModelScope.launch {
            _users.value = getUsersUseCase(forceRefresh = false)
        }
    }

    fun refreshUsers() {
        viewModelScope.launch {
            _users.value = getUsersUseCase(forceRefresh = true)
        }
    }
}

// UI ✅ GOOD - show appropriate state based on Result
@Composable
fun UserScreen(viewModel: UserViewModel) {
    val users by viewModel.users.collectAsState()

    when (users) {
        is Result.Success -> {
            val data = (users as Result.Success<List<User>>).data
            val source = (users as Result.Success<List<User>>).source

            LazyColumn {
                items(data) { user ->
                    UserItem(user, source)
                }
            }

            // Show indicator về data source
            if (source == DataSource.LOCAL) {
                Text("⚠️ Loading from cache", color = Color.Gray)
            }
        }
        is Result.Error -> {
            val exception = (users as Result.Error).exception
            val cachedData = (users as Result.Error).cachedData

            if (cachedData != null) {
                // Show stale data với error indicator
                LazyColumn {
                    items(cachedData) { user ->
                        UserItem(user, DataSource.LOCAL)
                    }
                }
                Text(
                    "⚠️ Network failed. Showing cached data.",
                    color = Color.Red
                )
            } else {
                // Show error screen
                Text("Error: ${exception.message}", color = Color.Red)
                Button(onClick = { viewModel.refreshUsers() }) {
                    Text("Retry")
                }
            }
        }
    }
}
```

#### Cách 2: Sử dụng Resource pattern (Google's implementation)

```kotlin
// Resource pattern
sealed class Resource<out T> {
    data class Success<T>(val data: T) : Resource<T>()
    data class Loading<T>(val data: T? = null) : Resource<T>()
    data class Error<T>(val message: String, val data: T? = null) : Resource<T>()
}

// Repository
class UserRepository @Inject constructor(
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource: UserLocalDataSource
) {
    // Chỉ wrapper
    suspend fun getUsersFromRemote(): List<User> = remoteDataSource.getUsers()
    suspend fun getUsersFromLocal(): List<User> = localDataSource.getUsers()
    suspend fun saveUsers(users: List<User>) = localDataSource.saveUsers(users)
}

// Use case với cache strategy
class GetUsersUseCase @Inject constructor(
    private val repository: UserRepository
) {
    suspend operator fun invoke(forceRefresh: Boolean = false): Flow<Resource<List<User>>> = flow {
        emit(Resource.Loading())

        if (forceRefresh) {
            try {
                val remote = repository.getUsersFromRemote()
                repository.saveUsers(remote)
                emit(Resource.Success(remote))
            } catch (e: Exception) {
                val cached = repository.getUsersFromLocal()
                if (cached.isNotEmpty()) {
                    emit(Resource.Error(e.message ?: "Unknown error", cached))
                } else {
                    emit(Resource.Error(e.message ?: "Unknown error"))
                }
            }
        } else {
            val cached = repository.getUsersFromLocal()
            if (cached.isNotEmpty()) {
                emit(Resource.Success(cached))
            } else {
                try {
                    val remote = repository.getUsersFromRemote()
                    repository.saveUsers(remote)
                    emit(Resource.Success(remote))
                } catch (e: Exception) {
                    emit(Resource.Error(e.message ?: "Unknown error"))
                }
            }
        }
    }
}
```

#### Cách 3: Separate cache manager (advanced)

```kotlin
// Cache manager
interface CacheManager<T> {
    suspend fun get(key: String): T?
    suspend fun put(key: String, value: T)
    suspend fun invalidate(key: String)
    suspend fun isStale(key: String, ttl: Duration): Boolean
}

// Repository chỉ wrapper
class UserRepository @Inject constructor(
    private val remoteDataSource: UserRemoteDataSource,
    private val cacheManager: CacheManager<List<User>>
) {
    suspend fun getUsers(forceRefresh: Boolean): Flow<Resource<List<User>>> = flow {
        val cacheKey = "users"

        if (forceRefresh || cacheManager.isStale(cacheKey, 5.minutes)) {
            emit(Resource.Loading())
            try {
                val remote = remoteDataSource.getUsers()
                cacheManager.put(cacheKey, remote)
                emit(Resource.Success(remote))
            } catch (e: Exception) {
                val cached = cacheManager.get(cacheKey)
                if (cached != null) {
                    emit(Resource.Error(e.message ?: "Network failed", cached))
                } else {
                    emit(Resource.Error(e.message ?: "Network failed"))
                }
            }
        } else {
            val cached = cacheManager.get(cacheKey)
            if (cached != null) {
                emit(Resource.Success(cached))
            } else {
                emit(Resource.Loading())
                try {
                    val remote = remoteDataSource.getUsers()
                    cacheManager.put(cacheKey, remote)
                    emit(Resource.Success(remote))
                } catch (e: Exception) {
                    emit(Resource.Error(e.message ?: "Network failed"))
                }
            }
        }
    }
}
```

### Diagram architecture

```
┌─────────────────────────────────────────────────────────┐
│                      Presentation Layer                  │
│  ┌──────────────┐         ┌──────────────┐              │
│  │   ViewModel  │────────▶│     UI       │              │
│  └──────┬───────┘         └──────────────┘              │
│         │                                                  │
└─────────┼────────────────────────────────────────────────┘
          │
┌─────────▼────────────────────────────────────────────────┐
│                      Domain Layer                         │
│  ┌────────────────────────────────────────────────────┐ │
│  │              Use Cases (Interactors)                  │ │
│  │  - GetUsersUseCase                                  │ │
│  │  - RefreshUsersUseCase                              │ │
│  │  - Orchestrate data strategy                        │ │
│  │  - Handle business logic                           │ │
│  └───────────────┬──────────────────────────────────────┘ │
└──────────────────┼────────────────────────────────────────┘
                   │
┌──────────────────▼────────────────────────────────────────┐
│                       Data Layer                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │                  Repository                         │ │
│  │  - getUsersFromRemote()                            │ │
│  │  - getUsersFromLocal()                             │ │
│  │  - saveUsers()                                     │ │
│  │  - Only wrapper, no business logic                 │ │
│  └───────┬───────────────────────┬────────────────────┘ │
│          │                       │                        │
│  ┌───────▼──────────┐    ┌───────▼──────────┐           │
│  │ Remote Data      │    │ Local Data        │           │
│  │ Source (API)     │    │ Source (Room)     │           │
│  └──────────────────┘    └──────────────────┘           │
└──────────────────────────────────────────────────────────┘
```

## Key Points

1. **Repository là abstraction layer**, không nên có business logic về data strategy
2. **Use case orchestrate logic** - quyết định cache/remote/error handling
3. **Result/Resource pattern** - distinguish giữa success, error, stale data
4. **UI aware of data state** - show appropriate feedback to user
5. **Separate concerns** - dễ test, maintain, và change strategy
6. **Refresh gesture** - use case có thể force refresh và bypass cache

## Related Topics

- Clean Architecture
- Repository Pattern
- Single Responsibility Principle
- Cache Strategy
- Error Handling
