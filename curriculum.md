# Curriculum — Android Modern Development

> Full roadmap 21 bài giảng, từ Coroutines cơ bản đến Advanced Patterns production-ready.

---

> 💡 **Tài liệu nền tảng chuẩn mực:**
> - [**Kotlin Coroutines & Flow — JetBrains Official Guide (Dịch chuẩn 1:1)**](official-coroutines-guide/README.md) bám sát cẩm nang cốt lõi ngôn ngữ từ JetBrains.
> - [**Android Coroutines & Flow — Google Official Guide (Dịch chuẩn 1:1)**](official-android-coroutines-flow/README.md) bám sát tài liệu chính thức từ Google Android Developers (`developer.android.com`).
> - [**Jetpack Compose State — Google Official Guide (Dịch chuẩn 1:1)**](official-android-compose-state/README.md) bám sát tài liệu chính thức về State, State Hoisting, Saving State, Phases & Snapshot System từ Google Android Developers.

---

## Module 1 — Kotlin Coroutines & Flow Foundation
*Nền tảng bắt buộc. Phải nắm vững trước khi học các module tiếp theo.*

### Bài 01 — Coroutines Basics ✅
**File:** `module-1-coroutines-flow/01-coroutines-basics.md`  
**Mục tiêu:** Hiểu suspend functions, CPS state machine, CoroutineScope, CoroutineContext, Dispatcher, Structured Concurrency.  
**Key topics:**
- `suspend` keyword và Continuation-Passing Style (CPS) State Machine
- Thread Stack vs Heap Frame — tại sao coroutines siêu nhẹ
- `CoroutineScope` vs `GlobalScope` — Structured Concurrency
- `CoroutineContext` = `Job` + `CoroutineDispatcher` + `CoroutineExceptionHandler` + `CoroutineName`
- `Dispatchers.Main`, `IO`, `Default`, `Unconfined`
- Job Hierarchy & Cancellation Cascading
- `launch` vs `async`/`await`
- Production Clean Architecture & Unit Test với `StandardTestDispatcher`

---

### Bài 02 — Cold Flow vs Hot Flow ✅
**File:** `module-1-coroutines-flow/02-cold-flow-vs-hot-flow.md`  
**Mục tiêu:** Phân biệt Cold/Hot flow; thành thạo StateFlow và SharedFlow.  
**Key topics:**
- Cold Flow: lazy execution, independent per-collector
- Hot Flow: active independently, multicast
- `StateFlow<T>`: UI state holder, conflated, always has value
- `SharedFlow<T>`: event bus, configurable replay/buffer
- `stateIn()` và `shareIn()` operators
- `WhileSubscribed(5000)` pattern

---

### Bài 03 — Flow Operators Deep Dive ✅
**File:** `module-1-coroutines-flow/03-flow-operators.md`  
**Mục tiêu:** Thành thạo các operator biến đổi, lọc, làm phẳng, gộp luồng, backpressure và giữ vững Context Preservation.  
**Key topics:**
- Intermediate (cold/lazy) vs Terminal (suspend) operators
- Lazy evaluation pipeline & Flattening: `flatMapLatest`, `flatMapMerge`, `flatMapConcat`
- Combining: `combine` vs `zip` vs `merge`
- Backpressure & Time: `debounce`, `sample`, `conflate`, `buffer`
- Context Preservation Invariant & Ranh giới `flowOn`
- Triển khai E-Commerce Search-as-you-type và Unit Test với `Turbine`

---

### Bài 04 — Flow trong Android Lifecycle ✅
**File:** `module-1-coroutines-flow/04-flow-lifecycle-android.md`  
**Mục tiêu:** Collect Flow an toàn tuyệt đối theo vòng đời Android, tránh rò rỉ tài nguyên, pin, network khi app ở background.  
**Key topics:**
- Android Lifecycle states & Lifecycle-aware collection
- Tại sao `collectAsState()` trong Compose nguy hiểm khi vào background
- `collectAsStateWithLifecycle()` — Tiêu chuẩn vàng của Jetpack Compose
- `repeatOnLifecycle` trong Fragment/Activity
- Bản chất hoãn hủy 5 giây của `stateIn(WhileSubscribed(5000))` khi xoay màn hình
- Triển khai Live GPS Tracking an toàn và Compose UI Test

