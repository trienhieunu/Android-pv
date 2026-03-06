---
ID: 001
Date: 2026-03-06
Tags: multiple-state-changes, race-condition, flicker, throttle-debounce, reducer, pure-function, state-management, intent-queue
---

# MVI Multiple State Changes & Race Condition

## Câu hỏi

Trong MVI, khi có multiple state changes trong short timeframe (như user click nhanh 3 lần), làm thế nào để đảm bảo UI chỉ render final state mà không bị flicker hay race condition?

## Câu trả lời

### Giải đáp gốc rễ

- **Intent queue** - mỗi user action tạo intent, có thể pending khi xử lý bất đồng bộ
- **State reduction** - reducer phải là pure function, nhưng nhiều dev nhầm với mutable state
- **State snapshotting** - UI nhận state mới, render, rồi state mới nữa đến gây flicker
- **Throttle/debounce** - cần thiết cho UI events nhưng sai chỗ sẽ break user expectation
- **Single source of truth** - reducer phải deterministically map previous state + intent → new state
- **Bẫy thực chiến** - nhiều dev xử lý race condition ở UI layer thay vì domain layer, gây inconsistency

### Trả lời theo kiểu phỏng vấn

> "Trong MVI, key là reducer phải là pure function và deterministically transform state. Khi nhiều intents đến nhanh, có 2 giải pháp: throttle UI events tại source hoặc debounce nếu action có thể gộp, nhưng phải hiểu rõ difference - throttle bỏ bớt intent, debounce gộp intent. Quan trọng hơn là implement state properly: mỗi state object immutable, UI render snapshot của state hiện tại, không render mỗi intent. Nếu cần avoid flicker, thêm loading state hoặc skeleton screen thay vì render nhiều partial states. Trong production, tôi thường combine debounce cho search/filter input với single-pass reducer, đảm bảo UI chỉ nhận consistent state sequence. Race condition nên xử lý ở use case/repository layer với proper synchronization, không đẩy xuống UI component."

### Giải pháp

1. **Reducer là pure function** - deterministic state transformation
2. **Throttle UI events tại source** - limit intent frequency
3. **Debounce cho gộp actions** - như search/filter input
4. **Immutable state objects** - mỗi state là snapshot
5. **Skeleton/loading state** - avoid rendering partial states
6. **Race condition xử lý ở domain layer** - không ở UI layer

## Tình huống ví dụ

### Code có vấn đề

```kotlin
// ❌ Vấn đề: Mutable state + race condition

data class State(
    var isLoading: Boolean = false,
    var data: String? = null,
    var error: String? = null
)

class MyViewModel : ViewModel() {
    private val _state = MutableStateFlow(State())
    val state: StateFlow<State> = _state.asStateFlow()

    fun fetchData() {
        // ⚠️ Vấn đề 1: Mutable state
        _state.value.isLoading = true
        _state.value = _state.value // Trigger recomposition

        viewModelScope.launch {
            try {
                val result = api.fetchData()
                // ⚠️ Vấn đề 2: Race condition nếu user click nhiều lần
                _state.value.data = result
                _state.value.isLoading = false
                _state.value = _state.value
            } catch (e: Exception) {
                _state.value.error = e.message
                _state.value.isLoading = false
                _state.value = _state.value
            }
        }
    }
}

// UI
@Composable
fun MyScreen(viewModel: MyViewModel) {
    val state by viewModel.state.collectAsState()

    // ⚠️ Vấn đề 3: Flicker - render mỗi state change
    when {
        state.isLoading -> CircularProgressIndicator()
        state.data != null -> Text(state.data!!)
        state.error != null -> Text(state.error!!)
    }
}
```

### User click 3 lần nhanh → Race condition timeline

```
T=0ms: User click #1
  → isLoading = true (render loading spinner)
  → API call #1 start

T=100ms: User click #2
  → isLoading = true (render loading spinner again ⚠️)
  → API call #2 start

T=200ms: User click #3
  → isLoading = true (render loading spinner again ⚠️)
  → API call #3 start

T=500ms: API call #1 complete
  → data = "result1" (render result #1) ⚠️
  → isLoading = false

T=600ms: API call #2 complete
  → data = "result2" (render result #2) ⚠️ Flicker!
  → isLoading = false

T=700ms: API call #3 complete
  → data = "result3" (render result #3) ⚠️ Flicker!
  → isLoading = false
```

