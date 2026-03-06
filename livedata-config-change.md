# LiveData Config Change Issue

## Câu hỏi

Trong MVVM với LiveData, khi ViewModel cần thực hiện async operation và observe kết quả, nhiều dev viết: `viewModel.getData()` trong Activity `onCreate`, sau đó observe LiveData. Vấn đề gì với cách này nếu Activity bị rotate trong lúc API call đang chạy?

## Câu trả lời

### Giải đáp gốc rễ

- **ViewModel survive config change**, Activity được recreate hoàn toàn
- `observe()` trong onCreate sẽ đăng ký observer mới mỗi lần Activity recreate
- LiveData giữ reference đến observer cũ cho đến khi lifecycle thay đổi
- Nếu API call mất nhiều thời gian, có thể có **multiple observers** cùng chờ
- **Bẫy:** Observer cũ có thể trigger callback sau khi Activity đã bị destroy
- **Kết quả:** Memory leak hoặc UI update cho Activity đã chết

### Trả lời theo kiểu phỏng vấn

> "Vấn đề chính là observer duplication và potential memory leak. Khi Activity rotate, ViewModel vẫn giữ LiveData đang fetch data, nhưng Activity mới lại đăng ký observer mới trong onCreate. Observer cũ vẫn tồn tại trong LiveData's observer list, và khi API response trả về, cả observer cũ lẫn mới đều nhận callback. Observer cũ thuộc về Activity đã bị destroy, nên callback này có thể gây memory leak hoặc worse - update UI của activity không còn tồn tại. Để tránh, tôi dùng SingleLiveEvent pattern hoặc switchMap để bắt đầu lại request khi data source thay đổi, hoặc đơn giản là remove observer trong onStop/onDestroy."

### Giải pháp

1. **Chỉ fetch data một lần trong ViewModel** (sử dụng flag)
2. **Dùng SingleLiveEvent pattern** - đảm bảo chỉ một observer nhận event
3. **Dùng switchMap/MediatorLiveData** - re-trigger request khi cần thiết
4. **Chuyển sang Kotlin Flow/StateFlow** - giải quyết root cause tốt hơn

## Tình huống ví dụ

### Code có vấn đề

```kotlin
// ViewModel
class MyViewModel : ViewModel() {
    private val _data = MutableLiveData<String>()
    val data: LiveData<String> = _data

    fun getData() {
        // Giả sử API call mất 10 giây
        viewModelScope.launch {
            delay(10000)
            _data.value = "Kết quả từ API"
        }
    }
}

// Activity
class MainActivity : AppCompatActivity() {
    private val viewModel: MyViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // ⚠️ Vấn đề 1: gọi getData() mỗi lần onCreate
        viewModel.getData()

        // ⚠️ Vấn đề 2: observe mỗi lần onCreate
        viewModel.data.observe(this) { result ->
            textView.text = result
        }
    }
}
```

### Timeline những gì xảy ra

```
T=0s: User mở app, Activity#1 được tạo
  → Activity#1.onCreate() được gọi
  → viewModel.getData() được gọi lần 1
  → API call bắt đầu (mất 10s)
  → Observer#1 được đăng ký

T=3s: User rotate màn hình
  → Activity#1 bị destroy
  → Observer#1 được remove (lifecycle-aware)
  → Activity#2 được tạo (nhiều instance mới!)
     → viewModel.getData() được gọi LẦN 2! ⚠️
        → API call mới bắt đầu (cũng mất 10s)
     → Observer#2 được đăng ký

T=10s: API call từ lần thứ nhất hoàn thành
  → LiveData.emit("Kết quả từ API")
  → Observer#1 đã bị remove (không nhận callback) ✅
  → Observer#2 nhận callback ✅

T=13s: API call từ lần thứ hai hoàn thành
  → LiveData.emit("Kết quả từ API")
  → Observer#2 nhận callback lần 2 (không cần thiết)
```

**Vấn đề:** API call được chạy 2 lần cho cùng dữ liệu → tốn resource, battery, bandwidth

### Race condition (rare but possible)

```
T=0s: Activity#1.onCreate()
  → viewModel.getData() #1
  → observer#1 registered

T=3s: Rotate màn hình
  → Activity#1.start destroying...
  → NHƯNG observer#1 chưa kịp được remove (lifecycle event chưa hoàn tất)
  → Activity#2.onCreate()
     → viewModel.getData() #2
     → observer#2 registered

T=3.05s: API call #1 hoàn thành (rất nhanh!)
  → LiveData emits value
  → observer#1 CÒN trong list (chưa kịp remove)
     → Callback cho Activity#1 ⚠️
     → textView của Activity#1 (đang destroy) được update
     → Memory leak hoặc ViewNotFound exception

T=3.10s: Lifecycle event hoàn tất
  → observer#1 được remove (nhưng đã quá muộn)
```

### Code giải pháp

#### Cách 1: Chỉ fetch data một lần (flag)

```kotlin
class MyViewModel : ViewModel() {
    private val _data = MutableLiveData<String>()
    val data: LiveData<String> = _data
    private var _fetched = false

    fun getData() {
        if (!_fetched) {
            viewModelScope.launch {
                _data.value = fetchFromApi()
                _fetched = true
            }
        }
    }
}
```

#### Cách 2: Dùng switchMap

```kotlin
class MyViewModel : ViewModel() {
    private val trigger = MutableLiveData<Unit>()
    val data: LiveData<String> = trigger.switchMap {
        MutableLiveData<String>().apply {
            viewModelScope.launch {
                value = fetchFromApi()
            }
        }
    }

    fun getData() {
        trigger.value = Unit
    }
}
```

#### Cách 3: SingleLiveEvent pattern

```kotlin
class SingleLiveEvent<T> : MutableLiveData<T>() {
    private val pending = AtomicBoolean(false)

    override fun observe(owner: LifecycleOwner, observer: Observer<in T>) {
        super.observe(owner) { t ->
            if (pending.compareAndSet(true, false)) {
                observer.onChanged(t)
            }
        }
    }

    override fun setValue(t: T?) {
        pending.set(true)
        super.setValue(t)
    }
}
```

#### Cách 4: Chuyển sang StateFlow (khuyên dùng)

```kotlin
class MyViewModel : ViewModel() {
    private val _data = MutableStateFlow<String?>(null)
    val data: StateFlow<String?> = _data.asStateFlow()

    fun getData() {
        viewModelScope.launch {
            if (_data.value == null) {
                _data.value = fetchFromApi()
            }
        }
    }
}

// Activity
lifecycleScope.launch {
    viewModel.data.collectLatest { result ->
        result?.let { textView.text = it }
    }
}
```

## Key Points

1. **ViewModel survive config change**, nhưng observer thì không
2. **Observe trong onCreate = multiple observers** khi rotate
3. **Gọi async function trong onCreate = duplicate work**
4. **LiveData lifecycle-aware giúp**, nhưng không giải quyết hết race condition
5. **Flow/StateFlow giải quyết root cause tốt hơn** vì emit-based, không observer-based

## Tags

`LiveData`, `ViewModel`, `Config Change`, `Observer Pattern`, `Lifecycle`
