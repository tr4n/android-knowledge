# Bài 02 — Cold Flow vs Hot Flow: StateFlow & SharedFlow

> **Module:** 1 — Kotlin Coroutines & Flow Foundation  
> **Prerequisite:** Bài 01 — Coroutines Basics (suspend, CoroutineScope, Dispatcher)  
> **Official Docs:**
> - [Kotlin Flow](https://kotlinlang.org/docs/flow.html)
> - [StateFlow & SharedFlow — Android Developers](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
> - [Collecting flows in Android](https://developer.android.com/kotlin/flow/collect)

---

## 1. Định nghĩa & Thuật ngữ chuẩn

### Flow\<T\>

Theo Kotlin official documentation:

> *"A cold asynchronous data stream that sequentially emits values and completes normally or with an exception."*

`Flow<T>` là một **asynchronous data stream** có thể:
- Phát ra (emit) nhiều giá trị theo thứ tự
- Hoàn thành bình thường (`onCompletion`) hoặc với exception
- Bị huỷ thông qua Structured Concurrency

```
Package: kotlinx.coroutines.flow
Interface: Flow<out T>
Key method: suspend fun collect(collector: FlowCollector<T>)
```

---

### Cold Flow (Regular Flow)

| Thuộc tính | Giá trị |
|-----------|---------|
| **Khi nào chạy?** | Chỉ chạy khi có collector gọi `collect {}` |
| **Mỗi collector** | Nhận bộ emission hoàn toàn độc lập (fresh execution) |
| **Analogies** | Một cuốn sách — mỗi người đọc từ trang đầu |
| **Builder** | `flow { emit(...) }` |

```kotlin
val coldFlow: Flow<Int> = flow {
    println("Flow started!")   // chỉ in khi có collector
    emit(1)
    emit(2)
    emit(3)
}

// Collector 1
coldFlow.collect { println("C1: $it") }
// Output: "Flow started!", C1: 1, C1: 2, C1: 3

// Collector 2 — chạy lại từ đầu, hoàn toàn độc lập
coldFlow.collect { println("C2: $it") }
// Output: "Flow started!", C2: 1, C2: 2, C2: 3
```

---

### Hot Flow

| Thuộc tính | Giá trị |
|-----------|---------|
| **Khi nào chạy?** | Chạy độc lập, không cần collector |
| **Mỗi collector** | Nhận các giá trị được emit **sau khi** subscribe |
| **Analogies** | Kênh radio — phát sóng dù có ai nghe hay không |
| **Types** | `StateFlow`, `SharedFlow` |

---

### StateFlow\<T\>

Theo Android official documentation:

> *"A SharedFlow that represents a read-only state with a single updatable data value that emits updates to the value to its collectors."*

```
Interface: StateFlow<out T> : SharedFlow<T>
Mutable: MutableStateFlow<T> : StateFlow<T>
```

| Đặc điểm | Chi tiết |
|----------|---------|
| **Initial value** | Bắt buộc có (không thể null-implicit) |
| **Current value** | Truy cập trực tiếp qua `.value` |
| **Conflated** | Chỉ giữ giá trị mới nhất, bỏ intermediate values |
| **Deduplication** | Không emit nếu `newValue == currentValue` (dùng `equals()`) |
| **New collectors** | Nhận ngay current value khi subscribe |
| **Replay** | Cố định = 1 (luôn replay giá trị cuối) |

```kotlin
val stateFlow = MutableStateFlow(0)   // initial value = 0
stateFlow.value = 1
stateFlow.value = 2
// New collector now subscribes → receives 2 immediately
```

---

### SharedFlow\<T\>

```
Interface: SharedFlow<out T> : Flow<T>
Mutable: MutableSharedFlow<T> : SharedFlow<T>
```

| Đặc điểm | Chi tiết |
|----------|---------|
| **Initial value** | Không có (optional replay cache) |
| **Current value** | Không có `.value` |
| **Replay** | Cấu hình được (0 = no replay, N = cache N values) |
| **Buffer** | `extraBufferCapacity` cho buffering |
| **Backpressure** | `onBufferOverflow`: SUSPEND / DROP_OLDEST / DROP_LATEST |
| **New collectors** | Nhận N giá trị cuối trong replay cache |

```kotlin
val sharedFlow = MutableSharedFlow<String>(
    replay = 0,                           // no cache for new subscribers
    extraBufferCapacity = 1,              // 1 slot buffer before suspending
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
```

---

### Tóm tắt so sánh

| | Cold Flow | StateFlow | SharedFlow |
|---|-----------|-----------|------------|
| **Loại** | Cold | Hot | Hot |
| **Initial value** | Không | Bắt buộc | Không |
| **Replay** | Full (re-execute) | 1 (current) | Cấu hình (0..N) |
| **Deduplication** | Không | Có (equals) | Không |
| **Use case chính** | Data streams | UI State | One-time Events |
| **Thay thế** | — | LiveData | EventBus/Channel |

---

## 2. Bản chất & Cơ chế hoạt động

### 2.1 Cold Flow — Cơ chế bên dưới

Hàm `flow {}` builder tạo ra một `SafeFlow` object — **không có gì chạy cả**. Execution chỉ bắt đầu khi `collect {}` được gọi:

```
Collector gọi collect()
        │
        ▼
  Tạo coroutine mới
        │
        ▼
  Thực thi block flow { } từ đầu
        │
   ┌────▼────┐
   │ emit(v) │  ←── producer SUSPEND ở đây
   └────┬────┘      chờ consumer xử lý xong
        │
        ▼
  Consumer nhận giá trị
        │
        ▼
  Producer RESUME → tiếp tục block
```

Đây là mô hình **sequential rendezvous**: producer và consumer đồng bộ với nhau. Không có giá trị nào bị "đệm" mặc định — mỗi `emit()` đợi consumer xử lý xong.

**Context preservation:** Cold flow mặc định chạy trong **cùng coroutine context** với collector. Dùng `flowOn(Dispatcher.IO)` để chuyển upstream sang thread khác.

```kotlin
// flowOn chỉ ảnh hưởng upstream (trước nó), KHÔNG ảnh hưởng downstream
repository.getUsers()           // chạy trên IO
    .map { transform(it) }      // chạy trên IO
    .flowOn(Dispatchers.IO)     // ← ranh giới context
    .collect { updateUI(it) }   // chạy trên Main (caller's context)
```

---

### 2.2 StateFlow — Cơ chế bên dưới

```
MutableStateFlow<T>
├── _state: AtomicReference<StateFlowSlot[]>    // collector slots
├── _value: Any (volatile via Kotlin atomic)    // current value
└── sequence: Int (version counter)

Khi set value:
    newValue.equals(oldValue)?
    ├── YES → return (không emit, không notify)
    └── NO  → update _value
              increment sequence
              notify all collector slots (wake up suspended collectors)
```

**Conflation in action:**
```
Emitter tốc độ cao:    1 → 2 → 3 → 4 → 5
Collector tốc độ thấp:                 5
                  (intermediate values 1,2,3,4 bị bỏ qua)
```

StateFlow là **conflated** — khi collector không kịp xử lý, nó chỉ nhận giá trị mới nhất. Đây là lý do StateFlow phù hợp với UI state (UI chỉ cần render trạng thái hiện tại).

---

### 2.3 SharedFlow — Cơ chế bên dưới

```
MutableSharedFlow<T>(replay=2, extraBufferCapacity=1)

Buffer layout:
┌─────────────────────────────────────────┐
│  Replay cache (size=2) │ Extra buf (1)  │
│  [v3]  [v4]           │    [v5]        │
└─────────────────────────────────────────┘
         ↑                      ↑
   New collectors          Active collectors
   receive these           processing here

Each collector has cursor:
  Collector A: points to [v4]
  Collector B: points to [v5]
  New Collector C: replays [v3][v4] then waits
```

**Backpressure handling với `onBufferOverflow`:**

```
Buffer full situation:

DROP_OLDEST:  [v1][v2][v3] + emit(v4) → [v2][v3][v4]  // v1 dropped
DROP_LATEST:  [v1][v2][v3] + emit(v4) → [v1][v2][v3]  // v4 dropped, tryEmit=false
SUSPEND:      [v1][v2][v3] + emit(v4) → emitter suspends until space available
```

---

### 2.4 Sơ đồ luồng dữ liệu tổng quát

```
COLD FLOW (Regular Flow):
─────────────────────────
Repository ──flow{emit}──► [no buffer] ──collect──► ViewModel/UI
                              │
                    (starts only on collect)
                    (each collector: independent execution)


HOT FLOW — StateFlow:
──────────────────────
               ┌──────────────────────┐
               │  StateFlow           │
               │  value: UiState      │─────────► Collector A (UI)
ViewModel ────►│  (always active)     │─────────► Collector B (Test)
               │  replay=1 (current)  │
               └──────────────────────┘
                    ↑
              .value = newState


HOT FLOW — SharedFlow:
───────────────────────
               ┌──────────────────────┐
               │  SharedFlow          │
               │  replay=0, buf=1     │─────────► Collector A (UI)
ViewModel ────►│  (always active)     │─────────► (event consumed once)
               │  no sticky value     │
               └──────────────────────┘
                    ↑
              .emit(event)
```

---

## 3. Giải quyết bài toán gì?

### 3.1 Trước khi có Flow — Lịch sử

#### Giai đoạn 1: Callbacks
```kotlin
// Trước 2018 — callback hell
fun loadUsers(callback: (List<User>?, Throwable?) -> Unit) {
    Thread {
        try {
            val users = api.getUsers().execute().body()
            Handler(Looper.getMainLooper()).post { callback(users, null) }
        } catch (e: Exception) {
            Handler(Looper.getMainLooper()).post { callback(null, e) }
        }
    }.start()
}
// Vấn đề: callback hell, no cancellation, thread management thủ công
```

#### Giai đoạn 2: RxJava (2016-2020)
```kotlin
// RxJava — mạnh nhưng nặng
class UserRepository {
    fun getUsers(): Observable<List<User>> =
        Observable.fromCallable { api.getUsers().execute().body()!! }
            .subscribeOn(Schedulers.io())
}

// Trong ViewModel
class UserViewModel : ViewModel() {
    private val compositeDisposable = CompositeDisposable()

    fun loadUsers() {
        repository.getUsers()
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(
                { users -> _uiState.value = UiState.Success(users) },
                { error -> _uiState.value = UiState.Error(error.message) }
            )
            .also { compositeDisposable.add(it) }  // manual cleanup!
    }

    override fun onCleared() {
        compositeDisposable.clear()  // phải nhớ gọi!
    }
}
```

**Nhược điểm RxJava:**
- Dependency nặng (`rxjava`, `rxandroid`, `rxkotlin` ~500KB)
- Java API — ít idiomatic với Kotlin
- `CompositeDisposable` phải quản lý thủ công → dễ leak
- Không tích hợp native với coroutines
- Learning curve cao (100+ operators)
- `Subject` (Hot Observable) hay bị dùng sai cách

#### Giai đoạn 3: LiveData (2018-2021)
```kotlin
// LiveData — đơn giản nhưng giới hạn
class UserViewModel : ViewModel() {
    // liveData builder auto-cancels when ViewModel cleared
    val users: LiveData<List<User>> = liveData(Dispatchers.IO) {
        val data = repository.getUsers()
        emit(data)
    }
}

// Trong Fragment
viewModel.users.observe(viewLifecycleOwner) { users ->
    adapter.submitList(users)
}
```

**Nhược điểm LiveData:**

| Vấn đề | Mô tả |
|--------|-------|
| **UI-layer only** | Cần `LifecycleOwner` → không dùng được ở Data/Domain layer |
| **Always sticky** | Observer mới luôn nhận giá trị cuối → navigation event bị trigger lại sau rotate |
| **Limited operators** | `map`, `switchMap` nghèo nàn so với Flow operators |
| **Not coroutine-native** | Không suspend, không backpressure |
| **Main thread only** | `setValue` phải gọi từ Main thread, `postValue` thì async nhưng may drop values |
| **Hard to combine** | `MediatorLiveData` phức tạp để combine nhiều sources |

**Sticky event bug với LiveData:**
```kotlin
// Bug kinh điển với LiveData
val navigationEvent: LiveData<Screen> = MutableLiveData()

// ViewModel emit navigate to Detail
(navigationEvent as MutableLiveData).value = Screen.Detail

// User xoay màn hình
// Fragment mới observe → NGAY LẬP TỨC nhận lại Screen.Detail → navigate lại!
```

---

### 3.2 Kotlin Flow giải quyết như thế nào?

```
Vấn đề              RxJava/LiveData           Kotlin Flow
────────────────────────────────────────────────────────────
Lifecycle           CompositeDisposable        Structured Concurrency
                    (thủ công)                 (tự động cancel)

Threading           subscribeOn/observeOn      flowOn / Dispatcher
                    (phức tạp)                 (đơn giản)

Backpressure        Flowable / Subject         Built-in operators
                    (cần hiểu riêng)           (buffer, conflate)

Operators           100+ Java-style ops        Kotlin idiomatic ops

Testing             RxJava TestObserver        Turbine / runTest
                    (boilerplate nặng)         (đơn giản)

Interop             RxJava ↔ Coroutines        Native coroutines
                    cần adapter                (suspend + Flow)

Use outside UI      LiveData: KHÔNG            Flow: CÓ (any layer)
```

---

## 4. Ứng dụng thực tế & Best Practices

### 4.1 Khi nào dùng loại nào?

```
Cần data stream từ Room/Network?
└── Trả về Cold Flow<T> từ Repository
    (Room tự emit lại khi DB thay đổi)

Cần expose state cho UI trong ViewModel?
└── Dùng StateFlow<UiState>
    - Có initial value (Loading)
    - New collector nhận current state ngay
    - Không sticky-event bug vì state là idempotent
    - stateIn(WhileSubscribed(5000)) để tối ưu resource

Cần gửi one-time event (navigation, snackbar)?
└── Dùng SharedFlow<UiEvent>(replay=0)
    - Không cached, consumed once
    - Collect qua LaunchedEffect trong Compose
    - Chú ý: event có thể mất khi không có collector (xem FAQ)
```

---

### 4.2 `stateIn(WhileSubscribed(5000))` — Best Practice quan trọng nhất

```kotlin
// Pattern chuẩn — Google Architecture Samples
val uiState: StateFlow<UiState> = repository.getData()
    .map { UiState.Success(it) }
    .catch { emit(UiState.Error(it.message)) }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000L),
        initialValue = UiState.Loading
    )
```

**Tại sao `5000ms`?**

```
Timeline:
0ms:    UI visible, collector bắt đầu → Flow ACTIVE (upstream chạy, network/DB query)
t=Xms:  User nhấn Home / switch app → UI không visible → collector dừng
t=X+1ms: Flow KHÔNG dừng ngay — đợi 5000ms
t=X+4999ms: User quay lại app → collector resume → Flow vẫn ACTIVE, không restart
t=X+5001ms: Flow DỪNG (nếu user không quay lại) → tiết kiệm resource

Tại sao 5000ms mà không phải 0ms?
  - 0ms: configuration change (rotate) cũng khiến flow restart → tốn kém
  - Configuration change thường mất < 1-2s
  - 5000ms đủ để bao phủ cả trường hợp "quick task switch"
  - Google đề xuất sau khi đo lường thực tế
```

**So sánh các `SharingStarted` strategies:**

| Strategy | Bắt đầu | Dừng | Dùng khi |
|----------|---------|------|----------|
| `Eagerly` | Ngay khi tạo | Không bao giờ | Cần data sẵn sàng ngay (hiếm) |
| `Lazily` | Khi có collector đầu tiên | Không bao giờ | Data quan trọng, không muốn restart |
| `WhileSubscribed(5000)` | Khi có collector | 5s sau khi hết collector | **UI State — khuyến nghị** |

---

### 4.3 `collectAsStateWithLifecycle()` — Bắt buộc thay `collectAsState()`

```kotlin
// BAD — collectAsState() vẫn collect khi app ở background
val uiState by viewModel.uiState.collectAsState()

// GOOD — collectAsStateWithLifecycle() dừng khi lifecycle < STARTED
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
// Requires: androidx.lifecycle:lifecycle-runtime-compose
```

`collectAsStateWithLifecycle()` tự động stop collection khi:
- App đi vào background (lifecycle xuống dưới `STARTED`)
- Fragment/Activity bị destroyed

Kết hợp với `WhileSubscribed(5000)`, upstream flow sẽ dừng hoàn toàn khi app background → tiết kiệm battery và network.

---

### 4.4 Common Pitfalls & Anti-patterns

#### Anti-pattern 1: Dùng SharedFlow thay StateFlow cho UI State
```kotlin
// BAD ❌ — mất state sau rotate
private val _uiState = MutableSharedFlow<UiState>()
val uiState: SharedFlow<UiState> = _uiState.asSharedFlow()
// Sau rotation, Compose recompose, collect lại → không nhận được state hiện tại
// Phải emit lại thủ công → phức tạp, error-prone

// GOOD ✅
private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
val uiState: StateFlow<UiState> = _uiState.asStateFlow()
// Sau rotation → nhận ngay current state
```

#### Anti-pattern 2: Dùng StateFlow cho One-time Events
```kotlin
// BAD ❌ — sticky event bug
private val _navEvent = MutableStateFlow<NavEvent?>(null)

// ViewModel emits: _navEvent.value = NavEvent.GoToDetail(id)
// User navigates to Detail...
// User presses back → returns to List
// New observer registers → receives NavEvent.GoToDetail AGAIN → navigate lại!

// Phải reset thủ công:
fun onEventConsumed() { _navEvent.value = null }  // error-prone, race condition

// GOOD ✅ — dùng SharedFlow với replay=0
private val _events = MutableSharedFlow<NavEvent>(replay = 0, extraBufferCapacity = 1)
val events = _events.asSharedFlow()
```

#### Anti-pattern 3: Collect trong `lifecycleScope.launch` không có `repeatOnLifecycle`
```kotlin
// BAD ❌ — continues collecting in background, wasting resources
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    lifecycleScope.launch {
        viewModel.uiState.collect { render(it) }  // runs even in background!
    }
}

// GOOD ✅ — trong Fragment/Activity (nếu không dùng Compose)
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    viewLifecycleOwner.lifecycleScope.launch {
        viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
            viewModel.uiState.collect { render(it) }
        }
    }
}

// BEST ✅ — trong Compose
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

#### Anti-pattern 4: Tạo StateFlow/SharedFlow trong `@Composable`
```kotlin
// BAD ❌ — tạo mới mỗi recomposition
@Composable
fun UserScreen() {
    val stateFlow = MutableStateFlow(listOf<User>())  // tạo mới mỗi recompose!
    // ...
}

// GOOD ✅ — state nằm trong ViewModel, Composable chỉ collect
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
}
```

---

## 5. Hướng dẫn triển khai từng bước

### 5.1 Dependencies

```kotlin
// build.gradle.kts (app module)
dependencies {
    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")

    // Lifecycle — collectAsStateWithLifecycle
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.8.4")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.8.4")

    // Compose BOM
    implementation(platform("androidx.compose:compose-bom:2024.06.00"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.runtime:runtime")

    // DI (optional for full example)
    implementation("com.google.dagger:hilt-android:2.51.1")
    ksp("com.google.dagger:hilt-compiler:2.51.1")
    implementation("androidx.hilt:hilt-navigation-compose:1.2.0")

    // Testing
    testImplementation("app.cash.turbine:turbine:1.1.0")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")
    testImplementation("junit:junit:4.13.2")
    testImplementation("com.google.truth:truth:1.4.2")
}
```

---

### 5.2 Data Layer — Cold Flow từ DAO

```kotlin
// domain/model/User.kt
data class User(
    val id: Long,
    val name: String,
    val email: String
)

// data/local/UserEntity.kt
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: Long,
    val name: String,
    val email: String
)

fun UserEntity.toDomain(): User = User(id, name, email)
fun User.toEntity(): UserEntity = UserEntity(id, name, email)

// data/local/UserDao.kt
@Dao
interface UserDao {
    // Room tự động trả về Cold Flow<T>
    // Khi DB thay đổi (insert/update/delete), Flow emit lại
    @Query("SELECT * FROM users ORDER BY name ASC")
    fun observeAllUsers(): Flow<List<UserEntity>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(users: List<UserEntity>)

    @Query("DELETE FROM users")
    suspend fun deleteAll()
}

// data/remote/UserApiService.kt
interface UserApiService {
    @GET("users")
    suspend fun fetchUsers(): List<UserDto>
}

// data/repository/UserRepositoryImpl.kt
class UserRepositoryImpl @Inject constructor(
    private val dao: UserDao,
    private val api: UserApiService,
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher
) : UserRepository {

    // Cold Flow: Room DAO → map entity → domain model
    // flowOn chuyển upstream (Room query) lên IO thread
    override fun getUsers(): Flow<List<User>> =
        dao.observeAllUsers()
            .map { entities -> entities.map { it.toDomain() } }
            .flowOn(ioDispatcher)

    // suspend function cho one-shot network call
    override suspend fun refreshUsers() {
        withContext(ioDispatcher) {
            val dtos = api.fetchUsers()
            val entities = dtos.map { it.toEntity() }
            dao.insertAll(entities)
        }
    }
}
```

---

### 5.3 Domain Layer — Interface

```kotlin
// domain/repository/UserRepository.kt
interface UserRepository {
    // Cold Flow — emit khi có collector, re-emit khi DB thay đổi
    fun getUsers(): Flow<List<User>>

    // suspend — one-shot operation
    suspend fun refreshUsers()
}
```

---

### 5.4 ViewModel Layer — StateFlow + SharedFlow

```kotlin
// presentation/userlist/UserListUiState.kt

// Sealed interface — compiler enforce exhaustive when()
sealed interface UserListUiState {
    data object Loading : UserListUiState
    data object Empty : UserListUiState
    data class Success(val users: List<User>) : UserListUiState
    data class Error(val message: String) : UserListUiState
}

// presentation/userlist/UserListEvent.kt

// One-time events — consumed once, không sticky
sealed interface UserListEvent {
    data class NavigateToDetail(val userId: Long) : UserListEvent
    data class ShowSnackbar(val message: String) : UserListEvent
}
```

```kotlin
// presentation/userlist/UserListViewModel.kt
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {

    // ────────────────────────────────────────────────────
    // UI STATE — StateFlow
    // - Cold Flow từ repository được convert sang Hot StateFlow
    // - stateIn: upstream chạy khi có collector, dừng sau 5s
    // - initialValue: Loading (tránh null state trong Compose)
    // ────────────────────────────────────────────────────
    val uiState: StateFlow<UserListUiState> = repository
        .getUsers()
        .map { users ->
            if (users.isEmpty()) UserListUiState.Empty
            else UserListUiState.Success(users)
        }
        .catch { throwable ->
            // catch chỉ bắt upstream errors (từ repository trở lên)
            emit(UserListUiState.Error(throwable.localizedMessage ?: "Unknown error"))
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000L),
            initialValue = UserListUiState.Loading
        )

    // ────────────────────────────────────────────────────
    // ONE-TIME EVENTS — SharedFlow
    // - replay=0: không cache, event consumed once
    // - extraBufferCapacity=1: tránh suspend khi collector chưa collect kịp
    // - DROP_OLDEST: nếu buffer đầy, bỏ event cũ nhất (tránh suspend ViewModel)
    // ────────────────────────────────────────────────────
    private val _events = MutableSharedFlow<UserListEvent>(
        replay = 0,
        extraBufferCapacity = 1,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val events: SharedFlow<UserListEvent> = _events.asSharedFlow()

    // Private loading state để track refresh
    private val _isRefreshing = MutableStateFlow(false)
    val isRefreshing: StateFlow<Boolean> = _isRefreshing.asStateFlow()

    // ────────────────────────────────────────────────────
    // USER ACTIONS (Intents)
    // ────────────────────────────────────────────────────

    fun refresh() {
        if (_isRefreshing.value) return  // debounce multiple taps
        viewModelScope.launch {
            _isRefreshing.value = true
            try {
                repository.refreshUsers()
                _events.emit(UserListEvent.ShowSnackbar("Refreshed successfully"))
            } catch (e: Exception) {
                _events.emit(
                    UserListEvent.ShowSnackbar("Refresh failed: ${e.localizedMessage}")
                )
            } finally {
                _isRefreshing.value = false
            }
        }
    }

    fun onUserClick(user: User) {
        viewModelScope.launch {
            _events.emit(UserListEvent.NavigateToDetail(user.id))
        }
    }
}
```

---

### 5.5 UI Layer — Jetpack Compose

```kotlin
// presentation/userlist/UserListScreen.kt

@Composable
fun UserListScreen(
    onNavigateToDetail: (Long) -> Unit,
    viewModel: UserListViewModel = hiltViewModel()
) {
    // ─────────────────────────────────────────────
    // collectAsStateWithLifecycle():
    // - Dừng collect khi lifecycle < STARTED (app background)
    // - Kết hợp với WhileSubscribed(5000) → upstream hoàn toàn dừng sau 5s
    // ─────────────────────────────────────────────
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val isRefreshing by viewModel.isRefreshing.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }

    // ─────────────────────────────────────────────
    // LaunchedEffect cho one-time events:
    // - key = Unit: coroutine chạy suốt vòng đời của composable
    // - Bị cancel khi composable leave composition
    // - SharedFlow.collect() suspend here, wakes up on new event
    // ─────────────────────────────────────────────
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is UserListEvent.NavigateToDetail ->
                    onNavigateToDetail(event.userId)
                is UserListEvent.ShowSnackbar ->
                    snackbarHostState.showSnackbar(
                        message = event.message,
                        duration = SnackbarDuration.Short
                    )
            }
        }
    }

    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) },
        topBar = {
            TopAppBar(title = { Text("Users") })
        },
        floatingActionButton = {
            FloatingActionButton(onClick = viewModel::refresh) {
                if (isRefreshing) {
                    CircularProgressIndicator(
                        modifier = Modifier.size(24.dp),
                        strokeWidth = 2.dp,
                        color = MaterialTheme.colorScheme.onPrimaryContainer
                    )
                } else {
                    Icon(Icons.Default.Refresh, contentDescription = "Refresh")
                }
            }
        }
    ) { paddingValues ->
        // Stateless composable — dễ test và preview
        UserListContent(
            uiState = uiState,
            onUserClick = viewModel::onUserClick,
            modifier = Modifier.padding(paddingValues)
        )
    }
}

