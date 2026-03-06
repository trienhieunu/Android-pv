---
ID: 002
Date: 2026-03-06
Tags: stateflow, mvi, events-vs-state, transient-events, snackbar, toast, configuration-change, idempotent
---

# Events vs State in StateFlow/MVI

## Câu hỏi

Trong MVI với StateFlow, state phải là immutable snapshot của UI. Nhưng nhiều dev thêm `errorMessage: String?` vào data class để show Snackbar hoặc Toast. Vấn đề gì với cách này khi có configuration change hoặc UI temporarily goes to background?

## Câu trả lời

### Giải đáp gốc rễ

- **StateFlow luôn emit giá trị LATEST** cho mọi observer mới subscribe
- Khi fragment recreate sau rotation, nó nhận lại state cũ có errorMessage
- **Snackbar/Toast sẽ show lại** dù user đã dismiss lần trước
- State supposed to be **idempotent** - same state should render same UI
- **Events vs State** - events là transient (một lần), state là persistent (lâu dài)
- Mix cả hai vào same object violates **single responsibility**

### Trả lời theo kiểu phỏng vấn

> "Vấn đề là confuse giữa state và event. StateFlow design để giữ UI state, không phải transient events. Khi tôi add errorMessage vào state, nó trở thành persistent - observer mới sẽ nhận lại và trigger event again. Với navigation, toast hay snackbar, tôi cần event-based solution. Mình có thể dùng SharedFlow with replay=1, hoặc implement SingleLiveEvent pattern, hoặc create separate Event wrapper class như `Event<Error>` và consume nó sau khi handle. Key principle: state reflects UI tại thời điểm hiện tại, events là action xảy ra một lần. Don't pollute state with transient stuff."

### Giải pháp

1. **Separate Events from State** - dùng SharedFlow cho transient events
2. **Event wrapper pattern** - `Event<T>` class với consumeAfterHandle
3. **Channel** - send one-time events, buffer size = 1
4. **SingleLiveEvent-like pattern** - ensure event only delivered once

## Tình huống ví dụ

### Code có vấn đề

```kotlin
// State data class
data class UiState(
    val isLoading: Boolean = false,
    val data: List<Item> = emptyList(),
    val errorMessage: String? = null  // ⚠️ Vấn đề ở đây!
)

// ViewModel
class MyViewModel : ViewModel() {
    private val _state = MutableStateFlow(UiState())
    val state: StateFlow<UiState> = _state.asStateFlow()

    fun loadData() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            try {
                val result = api.getData()
                _state.update { it.copy(isLoading = false, data = result) }
            } catch (e: Exception) {
                // ⚠️ Thêm error vào state - sẽ persist!
                _state.update { it.copy(isLoading = false, errorMessage = e.message) }
            }
        }
    }
}

// Fragment
class MyFragment : Fragment() {
    private val viewModel: MyViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.state.collect { state ->
                if (state.errorMessage != null) {
                    // ⚠️ Sẽ show lại mỗi lần configuration change!
                    Snackbar.make(requireView(), state.errorMessage, Snackbar.LENGTH_LONG)
                        .show()
                }
                // Update UI...
            }
        }

        viewModel.loadData()
    }
}
```

### Timeline những gì xảy ra

```
T=0s: Fragment#1 được tạo
  → viewModel.state.collect() bắt đầu
  → state = UiState(isLoading = true)
  → API call đang chạy...

T=2s: API error xảy ra
  → state = UiState(errorMessage = "Network error")
  → Snackbar shown ✅
  → User dismiss Snackbar ✅

T=3s: User rotate màn hình
  → Fragment#1 bị destroy
  → Observer#1 cancel ✅
  → Fragment#2 được tạo
     → viewModel.state.collect() bắt đầu LẠI
     → StateFlow emit LATEST state
     → state = UiState(errorMessage = "Network error")  // ⚠️ State vẫn có error!

T=3.1s: Observer#2 nhận state
  → errorMessage = "Network error"  // ⚠️ Không null!
  → Snackbar shown LẠI ⚠️
     → User đã dismiss rồi nhưng lại thấy! 🤬
```

**Vấn đề:** Event được trigger lại dù user đã xử lý rồi!

### Case khác: Background và quay lại

