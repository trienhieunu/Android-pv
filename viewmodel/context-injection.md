---
ID: 001
Date: 2026-03-06
Tags: context, androidviewmodel, memory-leak, constructor-injection, lifecycle, application-context
---

# ViewModel Context Injection

## Câu hỏi

Trong MVVM, ViewModel có thể nhận tham số Context thông qua constructor được không? Nếu được, làm thế nào để tránh memory leak khi Activity bị destroy?

## Câu trả lời

### Giải đáp gốc rễ

- **ViewModel không được giữ reference trực tiếp đến Activity/Fragment Context** - nguyên nhân chính gây memory leak
- **Application Context là an toàn** - nhưng đôi khi không phù hợp với nhu cầu (ví dụ: cần access themed resources)
- **AndroidViewModel** - nhận Application context qua constructor, giải quyết vấn đề một phần
- **Lifecycle aware** - ViewModel sống lâu hơn UI, nên reference tới short-lived object là nguy hiểm
- **ViewModelStoreOwner** - quan trọng hơn Context, dùng để bind lifecycle đúng
- **Trick thực chiến** - nhiều dev nhầm lẫn giữa Context, ApplicationContext và khi nào cần cái nào

### Trả lời theo kiểu phỏng vấn

> "ViewModel không nên nhận Activity hay Fragment Context qua constructor vì nó survive configuration changes và có thể sống lâu hơn UI component, dẫn đến memory leak. Nếu cần Context, có 2 cách: hoặc extend AndroidViewModel để nhận ApplicationContext - an toàn vì không bind tới UI lifecycle, hoặc inject dependency thay vì Context trực tiếp. Nhưng quan trọng nhất là hiểu rõ ViewModel cần Context để làm gì: nếu chỉ để access resource, ApplicationContext là đủ; nếu cần UI-related như themed resource hoặc system service, nên suy nghĩ lại kiến trúc - có thể responsibility đó thuộc View, không phải ViewModel. Trong dự án lớn, tôi thường dùng DI container để inject đúng loại Context cần thiết thay vì hardcode trong constructor."

### Giải pháp

1. **Dùng AndroidViewModel** - extend AndroidViewModel để nhận Application Context
2. **Dependency Injection** - inject context qua DI container (Hilt/Dagger)
3. **Interface abstration** - tạo interface cho resource access thay vì truyền Context
4. **Suy nghĩ lại kiến trúc** - ViewModel cần Context để làm gì? Nếu UI-related, move responsibility to View

## Tình huống ví dụ

### Code có vấn đề

```kotlin
// ❌ SAI: ViewModel nhận Activity Context
class MyViewModel(
    private val context: Context  // Activity Context passed in!
) : ViewModel() {

    fun getDeviceName(): String {
        // Truy cập Activity Context - nguy hiểm khi Activity destroyed
        return "${android.os.Build.MANUFACTURER} ${android.os.Build.MODEL}"
    }

    fun loadPreferences() {
        // Sử dụng Context - có thể gây memory leak
        val prefs = context.getSharedPreferences("settings", Context.MODE_PRIVATE)
    }
}

// Activity
class MainActivity : AppCompatActivity() {
    private val viewModel: MyViewModel by viewModels {
        object : ViewModelProvider.Factory {
            override fun <T : ViewModel> create(modelClass: Class<T>): T {
                @Suppress("UNCHECKED_CAST")
                return MyViewModel(this@MainActivity) as T  // ⚠️ Pass Activity Context!
            }
        }
    }
}
```

**Timeline:**

```
T=0s: MainActivity#1 được tạo
  → MyViewModel được tạo với Context của MainActivity#1
  → ViewModel holds reference đến Activity#1

T=3s: User rotate màn hình
  → MainActivity#1 bị destroy
  → NHƯNG ViewModel vẫn tồn tại (survive config change)
  → ViewModel vẫn giữ reference đến Activity#1 ⚠️
  → Activity#1 không thể được GC thu gom
  → Memory leak!

T=3.1s: MainActivity#2 được tạo
  → ViewModel được reuse (cùng instance)
  → ViewModel vẫn giữ reference cũ đến Activity#1 ⚠️
  → Activity#2 được tạo nhưng ViewModel không update reference
```

### Code giải pháp: AndroidViewModel