// ─────────────────────────────────────────────────────
// Stateless composable: nhận state + callbacks, không biết về ViewModel
// → Dễ test với ComposeTestRule
// → Dễ Preview với fake data
// ─────────────────────────────────────────────────────
@Composable
internal fun UserListContent(
    uiState: UserListUiState,
    onUserClick: (User) -> Unit,
    modifier: Modifier = Modifier
) {
    when (uiState) {
        UserListUiState.Loading -> LoadingContent(modifier)
        UserListUiState.Empty -> EmptyContent(modifier)
        is UserListUiState.Success -> SuccessContent(uiState.users, onUserClick, modifier)
        is UserListUiState.Error -> ErrorContent(uiState.message, modifier)
    }
}

@Composable
private fun LoadingContent(modifier: Modifier = Modifier) {
    Box(modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
        CircularProgressIndicator()
    }
}

@Composable
private fun EmptyContent(modifier: Modifier = Modifier) {
    Box(modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            Icon(
                Icons.Default.Person,
                contentDescription = null,
                modifier = Modifier.size(64.dp),
                tint = MaterialTheme.colorScheme.outline
            )
            Spacer(Modifier.height(16.dp))
            Text(
                "No users yet. Pull to refresh.",
                style = MaterialTheme.typography.bodyLarge,
                color = MaterialTheme.colorScheme.outline
            )
        }
    }
}