---

### Bài 05 — Flow Exception Handling & Testing ✅
**File:** `module-1-coroutines-flow/05-flow-exception-handling.md`  
**Mục tiêu:** Xử lý lỗi chuẩn mực theo nguyên lý Exception Transparency, triển khai retry exponential backoff và viết Unit Test Flow với Turbine.  
**Key topics:**
- Exception Transparency — tại sao `catch` chỉ bắt upstream mà không nuốt lỗi downstream
- `CancellationException` là control signal, không phải lỗi logic
- `catch`, `onCompletion` operators
- Exponential Backoff Retry kết hợp Jitter
- `SupervisorJob` vs `Job` trong việc cô lập ngoại lệ
- Triển khai Crypto Ticker với cache fallback và Unit Testing với `Turbine`

---

## Module 2 — Architecture & ViewModel
*Áp dụng Flow vào Clean Architecture thực tế.*

### Bài 06 — ViewModel + Flow + Compose (UDF) ✅
**File:** `module-2-architecture/06-viewmodel-flow-compose-udf.md`  
**Mục tiêu:** Xây dựng kiến trúc UDF hoàn chỉnh: UiState (SSOT), UiAction, UiEffect (Channel), Atomic CAS update.  
**Key topics:**
- Unidirectional Data Flow (UDF): UI Action → ViewModel → State Flow → Compose UI
- Consolidated `UiState` data class bất biến — xóa bỏ state fragmentation
- `UiEffect` One-time events: tại sao dùng `Channel(BUFFERED)` thay vì StateFlow/SingleLiveEvent
- Thread-safety với Atomic CAS: Bản chất của `MutableStateFlow.update { copy(...) }`
- Pure Kotlin ViewModel (100% không Android dependencies) & JVM Unit Test với `Turbine`

---

### Bài 07 — Domain Layer & Use Cases với Flow ✅
**File:** `module-2-architecture/07-domain-layer-usecases.md`  
**Mục tiêu:** Thiết kế Use Cases đơn nhiệm (Single Responsibility), tái sử dụng business logic, cô lập hoàn toàn khỏi Android framework.  
**Key topics:**
- Vai trò của Domain Layer theo Google Architecture Guidelines (khi nào nên / không nên dùng)
- Phân định rạch ròi: Business Logic vs UI Logic vs Data Logic
- Kotlin convention `operator fun invoke()` linh hoạt
- Use Case làm Orchestrator điều phối đa Repositories
- Threading độc lập với `@DefaultDispatcher` & JVM Unit Test siêu tốc (10ms)

---

### Bài 08 — Repository Pattern với Flow (Offline-First) ✅
**File:** `module-2-architecture/08-repository-pattern-flow.md`  
**Mục tiêu:** Xây dựng tầng Data Layer chuẩn mực với Single Source of Truth (SSOT), kiến trúc Offline-First (Room + Retrofit).  
**Key topics:**
- Repository Pattern & Single Source of Truth (SSOT)
- Reactive Sync Pipeline: Room `InvalidationTracker` tự động phát hiện thay đổi bảng và emit qua Flow
- Data Mappers: Entity (Room) $\leftrightarrow$ DTO (Retrofit) $\leftrightarrow$ Domain Model
- Chiến lược giải quyết xung đột (Conflict Resolution) & Cập nhật lạc quan (Optimistic Updates)
- Triển khai News Feed Offline-First và Unit Test với `Turbine`

---