```
T=0s: App đang ở foreground
  → State = UiState(errorMessage = "Error occurred")
  → Snackbar shown, user dismiss ✅

T=5s: User switch app, fragment goes to background
  → Fragment is in BACKSTACK state
  → Observer vẫn active (lifecycle in STARTED)

T=30s: User quay lại app
  → Fragment resumed
  → Nhận state again (vì lifecycle change)
  → Snackbar shown LẠI ⚠️
```

---

### Code giải pháp

#### Cách 1: Tách Events sang SharedFlow

```kotlin
// State - chỉ giữ UI state
data class UiState(
    val isLoading: Boolean = false,
    val data: List<Item> = emptyList()
)

// ViewModel
class MyViewModel : ViewModel() {
    private val _state = MutableStateFlow(UiState())
    val state: StateFlow<UiState> = _state.asStateFlow()

    // Tách events ra
    private val _events = MutableSharedFlow<UiEvent>()
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()

    sealed class UiEvent {
        data class ShowError(val message: String) : UiEvent()
        data class NavigateToDetail(val itemId: String) : UiEvent()
        data class ShowSnackbar(val message: String) : UiEvent()
    }

    fun loadData() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            try {
                val result = api.getData()
                _state.update { it.copy(isLoading = false, data = result) }
            } catch (e: Exception) {
                _state.update { it.copy(isLoading = false) }
                // Emit event thay vì update state
                _events.emit(UiEvent.ShowError(e.message ?: "Unknown error"))
            }
        }
    }
}

// Fragment
class MyFragment : Fragment() {
    private val viewModel: MyViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // Collect state
        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.state.collect { state ->
                // Update UI state
                progressBar.visibility = if (state.isLoading) View.VISIBLE else View.GONE
                adapter.submitList(state.data)
            }
        }

        // Collect events (separately!)
        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.events.collect { event ->
                when (event) {
                    is UiEvent.ShowError -> {
                        Snackbar.make(requireView(), event.message, Snackbar.LENGTH_LONG)
                            .show()
                    }
                    is UiEvent.NavigateToDetail -> {
                        findNavController().navigate(
                            MyFragmentDirections.actionToDetail(event.itemId)
                        )
                    }
                    is UiEvent.ShowSnackbar -> {
                        Toast.makeText(requireContext(), event.message, Toast.LENGTH_SHORT)
                            .show()
                    }
                }
            }
        }

        viewModel.loadData()
    }
}
```

**Lưu ý:** SharedFlow với default `replay=0` sẽ chỉ emit cho collector active tại thời điểm emit. Nếu collector chưa subscribe → miss event!

---

#### Cách 2: Event Wrapper Pattern (consumeAfterHandle)

```kotlin
// Event wrapper class
open class Event<out T>(private val content: T) {

    private var hasBeenHandled = false

    fun getContentIfNotHandled(): T? {
        return if (hasBeenHandled) {
            null
        } else {
            hasBeenHandled = true
            content
        }
    }

    fun peekContent(): T = content
}

// State giữ Event thay vì trực tiếp
data class UiState(
    val isLoading: Boolean = false,
    val data: List<Item> = emptyList(),
    val errorEvent: Event<String>? = null  // Event wrapper
)

// ViewModel
class MyViewModel : ViewModel() {
    private val _state = MutableStateFlow(UiState())
    val state: StateFlow<UiState> = _state.asStateFlow()

    fun loadData() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            try {
                val result = api.getData()
                _state.update {
                    it.copy(isLoading = false, data = result, errorEvent = null)
                }
            } catch (e: Exception) {
                _state.update {
                    it.copy(
                        isLoading = false,
                        errorEvent = Event(e.message ?: "Unknown error")
                    )
                }
            }
        }
    }
}

// Fragment
class MyFragment : Fragment() {
    private val viewModel: MyViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.state.collect { state ->
                if (state.errorEvent != null) {
                    val message = state.errorEvent.getContentIfNotHandled()
                    if (message != null) {
                        // Chỉ handle nếu chưa được consume
                        Snackbar.make(requireView(), message, Snackbar.LENGTH_LONG)
                            .show()
                    }
                }

                // Update UI state
                progressBar.visibility = if (state.isLoading) View.VISIBLE else View.GONE
                adapter.submitList(state.data)
            }
        }

        viewModel.loadData()
    }
}
```

---

#### Cách 3: Dùng Channel cho one-time events