@Composable
private fun SuccessContent(
    users: List<User>,
    onUserClick: (User) -> Unit,
    modifier: Modifier = Modifier
) {
    LazyColumn(modifier = modifier) {
        items(
            items = users,
            key = { user -> user.id }   // stable key: minimize recomposition
        ) { user ->
            UserItem(user = user, onClick = { onUserClick(user) })
            HorizontalDivider()
        }
    }
}

@Composable
private fun UserItem(
    user: User,
    onClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    ListItem(
        modifier = modifier.clickable(onClick = onClick),
        headlineContent = { Text(user.name) },
        supportingContent = { Text(user.email) },
        leadingContent = {
            Icon(Icons.Default.Person, contentDescription = null)
        }
    )
}

@Composable
private fun ErrorContent(
    message: String,
    modifier: Modifier = Modifier
) {
    Box(modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
        Text(
            text = "Error: $message",
            color = MaterialTheme.colorScheme.error,
            style = MaterialTheme.typography.bodyLarge
        )
    }
}

// Preview cho stateless composable
@Preview(showBackground = true)
@Composable
private fun UserListContentSuccessPreview() {
    val fakeUsers = listOf(
        User(1L, "Alice Johnson", "alice@example.com"),
        User(2L, "Bob Smith", "bob@example.com"),
        User(3L, "Charlie Brown", "charlie@example.com"),
    )
    MaterialTheme {
        UserListContent(
            uiState = UserListUiState.Success(fakeUsers),
            onUserClick = {}
        )
    }
}
```

---

### 5.6 Unit Test — ViewModel với Turbine

```kotlin
// test/presentation/userlist/UserListViewModelTest.kt
@OptIn(ExperimentalCoroutinesApi::class)
class UserListViewModelTest {