### Bài 09 — Paging 3 + Flow (Offline Cache) ✅
**File:** `module-2-architecture/09-paging3-flow.md`  
**Mục tiêu:** Load và hiển thị tập dữ liệu lớn mượt mà, kết hợp phân trang mạng và cache offline với RemoteMediator trong Compose.  
**Key topics:**
- Kiến trúc Paging 3: `PagingSource`, `RemoteMediator`, `Pager`, `PagingData`
- Bản chất bắt buộc của `.cachedIn(viewModelScope)` — bảo tồn snapshot khi xoay màn hình
- Phối hợp Network + Room qua bảng `remote_keys` trong SQLite
- Tích hợp Compose UI với `collectAsLazyPagingItems()`, Stable item keys
- Xử lý LoadState (`refresh` fullscreen vs `append` footer indicator & retry)

---

## Module 3 — Jetpack Compose Foundation
*Tư duy Compose khác hoàn toàn với XML — phải hiểu từ gốc.*

### Bài 10 — Composable Functions ✅
**File:** `module-3-compose-foundation/10-composable-functions.md`  
**Mục tiêu:** Hiểu bản chất Composable, Function coloring, Slot Table / Gap Buffer, Slot API Design System.  
**Key topics:**
- `@Composable` type system transformer ($composer & $changed bitmask)
- Composable lifecycle: Enter Composition → Recompose 0..N → Leave Composition
- Cấu trúc dữ liệu bên dưới: Slot Table & Gap Buffer O(1) mutations
- Idempotent & side-effect free composable contracts
- Slot API pattern (`header`, `content`, `actions`) & Design System Card
- `CompositionLocal`: `staticCompositionLocalOf` vs `compositionLocalOf`

---

### Bài 11 — Compose State ✅
**File:** `module-3-compose-foundation/11-compose-state.md`  
**Mục tiêu:** Quản lý State chuẩn mực, Snapshot State System (MVCC), State Hoisting, derivedStateOf.  
**Key topics:**
- `State<T>` vs `MutableState<T>` & Snapshot State System (Read/Write Tracking)
- Vòng đời lưu trữ: `remember` (RAM) vs `rememberSaveable` (Bundle) vs `ViewModel`
- Custom `Saver` interface (`mapSaver`) cho data class phức tạp
- State Hoisting: Chuyển đổi Stateful thành Stateless Composable
- `derivedStateOf` — Bản chất tối ưu hóa debounce snapshot mutations
- Triển khai Smart Registration Form và Compose UI Test

---

### Bài 12 — Recomposition & Stability ✅
**File:** `module-3-compose-foundation/12-recomposition-stability.md`  
**Mục tiêu:** Tối ưu hóa Recomposition, triệt tiêu giật lag với Stability System và Compose Compiler Metrics.  
**Key topics:**
- Smart Recomposition & Tính Skippable vs Restartable
- Vấn nạn `List<T>` Unstable trong Kotlin và giải pháp `ImmutableList` (`kotlinx.collections.immutable`)
- `@Immutable` vs `@Stable` annotations & Strong Skipping Mode
- Ổn định lambdas với method references và `remember`
- Bật và phân tích file Compose Compiler Metrics (`app_release-composables.txt`)
- Triển khai High-Performance Transaction Feed đạt chuẩn Skippable 100%

---

### Bài 13 — Side Effects trong Compose ✅
**File:** `module-3-compose-foundation/13-side-effects.md`  
**Mục tiêu:** Quản lý tác dụng phụ chuẩn xác theo vòng đời Composition, giải quyết triệt để Stale Lambdas.  
**Key topics:**
- Bản chất Side-Effect trong Declarative UI và hiểm họa khi chạy trực tiếp trong Composable
- `LaunchedEffect(key)` — Coroutine gắn với Composition lifecycle
- `DisposableEffect(key)` — Bắt buộc cleanup với `onDispose { }`
- `SideEffect { }` — Chạy sau mỗi lần vẽ frame thành công (Analytics/Logging)
- `rememberUpdatedState` — Bắt giữ callback mới nhất mà không restart Effect (Fix Stale Lambda)
- Triển khai OTP Countdown Timer & Network Monitor và Compose UI Test