**Vấn đề:** UI flicker 3 lần, render multiple intermediate states, không biết state nào là final.

### Code giải pháp: Immutable state + Throttle

```kotlin
// ✅ Giải pháp 1: Immutable state + pure reducer

data class State(
    val isLoading: Boolean = false,
    val data: String? = null,
    val error: String? = null
)

sealed class Intent {
    object FetchData : Intent()
    data class DataLoaded(val result: String) : Intent()
    data class ErrorOccurred(val error: String) : Intent()
}

class MyViewModel : ViewModel() {
    private val _state = MutableStateFlow(State())
    val state: StateFlow<State> = _state.asStateFlow()

    fun onIntent(intent: Intent) {
        _state.value = reduce(_state.value, intent)
    }

    private fun reduce(state: State, intent: Intent): State {
        return when (intent) {
            is Intent.FetchData -> state.copy(isLoading = true, error = null)
            is Intent.DataLoaded -> state.copy(
                isLoading = false,
                data = intent.result
            )
            is Intent.ErrorOccurred -> state.copy(
                isLoading = false,
                error = intent.error
            )
        }
    }

    fun fetchData() {
        onIntent(Intent.FetchData)
        viewModelScope.launch {
            try {
                val result = api.fetchData()
                onIntent(Intent.DataLoaded(result))
            } catch (e: Exception) {
                onIntent(Intent.ErrorOccurred(e.message ?: "Unknown error"))
            }
        }
    }
}
```

### Giải pháp 2: Throttle tại source

```kotlin
// ✅ Giải pháp 2: Throttle UI events

@Composable
fun MyScreen(viewModel: MyViewModel) {
    val state by viewModel.state.collectAsState()
    val coroutineScope = rememberCoroutineScope()

    // ⚡ Throttle: chỉ cho phép 1 click trong 1 giây
    val throttledFetchData = remember {
        throttle(coroutineScope) {
            viewModel.fetchData()
        }
    }

    Button(onClick = throttledFetchData) {
        Text("Fetch Data")
    }

    when {
        state.isLoading -> CircularProgressIndicator()
        state.data != null -> Text(state.data!!)
        state.error != null -> Text(state.error!!)
    }
}

// Throttle helper function
fun throttle(
    scope: CoroutineScope,
    waitTime: Long = 1000L,
    action: () -> Unit
): () -> Unit {
    var lastExecutionTime = 0L

    return {
        val currentTime = System.currentTimeMillis()
        if (currentTime - lastExecutionTime >= waitTime) {
            lastExecutionTime = currentTime
            scope.launch {
                action()
            }
        }
    }
}
```

### Giải pháp 3: Debounce cho search input

```kotlin
// ✅ Giải pháp 3: Debounce cho search/filter

class SearchViewModel : ViewModel() {
    private val _state = MutableStateFlow(State())
    val state: StateFlow<State> = _state.asStateFlow()

    private val searchQuery = MutableStateFlow("")

    init {
        // ⚡ Debounce: đợi 300ms sau khi user dừng gõ mới search
        searchQuery
            .debounce(300)
            .distinctUntilChanged()
            .onEach { query ->
                onIntent(Intent.Search(query))
            }
            .launchIn(viewModelScope)
    }

    fun onSearchQueryChanged(query: String) {
        searchQuery.value = query
    }

    private fun onIntent(intent: Intent) {
        _state.value = reduce(_state.value, intent)
    }

    private fun reduce(state: State, intent: Intent): State {
        return when (intent) {
            is Intent.Search -> state.copy(isLoading = true)
            is Intent.SearchResult -> state.copy(
                isLoading = false,
                results = intent.results
            )
            is Intent.SearchError -> state.copy(
                isLoading = false,
                error = intent.error
            )
        }
    }
}

// UI
@Composable
fun SearchScreen(viewModel: SearchViewModel) {
    val state by viewModel.state.collectAsState()

    TextField(
        value = viewModel.searchQuery.value,
        onValueChange = { viewModel.onSearchQueryChanged(it) },
        label = { Text("Search") }
    )

    when {
        state.isLoading -> CircularProgressIndicator()
        state.results.isNotEmpty() -> LazyColumn {
            items(state.results) { item -> Text(item.name) }
        }
        state.error != null -> Text(state.error!!)
    }
}
```