```kotlin
// ViewModel
class MyViewModel : ViewModel() {
    private val _state = MutableStateFlow(UiState())
    val state: StateFlow<UiState> = _state.asStateFlow()

    // Channel cho one-time events
    private val _events = Channel<UiEvent>(Channel.BUFFERED)
    val events: Flow<UiEvent> = _events.receiveAsFlow()

    sealed class UiEvent {
        data class ShowError(val message: String) : UiEvent()
        data class NavigateToDetail(val itemId: String) : UiEvent()
    }

    fun loadData() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            try {
                val result = api.getData()
                _state.update { it.copy(isLoading = false, data = result) }
            } catch (e: Exception) {
                _state.update { it.copy(isLoading = false) }
                _events.send(UiEvent.ShowError(e.message ?: "Unknown error"))
            }
        }
    }
}

// Fragment
class MyFragment : Fragment() {
    private val viewModel: MyViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.state.collect { state ->
                // Update UI state
                progressBar.visibility = if (state.isLoading) View.VISIBLE else View.GONE
                adapter.submitList(state.data)
            }
        }

        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.events.collect { event ->
                when (event) {
                    is UiEvent.ShowError -> {
                        Snackbar.make(requireView(), event.message, Snackbar.LENGTH_LONG)
                            .show()
                    }
                    is UiEvent.NavigateToDetail -> {
                        findNavController().navigate(
                            MyFragmentDirections.actionToDetail(event.itemId)
                        )
                    }
                }
            }
        }

        viewModel.loadData()
    }
}
```

---

#### Cách 4: SharedFlow với replay=1 (hơi tricky)

```kotlin
// ViewModel
class MyViewModel : ViewModel() {
    private val _state = MutableStateFlow(UiState())
    val state: StateFlow<UiState> = _state.asStateFlow()

    // replay=1 sẽ giữ last event cho collector mới
    private val _events = MutableSharedFlow<UiEvent>(
        replay = 1,
        extraBufferCapacity = 0,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()

    fun loadData() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            try {
                val result = api.getData()
                _state.update { it.copy(isLoading = false, data = result) }
                // Reset event khi success
                _events.tryEmit(UiEvent.NoEvent)
            } catch (e: Exception) {
                _state.update { it.copy(isLoading = false) }
                _events.emit(UiEvent.ShowError(e.message ?: "Unknown error"))
            }
        }
    }

    sealed class UiEvent {
        object NoEvent : UiEvent()
        data class ShowError(val message: String) : UiEvent()
    }
}
```

---

## So sánh các cách

| Cách | Ưu điểm | Nhược điểm | Khi dùng |
|------|---------|------------|----------|
| **SharedFlow** | Reactive, đơn giản | Collector chưa subscribe = miss event | Event có thể miss không sao |
| **Event Wrapper** | State-based, dễ debug | State bị polluted với Event wrapper | Cần event "survive" rotation |
| **Channel** | Đảm bảo event đến đúng 1 lần | Không replay, collector mới miss | Critical events không được miss |
| **SharedFlow + replay=1** | Hybrid | Phức tạp, cần reset event | Edge case đặc biệt |

---

## Key Points

1. **State = Snapshot of UI** - phải idempotent, same state → same UI
2. **Events = One-time actions** - navigation, toast, snackbar, dialog
3. **StateFlow không phải cho transient events** - dùng SharedFlow/Channel
4. **Configuration change** sẽ trigger state emit lại → events show lại nếu trong state
5. **Single responsibility** - tách state và events ra
6. **Pick solution dựa trên use case** - event có thể miss không? có cần survive rotation không?

---

## Best Practices

✅ **State:** isLoading, data, selected items, form inputs, UI mode  
✅ **Events:** navigation, toast, snackbar, dialog, permission request  
❌ **Không đưa events vào state** - errorMessage, navigationAction, showToast  
✅ **Dùng SharedFlow** cho events (replay=0 mặc định)  
✅ **Dùng Channel** cho critical events không được miss  
✅ **Event wrapper** khi cần event survive rotation (nhưng cẩn thận với lifecycle)

---

## Tài liệu tham khảo

- [Google Codelab: Advanced State & Side Effects](https://developer.android.com/codelabs/advanced-android-kotlin-training-lifecycle-state-management)
- [Android Guide to App Architecture](https://developer.android.com/topic/architecture)
- [StateFlow vs SharedFlow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