---

## Module 4 — Jetpack Compose Advanced

### Bài 14 — Compose Navigation ✅
**File:** `module-4-compose-advanced/14-navigation.md`  
**Mục tiêu:** Làm chủ Navigation 2.8+ Type-Safe với Kotlinx Serialization, Multiple Back Stacks, Deep Links.  
**Key topics:**
- Navigation 2.8+ Type-Safe với `@Serializable data class/object` routes
- Cơ chế Route Serialization & Bundle mapping ngầm định
- Quản lý Multiple Back Stacks (`saveState = true`, `restoreState = true`) cho BottomNavigation
- Trích xuất tham số an toàn trong ViewModel với `savedStateHandle.toRoute<T>()`
- Tích hợp Deep Links mở từ Web URL và Navigation UI Testing

---

### Bài 15 — Compose Animation ✅
**File:** `module-4-compose-advanced/15-animation.md`  
**Mục tiêu:** Animation vật lý tự nhiên, mượt mà 120Hz, tối ưu hóa Draw phase bằng graphicsLayer.  
**Key topics:**
- High-level APIs (`animate*AsState`, `AnimatedVisibility`, `AnimatedContent`)
- Low-level APIs (`updateTransition`, `rememberInfiniteTransition`, `Animatable`)
- Spring Physics: Cơ học lò xo (`stiffness`, `dampingRatio`) và bảo tồn vận tốc khi bị gián đoạn
- Tối ưu hóa GPU: Bỏ qua Recomposition & Layout phase với `Modifier.graphicsLayer`
- Triển khai Interactive Music Player Card và Compose UI Animation Test

---

### Bài 16 — Performance & Lazy Layouts ✅
**File:** `module-4-compose-advanced/16-performance-lazy-layouts.md`  
**Mục tiêu:** Đạt chuẩn hiệu năng 120fps mượt mà, tối ưu hóa Subcomposition và tạo Baseline Profiles.  
**Key topics:**
- Cơ chế Subcomposition & Item Prefetching trong Lazy Layouts
- Tái sử dụng Slot Table với `contentType` và `key = { it.id }`
- Tránh gián đoạn và tối ưu hóa cuộn với `derivedStateOf`
- Bản chất AOT của Baseline Profiles (loại bỏ độ trễ JIT, tăng tốc 40% khởi động)
- Triển khai High-Performance Social Feed và cấu hình Macrobenchmark

---

### Bài 17 — Compose Testing ✅
**File:** `module-4-compose-advanced/17-compose-testing.md`  
**Mục tiêu:** Viết kiểm thử tự động UI đáng tin cậy, làm chủ Cây ngữ nghĩa và Screenshot Testing.  
**Key topics:**
- Kiến trúc Semantics Tree (Cây ngữ nghĩa) & Merged vs Unmerged node tree
- Bốn trụ cột: Finders (`onNode`), Matchers, Actions (`performClick`), Assertions
- Tự động đồng bộ hóa (Auto-synchronization) & Điều khiển Virtual Clock
- Phân biệt `assertIsDisplayed()` vs `assertExists()`
- Triển khai Cart & Checkout UI Test và Screenshot Testing với Roborazzi trên JVM

---

## Module 5 — Advanced Patterns & Integration
*Các pattern nâng cao cấp Senior / Staff Architect: MVI State Machine, Hilt DI, Background Work bền vững & Reactive Database.*

