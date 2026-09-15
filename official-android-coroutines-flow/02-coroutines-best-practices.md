# Bài 02 — Thực hành Tốt nhất cho Coroutines trên Android (Coroutines Best Practices)

> **Tài liệu tham chiếu gốc:** [Best practices for coroutines in Android — Android Developers](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)  
> **Áp dụng:** Kotlin 2.0+, `kotlinx.coroutines:1.11.0`, Android Architecture Guidelines  
> **Mục tiêu:** Nắm vững toàn bộ các quy tắc vàng và thực hành chuẩn mực từ đội ngũ kỹ sư Google Android để viết mã Coroutines chuyên nghiệp, tránh lỗi rò rỉ bộ nhớ, tối ưu hóa hiệu năng và viết Unit Test tin cậy.

---

## Danh mục Các Quy tắc Vàng từ Google

1. [Inject Dispatchers (Truyền phụ thuộc cho Dispatcher)](#1-inject-dispatchers-truyền-phụ-thuộc-cho-dispatcher)
2. [Suspend functions phải đảm bảo tính Main-safe](#2-suspend-functions-phải-đảm-bảo-tính-main-safe)
3. [ViewModel nên là nơi khởi tạo Coroutines](#3-viewmodel-nên-là-nơi-khởi-tạo-coroutines)
4. [Tuyệt đối không phơi bày các kiểu Mutable ra bên ngoài](#4-tuyệt-đối-không-phơi-bày-các-kiểu-mutable-ra-bên-ngoài)
5. [Tầng Data và Domain nên phơi bày Suspend functions và Flows](#5-tầng-data-và-domain-nên-phơi-bày-suspend-functions-và-flows)
6. [Tạo CoroutineScope độc lập ở tầng nghiệp vụ đúng cách](#6-tạo-coroutinescope-độc-lập-ở-tầng-nghiệp-vụ-đúng-cách)
7. [Tuyệt đối tránh sử dụng GlobalScope](#7-tuyệt-đối-tránh-sử-dụng-globalscope)
8. [Đảm bảo Coroutine có khả năng hủy bỏ phối hợp (Cooperative Cancellation)](#8-đảm-bảo-coroutine-có-khả-năng-hủy-bỏ-phối-hợp-cooperative-cancellation)
9. [Xử lý Ngoại lệ cẩn trọng và không nuốt chửng CancellationException](#9-xử-lý-ngoại-lệ-cẩn-trọng-và-không-nuốt-chửng-cancellationexception)

---

## 1. Inject Dispatchers (Truyền phụ thuộc cho Dispatcher)

> [!IMPORTANT]
> **Quy tắc:** Không hardcode `Dispatchers.IO` hoặc `Dispatchers.Default` trực tiếp bên trong các class. Thay vào đó, hãy truyền `CoroutineDispatcher` qua constructor (Dependency Injection).

### Tại sao?
Trong môi trường kiểm thử tự động (Unit Testing), các thread của `Dispatchers.IO` hay `Dispatchers.Default` chạy bất đồng bộ thực sự trên JVM, khiến các bài test chạy không đoán trước được (flaky tests) và rất khó kiểm soát thời gian. Khi bạn inject Dispatcher, bạn có thể dễ dàng thay thế chúng bằng `TestDispatcher` (ví dụ: `StandardTestDispatcher` hoặc `UnconfinedTestDispatcher`) để chạy test đồng bộ tức thì.

### ❌ Anti-pattern (Không nên làm):
```kotlin
// Không thể thay thế Dispatcher khi viết Unit Test!
class NewsRepository {
    suspend fun fetchNews() = withContext(Dispatchers.IO) {
        // Gọi Network/Database
    }
}
```

### ✅ Recommended pattern (Chuẩn Google):
```kotlin
class NewsRepository(
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) {
    suspend fun fetchNews() = withContext(ioDispatcher) {
        // Thực thi an toàn, dễ dàng mock ioDispatcher trong Unit Test
    }
}
```

#### Thiết lập với Hilt / Dagger:
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DispatcherModule {
    @Provides
    @IoDispatcher
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO

    @Provides
    @DefaultDispatcher
    fun provideDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default
}
```

---

## 2. Suspend functions phải đảm bảo tính Main-safe

> [!IMPORTANT]
> **Quy tắc:** Mọi hàm `suspend` đều phải an toàn khi được gọi từ luồng chính (Main Thread).

Nếu một hàm suspend thực hiện thao tác I/O (mạng, đĩa) hoặc tính toán nặng ngốn CPU, chính hàm đó phải có trách nhiệm bọc mã thực thi trong `withContext(dispatcher)`. Caller (ví dụ: ViewModel hoặc UI) không cần phải biết bên dưới đang dùng luồng nào, họ chỉ cần gọi hàm suspend trên Main Thread mà không lo bị giật khung hình.

### ❌ Anti-pattern:
```kotlin
class UserRepository {
    // Đùn đẩy trách nhiệm đổi thread cho ViewModel
    suspend fun saveUserToDb(user: User) {
        database.userDao().insert(user) // Gây block nếu gọi từ Main Thread!
    }
}
```

### ✅ Recommended pattern:
```kotlin
class UserRepository(
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) {
    suspend fun saveUserToDb(user: User) = withContext(ioDispatcher) {
        database.userDao().insert(user) // Luôn an toàn tuyệt đối với caller!
    }
}
```

---

## 3. ViewModel nên là nơi khởi tạo Coroutines

> [!IMPORTANT]
> **Quy tắc:** ViewModel nên khởi tạo coroutine bằng `viewModelScope.launch` để thực hiện tác vụ nghiệp vụ thay vì để UI (Activity/Compose) tự launch hoặc để Repository launch.

ViewModel giữ trạng thái giao diện và tồn tại qua các sự kiện thay đổi cấu hình (như xoay màn hình). Khi người dùng nhấn một nút trên giao diện, UI chỉ nên gọi một hàm thông thường của ViewModel (ví dụ: `viewModel.refresh()`), sau đó ViewModel sẽ launch coroutine trong `viewModelScope`.

### ❌ Anti-pattern:
```kotlin
// UI (Compose hoặc Fragment) tự bọc launch coroutine để gọi suspend function
Button(onClick = {
    coroutineScope.launch {
        viewModel.loadData() // Nếu màn hình xoay, tác vụ này có thể bị cancel và chạy lại vô cớ!
    }
}) { Text("Tải dữ liệu") }
```

### ✅ Recommended pattern:
```kotlin
// UI chỉ gửi sự kiện
Button(onClick = { viewModel.loadData() }) {
    Text("Tải dữ liệu")
}

// ViewModel quản trị vòng đời và trạng thái
class MyViewModel(private val repository: NewsRepository) : ViewModel() {
    fun loadData() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            _uiState.value = repository.fetchNews()
        }
    }
}
```

---

## 4. Tuyệt đối không phơi bày các kiểu Mutable ra bên ngoài

> [!IMPORTANT]
> **Quy tắc:** Chỉ phơi bày (expose) các kiểu dữ liệu luồng chỉ đọc (`StateFlow`, `SharedFlow`, `LiveData`) ra bên ngoài class. Giữ biến `Mutable` ở phạm vi `private`.

Việc làm lộ `MutableStateFlow` ra bên ngoài khiến các thành phần khác (như UI hoặc các class khác) có thể can thiệp và thay đổi trạng thái của ViewModel một cách tùy tiện, phá vỡ nguyên lý **Single Source of Truth (Nguồn chân lý duy nhất)** và gây ra các lỗi khó truy vết.

### ❌ Anti-pattern:
```kotlin
class UserViewModel : ViewModel() {
    // Nguy hiểm: UI bên ngoài có thể gọi `uiState.value = ...` làm sai lệch logic!
    val uiState = MutableStateFlow<UserUiState>(UserUiState.Loading)
}
```

### ✅ Recommended pattern:
```kotlin
class UserViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<UserUiState>(UserUiState.Loading)
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()
}
```

---

## 5. Tầng Data và Domain nên phơi bày Suspend functions và Flows

> [!IMPORTANT]
> **Quy tắc:** Các class ở tầng Data (Repository, Data Source) và Domain (Use Case) chỉ nên phơi bày:
> - **Hàm `suspend`** cho các tác vụ lấy dữ liệu một lần (one-shot requests).
> - **`Flow<T>`** cho các luồng dữ liệu cần theo dõi liên tục theo thời gian (continuous data streams).

Các tầng này **không nên** tự ý khởi chạy coroutine bằng một scope chưa được kiểm soát vòng đời. Hãy để caller ở tầng trên (ViewModel) quyết định khi nào cần bắt đầu, khi nào cần hủy, và chạy trong phạm vi nào.

### ✅ Recommended pattern:
```kotlin
interface ArticleRepository {
    // 1. One-shot operation: dùng suspend
    suspend fun getArticleById(id: String): Article

    // 2. Stream observation: dùng Flow
    fun observeLatestArticles(): Flow<List<Article>>
}
```

---

## 6. Tạo CoroutineScope độc lập ở tầng nghiệp vụ đúng cách

### Vấn đề: Tác vụ không được phép hủy khi ViewModel bị đóng
Thông thường, khi người dùng thoát màn hình, `viewModelScope` sẽ tự động bị hủy. Tuy nhiên, có những tác vụ nghiệp vụ quan trọng **không được phép bị hủy** giữa chừng, ví dụ:
- Ghi nhật ký phân tích (analytics logging)
- Lưu trữ giao dịch thanh toán vào local database
- Đồng bộ dữ liệu dở dang lên máy chủ

### Giải pháp chuẩn của Google:
Không dùng `GlobalScope`! Hãy định nghĩa một `CoroutineScope` cấp ứng dụng (Application Scope) được inject qua Dagger/Hilt, gắn liền với vòng đời của `Application`:

```kotlin
@Retention(AnnotationRetention.RUNTIME)
@Qualifier
annotation class ApplicationScope

@Module
@InstallIn(SingletonComponent::class)
object CoroutineScopesModule {
    @Provides
    @Singleton
    @ApplicationScope
    fun provideApplicationScope(
        @DefaultDispatcher defaultDispatcher: CoroutineDispatcher
    ): CoroutineScope = CoroutineScope(SupervisorJob() + defaultDispatcher)
}
```

Khi sử dụng trong Repository:
```kotlin
class LoggingRepository(
    @ApplicationScope private val externalScope: CoroutineScope,
    private val ioDispatcher: CoroutineDispatcher
) {
    suspend fun logUserAction(action: Action) {
        // Tác vụ này sẽ chạy đến cùng ngay cả khi ViewModel gọi nó đã bị hủy!
        externalScope.launch(ioDispatcher) {
            analyticsApi.send(action)
        }
    }
}
```

---

## 7. Tuyệt đối tránh sử dụng `GlobalScope`

> [!WARNING]
> **Cảnh báo nghiêm ngặt:** `GlobalScope` là một anti-pattern lớn trong Android và đã được đánh dấu là không khuyến khích (discouraged).

```
Tại sao GlobalScope gây nguy hiểm?
1. Phá vỡ Structured Concurrency: Không thuộc về bất kỳ cây phân cấp công việc nào.
2. Gây Memory Leak: Tiếp tục chạy và giữ tham chiếu đến Context/Activity đã bị hủy.
3. Bất khả thi trong Unit Test: Test runner kết thúc nhưng GlobalScope vẫn chạy ngầm trên thread khác.
```

---

## 8. Đảm bảo Coroutine có khả năng Hủy bỏ Phối hợp (Cooperative Cancellation)

> [!IMPORTANT]
> **Quy tắc:** Quá trình hủy bỏ Coroutine là **phối hợp (cooperative)**. Nếu mã của bạn đang chạy một vòng lặp tính toán nặng CPU mà không gọi bất kỳ hàm `suspend` nào của thư viện `kotlinx.coroutines`, coroutine đó **sẽ không thể bị hủy**!

### ❌ Anti-pattern:
```kotlin
suspend fun computePrimes() = withContext(Dispatchers.Default) {
    var i = 0
    while (i < 100_000_000) {
        // Vòng lặp tính toán không hề kiểm tra tín hiệu hủy!
        // Dù coroutine cha bị cancel, vòng lặp này vẫn chạy hết gây tốn pin và CPU.
        calculatePrime(i)
        i++
    }
}
```

### ✅ Recommended pattern:
Sử dụng hàm `ensureActive()` hoặc kiểm tra `isActive`:
```kotlin
suspend fun computePrimes() = withContext(Dispatchers.Default) {
    var i = 0
    while (i < 100_000_000) {
        ensureActive() // Sẽ ném CancellationException ngay lập tức nếu coroutine bị cancel!
        calculatePrime(i)
        i++
    }
}
```
Hoặc dùng `yield()` để nhường quyền CPU cho các coroutine khác cùng lúc kiểm tra cancellation:
```kotlin
suspend fun processLargeList(items: List<Item>) = withContext(Dispatchers.Default) {
    items.forEach { item ->
        yield() // Vừa kiểm tra cancel, vừa nhường CPU
        processItem(item)
    }
}
```

---

## 9. Xử lý Ngoại lệ Cẩn trọng và Không nuốt chửng `CancellationException`

Trong Kotlin Coroutines, cơ chế hủy bỏ được hiện thực hóa bằng cách ném ngoại lệ đặc biệt: `CancellationException`. 

> [!CAUTION]
> **Lưu ý sống còn:**  
> Nếu bạn dùng khối `try { ... } catch (e: Exception)` chung chung và không ném lại `CancellationException`, bạn đã vô tình **nuốt chửng tín hiệu hủy**, làm hỏng toàn bộ cơ chế Structured Concurrency của coroutine!

### ❌ Anti-pattern (Nuốt chửng cancellation):
```kotlin
suspend fun doWork() {
    try {
        delay(1000) // Ném CancellationException khi bị cancel
    } catch (e: Exception) {
        // Sai lầm nghiêm trọng: Nuốt luôn CancellationException!
        Log.e("Tag", "Đã xảy ra lỗi: ${e.message}")
    }
}
```

### ✅ Recommended pattern:
```kotlin
suspend fun doWork() {
    try {
        delay(1000)
    } catch (e: Exception) {
        if (e is CancellationException) {
            throw e // BẮT BUỘC rethrow CancellationException để coroutine hủy thành công!
        }
        Log.e("Tag", "Lỗi logic: ${e.message}")
    }
}
```

---

## Bảng Tóm tắt Thực hành Chuẩn Google

| Tiêu chí | ❌ Tránh (Anti-pattern) | ✅ Khuyến nghị (Best Practice) |
| :--- | :--- | :--- |
| **Dispatcher** | Hardcode `Dispatchers.IO` | Inject `CoroutineDispatcher` qua constructor |
| **Main-safety** | Caller phải lo chuyển thread | Suspend function tự chuyển qua `withContext` |
| **Quản trị Scope** | Dùng `GlobalScope` | Dùng `viewModelScope` hoặc Custom App Scope |
| **Khởi tạo Coroutine** | UI tự launch | ViewModel launch và phơi bày `StateFlow` |
| **Expose State** | Phơi bày `MutableStateFlow` | `private _state` + phơi bày `val state = _state.asStateFlow()` |
| **Tầng Repository** | Tự ý launch không kiểm soát | Chỉ phơi bày `suspend fun` và `Flow<T>` |
| **Cancellation** | Vòng lặp không kiểm tra trạng thái | Gọi `ensureActive()`, `yield()` hoặc check `isActive` |
| **Try-Catch** | Nuốt toàn bộ `Exception` | Luôn kiểm tra và `throw` lại `CancellationException` |