### Giải pháp 4: Skeleton screen để avoid flicker

```kotlin
// ✅ Giải pháp 4: Skeleton screen thay vì loading spinner

@Composable
fun MyScreen(viewModel: MyViewModel) {
    val state by viewModel.state.collectAsState()

    when {
        state.isLoading && state.data == null -> {
            // Skeleton screen - giữ layout giống kết quả
            Column {
                repeat(3) {
                    Card(modifier = Modifier.fillMaxWidth().padding(8.dp)) {
                        Box(
                            modifier = Modifier
                                .fillMaxWidth()
                                .height(60.dp)
                                .shimmer()
                                .background(Color.Gray.copy(alpha = 0.3f))
                        )
                    }
                }
            }
        }
        state.data != null -> {
            // Real content - không flicker vì layout giống skeleton
            LazyColumn {
                items(state.data) { item ->
                    Card(modifier = Modifier.fillMaxWidth().padding(8.dp)) {
                        Text(item.name)
                    }
                }
            }
        }
        state.error != null -> {
            ErrorView(state.error!!) {
                viewModel.fetchData()
            }
        }
    }
}

// Shimmer effect helper
@Composable
fun Modifier.shimmer(): Modifier {
    // Implement shimmer animation
    return this
}
```

### Giải pháp 5: Race condition ở domain layer

```kotlin
// ✅ Giải pháp 5: Xử lý race condition ở use case/repository

class FetchDataUseCase(
    private val repository: DataRepository
) {
    // ⚡ Single-flight: nếu request đang chạy, return current job
    private var currentJob: Job? = null

    suspend fun execute(): Result<String> {
        // Nếu có request đang chạy, chờ kết quả
        currentJob?.let { job ->
            return try {
                job.join()
                // Job đã complete, trigger mới
                fetchNewData()
            } catch (e: CancellationException) {
                // Job bị cancel, request mới
                fetchNewData()
            }
        }

        return fetchNewData()
    }

    private suspend fun fetchNewData(): Result<String> {
        val job = viewModelScope.launch {
            repository.fetchData()
        }
        currentJob = job

        return try {
            job.await()
        } finally {
            currentJob = null
        }
    }
}

// Repository với proper synchronization
class DataRepository(
    private val api: ApiService,
    private val cache: DataCache
) {
    private val mutex = Mutex()

    suspend fun fetchData(): Result<String> = withContext(Dispatchers.IO) {
        mutex.withLock {
            // Single-threaded access đến data source
            try {
                val result = api.fetchData()
                cache.save(result)
                Result.success(result)
            } catch (e: Exception) {
                // Fallback to cache
                val cached = cache.get()
                if (cached != null) {
                    Result.success(cached)
                } else {
                    Result.failure(e)
                }
            }
        }
    }
}
```

## Key Points

1. **Reducer là pure function** - deterministic state transformation, không side effect
2. **Immutable state objects** - mỗi state là snapshot, không mutate
3. **Throttle tại source** - limit intent frequency, avoid redundant work
4. **Debounce cho gộp actions** - search/filter input, gộp nhiều intent thành 1
5. **Skeleton/loading state** - avoid rendering partial states, reduce flicker
6. **Race condition ở domain layer** - use case/repository với proper synchronization
7. **Single source of truth** - reducer deterministically map previous state + intent → new state
8. **Không xử lý race condition ở UI layer** - gây inconsistency, hard to debug

## Throttle vs Debounce - Khi dùng cái nào?

| Scenario | Solution | Why |
|----------|----------|-----|
| Button click | **Throttle** | Limit frequency, vẫn trigger lần đầu |
| Search input | **Debounce** | Gộp nhiều keystrokes thành 1 search |
| Scroll events | **Throttle** | Limit event frequency |
| Resize events | **Debounce** | Chỉ xử lý khi user dừng resize |
| Auto-save | **Debounce** | Gộp nhiều changes thành 1 save |

---

## Tags

`MVI`, `State Management`, `Race Condition`, `Throttle`, `Debounce`, `Reducer`, `Pure Function`, `Immutable State`