### Bài 18 — MVI Architecture với Compose ✅
**File:** `module-5-advanced-patterns/18-mvi-architecture.md`  
**Mục tiêu:** Nắm vững Model-View-Intent (MVI), Finite State Machine (FSM), Pure Reducer `(State, Intent) -> State`, Sequential Intent Queue chống race conditions.  
**Key topics:**
- Bản chất toán học của Redux/MVI: $S_{n+1} = \text{reduce}(S_n, I)$
- Xử lý Intent tuần tự qua `Channel<Intent>(Channel.UNLIMITED)` loại bỏ Intent Storm
- Tách biệt State Transition vs Async Side-Effects
- Quản lý One-time Effects qua `Channel<EFFECT>.receiveAsFlow()`
- Xây dựng Shopping Cart & Voucher MVI Feature và Pure Reducer Unit Test (100% JVM)

---

### Bài 19 — Hilt + Flow + Compose: DI End-to-End ✅
**File:** `module-5-advanced-patterns/19-hilt-flow-compose.md`  
**Mục tiêu:** Dependency Injection cấp doanh nghiệp với Dagger Hilt, Component Hierarchy, bytecode tối ưu với `@Binds`, Custom Qualifiers và Multi-Module DI.  
**Key topics:**
- Phân cấp Hilt Component Hierarchy: `SingletonComponent` $\to$ `ActivityRetainedComponent` $\to$ `ViewModelComponent`
- Bytecode internals: `@Binds` (0 wrapper class, zero overhead) vs `@Provides`
- Quản lý CoroutineDispatchers với Custom Qualifiers (`@IoDispatcher`, `@DefaultDispatcher`)
- Hilt + Navigation Compose với `hiltViewModel()` scoped theo BackStack
- Hilt Integration Testing với `HiltAndroidRule`, `@TestInstallIn`, và Fake Module

---

### Bài 20 — WorkManager + Flow: Reliable Background Work ✅
**File:** `module-5-advanced-patterns/20-workmanager-flow.md`  
**Mục tiêu:** Tác vụ nền bền vững (Guaranteed Background Work) sống sót qua Reboot, quan sát tiến độ thời gian thực qua Flow và Expedited Work.  
**Key topics:**
- SQLite Persistence Layer nội bộ (`workdb`) bảo toàn công việc qua app kill và reboot
- Quan sát tiến độ thời gian thực dạng reactive stream qua `getWorkInfoByIdFlow()`
- Giới hạn hệ điều hành Android 14+, Doze Mode và cơ chế Expedited Work
- Assisted Injection với `@HiltWorker` và `@AssistedInject`
- Triển khai Background Image Processor và Automated Testing với `TestListenableWorkerBuilder`

---

### Bài 21 — Room + Flow: Reactive Database, WAL & Safe Migrations ✅
**File:** `module-5-advanced-patterns/21-room-flow.md`  
**Mục tiêu:** Cơ sở dữ liệu SQLite nâng cao với Room, Write-Ahead Logging (WAL), InvalidationTracker, quan hệ 1-N nguyên tử với `@Transaction` và Migration an toàn.  
**Key topics:**
- SQLite Write-Ahead Logging (WAL) mode: Đọc và ghi đồng thời không khóa database
- Cơ chế `InvalidationTracker`: Triggers, modification tracker table và Flow re-query cycle
- `@Transaction` bắt buộc cho quan hệ `@Relation` & `@Embedded` chống Phantom Reads
- Chiến lược Migration thủ công (Manual Migration v1 $\to$ v2) bảo toàn toàn vẹn dữ liệu
- Automated Database & Migration Testing với `MigrationTestHelper` và `inMemoryDatabaseBuilder`

---

## Prerequisites

```
Kotlin cơ bản → OOP/FP concepts → Coroutines basics (Bài 01) → Bắt đầu từ Bài 02
```

## Ký hiệu trạng thái

| Ký hiệu | Nghĩa |
|---------|-------|
| ✅ | Hoàn chỉnh đầy đủ 6 phần |
| 🔄 | Đang viết |
| 🔲 | Chưa viết |
| 🔁 | Cần cập nhật |