    @get:Rule
    val mainCoroutineRule = MainCoroutineRule()

    private lateinit var fakeRepository: FakeUserRepository
    private lateinit var viewModel: UserListViewModel

    @Before
    fun setUp() {
        fakeRepository = FakeUserRepository()
        viewModel = UserListViewModel(fakeRepository)
    }

    @Test
    fun `initial uiState is Loading`() = runTest {
        // StateFlow initial value — check via .value (no collect needed)
        assertThat(viewModel.uiState.value).isEqualTo(UserListUiState.Loading)
    }

    @Test
    fun `uiState emits Success when repository emits users`() = runTest {
        val users = listOf(User(1L, "Alice", "alice@example.com"))

        // Turbine: .test {} block — clean API cho testing flows
        viewModel.uiState.test {
            // awaitItem() suspends until next emission
            assertThat(awaitItem()).isEqualTo(UserListUiState.Loading)

            // Simulate repository emitting data
            fakeRepository.emit(users)

            assertThat(awaitItem()).isEqualTo(UserListUiState.Success(users))
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `uiState emits Empty when repository emits empty list`() = runTest {
        viewModel.uiState.test {
            assertThat(awaitItem()).isEqualTo(UserListUiState.Loading)
            fakeRepository.emit(emptyList())
            assertThat(awaitItem()).isEqualTo(UserListUiState.Empty)
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `uiState emits Error when repository throws`() = runTest {
        viewModel.uiState.test {
            assertThat(awaitItem()).isEqualTo(UserListUiState.Loading)
            fakeRepository.emitError(RuntimeException("Network error"))
            val errorState = awaitItem()
            assertThat(errorState).isInstanceOf(UserListUiState.Error::class.java)
            assertThat((errorState as UserListUiState.Error).message).contains("Network error")
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `onUserClick emits NavigateToDetail event`() = runTest {
        val user = User(42L, "Bob", "bob@example.com")

        viewModel.events.test {
            viewModel.onUserClick(user)
            assertThat(awaitItem())
                .isEqualTo(UserListEvent.NavigateToDetail(42L))
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `refresh updates isRefreshing state`() = runTest {
        viewModel.isRefreshing.test {
            assertThat(awaitItem()).isFalse()  // initial
            viewModel.refresh()
            assertThat(awaitItem()).isTrue()   // refreshing
            assertThat(awaitItem()).isFalse()  // done
            cancelAndIgnoreRemainingEvents()
        }
    }
}

// ─────────────────────────────────────────────────────
// Fake Repository — không dùng Mockito/MockK để tránh
// phụ thuộc reflection, test nhanh hơn
// ─────────────────────────────────────────────────────
class FakeUserRepository : UserRepository {

    private val _users = MutableSharedFlow<List<User>>(replay = 1)

    fun emit(users: List<User>) {
        _users.tryEmit(users)
    }

    fun emitError(error: Throwable) {
        // Simulate error by using a flow that throws
        // We'll override getUsers() with error-throwing flow
        errorToThrow = error
        _users.tryEmit(emptyList())  // trigger re-emission path
    }

    private var errorToThrow: Throwable? = null
    var refreshCallCount = 0

    override fun getUsers(): Flow<List<User>> = flow {
        errorToThrow?.let { throw it }
        emitAll(_users)
    }

    override suspend fun refreshUsers() {
        refreshCallCount++
    }
}

// ─────────────────────────────────────────────────────
// MainCoroutineRule — replace Dispatchers.Main for tests
// ─────────────────────────────────────────────────────
@OptIn(ExperimentalCoroutinesApi::class)
class MainCoroutineRule : TestWatcher() {
    val testDispatcher = UnconfinedTestDispatcher()

    override fun starting(description: Description?) {
        Dispatchers.setMain(testDispatcher)
    }

    override fun finished(description: Description?) {
        Dispatchers.resetMain()
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế

### Q1: Tại sao `StateFlow` không emit khi tôi set cùng giá trị hai lần?

**Câu trả lời:**

`StateFlow` dùng `equals()` để so sánh giá trị trước khi emit. Nếu `newValue == currentValue`, **không có emission nào xảy ra**.

```kotlin
val flow = MutableStateFlow(0)
flow.value = 5
flow.value = 5  // KHÔNG emit — 5 == 5

// Cạm bẫy với data class:
data class UiState(val count: Int, val isLoading: Boolean)
flow.value = UiState(1, false)
flow.value = UiState(1, false)  // KHÔNG emit — structural equality

// Cạm bẫy với mutable object (KHÔNG dùng):
class MutableState(var count: Int)
val mutableFlow = MutableStateFlow(MutableState(0))
mutableFlow.value.count = 5
// Flow KHÔNG detect thay đổi vì reference vẫn cũ!
// Phải reassign: mutableFlow.value = MutableState(5)
```

**Rule:** Luôn dùng **immutable data class** với StateFlow. Thay đổi → tạo copy mới.

---

### Q2: `stateIn` vs `shareIn` — khác nhau thế nào, khi nào dùng cái nào?

| | `stateIn()` | `shareIn()` |
|---|------------|------------|
| **Trả về** | `StateFlow<T>` | `SharedFlow<T>` |
| **Initial value** | Bắt buộc | Không có |
| **Current value** | Có (`.value`) | Không có |
| **Replay** | Cố định = 1 | Cấu hình được |
| **Deduplication** | Có (equals) | Không |
| **Dùng cho** | UI state | Multicast events/data |

```kotlin
// stateIn — cho UI state
val uiState: StateFlow<UiState> = repository.getData()
    .stateIn(viewModelScope, WhileSubscribed(5000), UiState.Loading)

// shareIn — cho multicasting data đến nhiều consumer mà không restart upstream
val sharedData: SharedFlow<Data> = repository.getExpensiveData()
    .shareIn(viewModelScope, WhileSubscribed(5000), replay = 1)
// Nếu 2 collectors cùng collect sharedData → upstream chỉ chạy 1 lần
```

---

### Q3: SharedFlow `replay=0` + navigation event bị mất khi rotate — xử lý thế nào?

**Vấn đề:**
```
1. ViewModel emit NavigateToDetail
2. Compose recomposition đang diễn ra (sau rotate)
3. LaunchedEffect bị cancel → collector tạm dừng
4. SharedFlow(replay=0) không cache → event BỊ MẤT
5. LaunchedEffect restart → bắt đầu collect từ event mới
```

**Các giải pháp (theo thứ tự ưu tiên):**

**Giải pháp 1 — Channel (khuyến nghị cho events thực sự one-shot):**
```kotlin
// ViewModel
private val _events = Channel<UserListEvent>(Channel.BUFFERED)
val events: Flow<UserListEvent> = _events.receiveAsFlow()

fun onUserClick(user: User) {
    viewModelScope.launch {
        _events.send(UserListEvent.NavigateToDetail(user.id))
    }
}
// Channel buffer event cho đến khi có collector consume
```

**Giải pháp 2 — Event trong UiState (Google Architecture Samples approach):**
```kotlin
data class UserListUiState(
    val users: List<User> = emptyList(),
    val navigationEvent: NavEvent? = null  // null = no event
)

// Sau khi UI consume:
fun onNavigationEventConsumed() {
    _uiState.update { it.copy(navigationEvent = null) }
}
```

**Giải pháp 3 — SharedFlow `replay=1` với consumed tracking:**
```kotlin
data class OneTimeEvent<T>(val content: T, val hasBeenConsumed: Boolean = false)

private val _events = MutableSharedFlow<OneTimeEvent<NavEvent>>(replay = 1)

// UI consume:
LaunchedEffect(Unit) {
    viewModel.events.collect { event ->
        if (!event.hasBeenConsumed) {
            viewModel.markEventConsumed()
            handleEvent(event.content)
        }
    }
}
```

---

### Q4: Sự khác biệt giữa `conflate()` và `StateFlow` — tại sao không dùng `conflate()` cho mọi thứ?

```kotlin
// conflate() — operator trên Cold Flow
val coldFlow = flow { /* emit fast */ }.conflate()
// → bỏ qua intermediate values khi collector chậm
// → vẫn là COLD, mỗi collector independent

// StateFlow — Hot Flow
val stateFlow = MutableStateFlow(initial)
// → LUÔN active, nhiều collector chia sẻ cùng stream
// → Có .value, replay=1, deduplication
// → Phù hợp cho "current state"

// Dùng conflate() khi:
// - Bạn muốn Cold Flow nhưng bỏ qua intermediate values khi collector chậm
// - Không cần current value, không cần Hot stream

// Dùng StateFlow khi:
// - Cần current state luôn có mặt (.value)
// - Nhiều collectors chia sẻ cùng state
// - Cần Hot stream (active kể cả không có collector)
```

---

### Q5: Coroutine leak — làm sao phát hiện và ngăn chặn?

**Leak xảy ra khi:**
```kotlin
// BAD ❌ — GlobalScope không bị cancel khi ViewModel cleared
class BadViewModel : ViewModel() {
    init {
        GlobalScope.launch {  // LEAK! Không bao giờ cancel
            repository.getUsers().collect { /* ... */ }
        }
    }
}

// BAD ❌ — lifecycleScope không có repeatOnLifecycle
class BadFragment : Fragment() {
    override fun onViewCreated(view: View, state: Bundle?) {
        lifecycleScope.launch {
            viewModel.uiState.collect { /* continues in background */ }
        }
    }
}
```

**Phát hiện:**
- Android Studio: Memory Profiler → track coroutine jobs
- `DebugProbes.dumpCoroutines()` từ `kotlinx-coroutines-debug`
- LeakCanary: detect ViewModel/Fragment leaks

**Phòng chặn:**
```kotlin
// GOOD ✅ — viewModelScope tự cancel khi ViewModel.onCleared()
class GoodViewModel : ViewModel() {
    val uiState = repository.getUsers()
        .stateIn(viewModelScope, WhileSubscribed(5000), emptyList())
}

// GOOD ✅ — Compose handles lifecycle automatically
@Composable
fun Screen(viewModel: GoodViewModel = hiltViewModel()) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    // collectAsStateWithLifecycle respects lifecycle automatically
}
```

---

## Tóm tắt — Bảng quyết định nhanh

```
Tình huống                              Giải pháp
───────────────────────────────────────────────────────────────────
Room/Network data stream                Flow<T> từ DAO/Repository
UI state trong ViewModel                StateFlow<UiState> + stateIn(WhileSubscribed(5000))
One-time event (navigate, snackbar)     Channel<Event> (hoặc SharedFlow replay=0)
Cold → Hot conversion                   stateIn() hoặc shareIn()
Collect trong Compose                   collectAsStateWithLifecycle()
Multiple collectors, share upstream     shareIn()
Combine 2 flows                         combine() hoặc zip()
Restart flow when input changes         flatMapLatest()
```

---

## Bài tiếp theo

➡️ [Bài 03 — Flow Operators: map, flatMapLatest, combine, zip, debounce](03-flow-operators.md)

---

*Cập nhật lần cuối: September 2026 | Kotlin 2.0 | Coroutines 1.8 | Compose BOM 2024.06*