```kotlin
// ✅ ĐÚNG: Sử dụng AndroidViewModel
class MyViewModel(application: Application) : AndroidViewModel(application) {

    fun loadPreferences() {
        // Application Context là an toàn
        val prefs = getApplication<Application>()
            .getSharedPreferences("settings", Context.MODE_PRIVATE)
    }

    fun getDeviceName(): String {
        return "${android.os.Build.MANUFACTURER} ${android.os.Build.MODEL}"
    }
}

// Activity
class MainActivity : AppCompatActivity() {
    private val viewModel: MyViewModel by viewModels()
}
```

### Code giải pháp: Dependency Injection (Hilt)

```kotlin
// ✅ ĐÚNG: Dùng Hilt để inject Context
@Module
@InstallIn(SingletonComponent::class)
object AppModule {

    @Provides
    @ApplicationContext
    fun provideApplicationContext(@ApplicationContext context: Context): Context {
        return context
    }
}

@HiltViewModel
class MyViewModel @Inject constructor(
    @ApplicationContext private val context: Context
) : ViewModel() {

    fun loadPreferences() {
        // Application Context được inject an toàn
        val prefs = context.getSharedPreferences("settings", Context.MODE_PRIVATE)
    }
}

// Activity
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    private val viewModel: MyViewModel by viewModels()
}
```

### Code giải pháp: Interface Abstraction

```kotlin
// ✅ ĐÚNG: Tạo interface thay vì truyền Context
interface PreferenceProvider {
    fun getString(key: String, default: String): String
    fun setString(key: String, value: String)
}

class SharedPreferencesPreferenceProvider(
    private val context: Context
) : PreferenceProvider {

    override fun getString(key: String, default: String): String {
        return context.getSharedPreferences("settings", Context.MODE_PRIVATE)
            .getString(key, default) ?: default
    }

    override fun setString(key: String, value: String) {
        context.getSharedPreferences("settings", Context.MODE_PRIVATE)
            .edit()
            .putString(key, value)
            .apply()
    }
}

class MyViewModel(
    private val preferenceProvider: PreferenceProvider
) : ViewModel() {

    fun loadPreferences() {
        val value = preferenceProvider.getString("key", "default")
    }
}

// Hilt module
@Module
@InstallIn(SingletonComponent::class)
object AppModule {

    @Provides
    fun providePreferenceProvider(@ApplicationContext context: Context): PreferenceProvider {
        return SharedPreferencesPreferenceProvider(context)
    }
}
```

## Comparison: ViewModel vs AndroidViewModel

| Feature | ViewModel | AndroidViewModel |
|---------|----------|------------------|
| Constructor | Không có tham số | Nhận `Application: Application` |
| Context access | Không có | Có `getApplication<Application>()` |
| Use case | Pure business logic | Cần Application Context |
| Memory safe | ✅ Không hold Context | ✅ Chỉ hold Application Context |

## Key Points

1. **ViewModel không giữ reference đến Activity/Fragment Context** - nguyên nhân memory leak
2. **AndroidViewModel nhận Application Context** - an toàn vì singleton
3. **Dependency Injection là best practice** - inject đúng loại context qua DI container
4. **Interface abstraction** - giảm dependency cụ thể vào Context
5. **Suy nghĩ lại kiến trúc** - nếu cần UI-related Context, responsibility có thể thuộc View

## Context Types Summary

| Type | Lifecycle | Use Case |
|------|-----------|----------|
| **Activity Context** | Bind to Activity lifecycle | ❌ KHÔNG dùng trong ViewModel |
| **Fragment Context** | Bind to Fragment lifecycle | ❌ KHÔNG dùng trong ViewModel |
| **Application Context** | Singleton, app lifecycle | ✅ An toàn cho ViewModel |
| **ContextWrapper** | Base class | Tùy thuộc implementation |

## When Context is Needed in ViewModel

### ✅ Safe use cases (Application Context đủ)

- **SharedPreferences** - data storage
- **Database initialization** - Room database
- **System services** (ConnectivityManager, NotificationManager) - global services
- **File operations** - đọc/ghi file trong app storage
- **Resources without theme** - string, dimension, raw resources

### ❌ Unsafe use cases (cần Activity Context)

- **Themed resources** (color, style với theme) - UI-related
- **View inflation** - cần Display metrics
- **WindowManager** - UI-related system service
- **AlertDialog** - UI component
- **LayoutInflater** - cần Activity Context

**Nếu ViewModel cần Activity Context:** suy nghĩ lại kiến trúc - responsibility đó có thể nên move lên View layer.
