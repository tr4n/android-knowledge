# Bài 04 — Flow trong Android Lifecycle: repeatOnLifecycle & collectAsStateWithLifecycle

> **Module:** 1 — Kotlin Coroutines & Flow Foundation  
> **Prerequisite:** [Bài 03 — Flow Operators: map, flatMapLatest, combine, zip, debounce](03-flow-operators.md)  
> **Official Docs:**
> - [A safer way to collect flows from Android UIs](https://medium.com/androiddevelopers/a-safer-way-to-collect-flows-from-android-uis-230ca3f1f1af)
> - [Lifecycle-aware coroutine APIs](https://developer.android.com/topic/libraries/architecture/coroutines#lifecycle-aware)
> - [Consuming flows safely in Jetpack Compose](https://developer.android.com/kotlin/flow/compose)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

Trong hệ điều hành Android, vòng đời của các thành phần UI (Activity, Fragment, Composable) liên tục thay đổi dựa trên hành vi của người dùng (chuyển đổi ứng dụng, màn hình tắt, xoay thiết bị, chia đôi màn hình). 

**Lifecycle-Aware Flow Collection** là cơ chế thu thập dữ liệu từ Flow có nhận thức đầy đủ về trạng thái vòng đời của UI Host, tự động kích hoạt hoặc hủy bỏ việc thu thập dữ liệu nhằm bảo vệ tài nguyên phần cứng (CPU, Pin, Network, Camera, GPS).

```
┌────────────────────────────────────────────────────────────────────────┐
│                        ANDROID LIFECYCLE STATES                        │
│                                                                        │
│   [INITIALIZED] ──► [CREATED] ──► [STARTED] ──► [RESUMED]              │
│                                       ▲            │                   │
│                                       │ ON_START   │ ON_RESUME         │
│                                       │            ▼                   │
│                                       │         (Active UI)            │
│                                       │            │                   │
│                                       │ ON_STOP    │ ON_PAUSE          │
│                                       ▼            ▼                   │
│                        [DESTROYED] ◄── [STOPPED] ◄─────────────────────┘
└────────────────────────────────────────────────────────────────────────┘
```

### 1.1 Các API tiêu chuẩn từ Google Architecture Components

#### `Lifecycle.repeatOnLifecycle(state: Lifecycle.State)`
- Hàm `suspend` thuộc thư viện `lifecycle-runtime-ktx`.
- Tự động **khởi chạy** một coroutine mới để thu thập Flow khi Lifecycle đạt trạng thái tối thiểu được chỉ định (`targetState`, thông thường là `Lifecycle.State.STARTED`), và **hủy bỏ hoàn toàn** coroutine đó khi Lifecycle tụt xuống dưới trạng thái này (ví dụ khi rơi vào `STOPPED`).

#### `Flow.flowWithLifecycle(lifecycle, minActiveState)`
- Toán tử Flow đóng gói `repeatOnLifecycle` dưới dạng một Flow Operator. Trả về một Flow mới chỉ emit giá trị khi lifecycle ở trạng thái mong muốn.

#### `collectAsStateWithLifecycle()`
- API chuyên biệt trong thư viện `lifecycle-runtime-compose` dành riêng cho Jetpack Compose.
- Thu thập Flow dưới dạng Compose `State<T>`, tự động lắng nghe theo Android Lifecycle của màn hình chứa Composable đó.

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Tại sao `collectAsState()` thông thường lại NGUY HIỂM trong Android?

Nhiều lập trình viên lầm tưởng rằng `collectAsState()` trong Jetpack Compose đã an toàn vì nó gắn với vòng đời của Composable. Đây là một sai lầm phổ biến:

```
VÒNG ĐỜI COMPOSE (Composition)  VS  VÒNG ĐỜI ANDROID HOST (Activity/Fragment)
┌──────────────────────────────┐    ┌────────────────────────────────────────┐
│ Enter Composition            │    │ ON_CREATE ──► ON_START ──► ON_RESUME   │
│   │                          │    │                                        │
│   ▼                          │    │ (Người dùng bấm nút Home ra màn hình)  │
│ Recomposition khi state đổi  │    │                    │                   │
│   │                          │    │                    ▼                   │
│   ▼                          │    │            ON_STOP (Background)        │
│ Leave Composition            │    │    Composable KHÔNG LEAVE COMPOSITION! │
└──────────────────────────────┘    └────────────────────────────────────────┘
```

#### Phân tích chi tiết:
1. Khi người dùng bấm nút **Home** hoặc chuyển sang ứng dụng khác, Activity rơi vào trạng thái `ON_STOP`.
2. Tuy nhiên, Composable **chưa hề rời khỏi Composition** (nó vẫn nằm trong bộ nhớ cache để sẵn sàng hiển thị lại ngay lập tức khi user quay lại).
3. Do đó, nếu sử dụng `collectAsState()`, coroutine thu thập Flow **vẫn tiếp tục chạy ngầm 100%**!
4. Nếu Flow này kết nối với GPS, Camera hoặc WebSocket, ứng dụng của bạn sẽ tiếp tục tiêu hao dữ liệu mạng và làm cạn kiệt pin người dùng trong nền mà không ai hay biết.

---

### 2.2 Cơ chế bên trong của `collectAsStateWithLifecycle()`

```
collectAsStateWithLifecycle()
            │
            ▼
┌────────────────────────────────────────────────────────┐
│ Lấy LocalLifecycleOwner.current                        │
│ Gọi repeatOnLifecycle(Lifecycle.State.STARTED)         │
│           │                                            │
│           ├─── Lifecycle ON_START:                     │
│           │      Spawn Coroutine mới ──► flow.collect()│
│           │                                            │
│           └─── Lifecycle ON_STOP:                      │
│                  Cancel Coroutine ──► Hủy collection   │
│                  Giải phóng upstream producer          │
└────────────────────────────────────────────────────────┘
```

`collectAsStateWithLifecycle` đóng vai trò là cây cầu kết nối giữa hai hệ thống vòng đời: **Android Platform Lifecycle** và **Jetpack Compose State Snapshot System**.

---

### 2.3 Cơ chế `SharingStarted.WhileSubscribed(stopTimeoutMillis = 5000)`

Trong ViewModel, khi chuyển đổi một Cold Flow (từ Room, Network, Sensors) sang Hot StateFlow thông qua toán tử `stateIn`, chiến lược quản lý subscriber `WhileSubscribed` là chuẩn mực vàng.

```
                  KỊCH BẢN: NGƯỜI DÙNG XOAY MÀN HÌNH (CONFIGURATION CHANGE)
                  
Activity A (Dọc)                                                   Activity A (Ngang)
      │                                                                    │
      ▼                                                                    ▼
[onDestroy() gọi]                                                    [onCreate() gọi]
Số subscriber = 0                                                    Số subscriber = 1
      │                                                                    │
      ├─────────────────────── Window 5000ms ──────────────────────────────┤
      │                                                                    │
      ▼                                                                    ▼
Downstream hủy subscribe                                         Downstream subscribe lại
      │                                                                    │
      └─────────► UPSTREAM FLOW KHÔNG BỊ HỦY (VẪN GIỮ KẾT NỐI)! ───────────┘
```

#### Tại sao lại là con số 5000ms (5 giây)?
1. **Khi xoay màn hình:** Activity cũ bị Destroy và Activity mới được Recreate. Quá trình này thường diễn ra trong khoảng vài trăm mili-giây.
2. Trong tích tắc đó, số lượng subscriber của StateFlow rơi về `0`.
3. Nếu không có bộ đếm hoãn hủy (`stopTimeoutMillis = 0`), Upstream Flow (ví dụ Socket hoặc truy vấn DB lớn) sẽ bị **hủy ngay lập tức**, sau đó 200ms lại bị **kết nối lại từ đầu**. Việc ngắt kết nối và tái kết nối liên tục gây giật lag giao diện, chớp màn hình và lãng phí request mạng.
4. Với `stopTimeoutMillis = 5000`, hệ điều hành Android có đủ thời gian hoàn tất việc xoay màn hình mà luồng upstream vẫn được duy trì liên tục!
5. Nếu sau 5 giây ứng dụng thực sự ở trong nền (người dùng đã bấm Home thật sự), upstream flow mới chính thức bị đóng lại để tiết kiệm pin.

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Sự tiến hóa trong việc lắng nghe dữ liệu ở UI Android

```
Kỷ nguyên 1: Callbacks & BroadcastReceivers
- Rò rỉ Activity Memory Leak triền miên nếu quên unregister trong onDestroy.

Kỷ nguyên 2: LiveData
- Lifecycle-aware tự động, rất tốt với XML Views.
- Nhược điểm: Chỉ chạy trên Main thread, thiếu các toán tử bất đồng bộ mạnh mẽ (flatMap, debounce), không tương thích tốt với Kotlin Multiplatform.

Kỷ nguyên 3: lifecycleScope.launchWhenStarted / launchWhenResumed
- THẢM HỌA TIÊU HAO TÀI NGUYÊN: launchWhenX chỉ TẠM DỪNG (SUSPEND) coroutine thu thập, nhưng KHÔNG HỦY coroutine của upstream flow! Upstream vẫn liên tục emit dữ liệu vào khoảng không, gây cạn kiệt pin và tràn buffer.
- Đã bị Google chính thức DEPRECATED hoàn toàn.

Kỷ nguyên 4 (Hiện đại): repeatOnLifecycle & collectAsStateWithLifecycle
- Giải pháp triệt để: HỦY TOÀN BỘ coroutine khi vào background, và TÁI KHỞI ĐỘNG an toàn khi quay lại màn hình.
```

---

### 3.2 Bảng so sánh trực diện các phương pháp Collect Flow trong Compose

| Tiêu chí | `collectAsState()` | `collectAsStateWithLifecycle()` |
|---|---|---|
| **Thư viện** | `androidx.compose.runtime` | `androidx.lifecycle:lifecycle-runtime-compose` |
| **Nhận thức Lifecycle Android?** | ❌ KHÔNG (Chỉ biết Compose) | ✅ CÓ (Tích hợp với Activity/Fragment Lifecycle) |
| **Dừng khi app vào background?** | ❌ Không dừng (Vẫn chạy ngầm) | ✅ Dừng hoàn toàn ở `ON_STOP` |
| **Tác động đến Pin & Tài nguyên**| Rất nguy hiểm nếu upstream là stream liên tục | Tối ưu hóa tuyệt đối theo khuyến nghị của Google |
| **Mức độ khuyến nghị của Google** | Chỉ dùng cho các stream thuần bộ nhớ trong Composition | **Bắt buộc dùng cho mọi Flow từ ViewModel** |

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Ma trận chọn cấu hình `SharingStarted` trong `stateIn` / `shareIn`

| SharingStarted Strategy | Hành vi khi không còn Collector nào | Use Cases phù hợp |
|---|---|---|
| **`WhileSubscribed(5000)`** | Chờ 5s sau khi collector cuối cùng rời đi rồi mới cancel upstream flow. | **99% UI Screens** trong Android (bảo vệ lifecycle và an toàn khi xoay màn hình). |
| **`Lazily`** | Bắt đầu chạy khi có collector đầu tiên, và **tiếp tục chạy vĩnh viễn** dù không còn collector nào. | Luồng caching dữ liệu toàn cục trong App Scope, cấu hình ứng dụng dùng chung. |
| **`Eagerly`** | Bắt đầu chạy ngay lập tức khi khởi tạo, bất kể có collector hay không. | Tác vụ khởi tạo sớm ở tầng Application hoặc Service nền độc lập. |

---

### 4.2 Lựa chọn `minActiveState` phù hợp

```kotlin
// 1. Lifecycle.State.STARTED (Mặc định & Khuyên dùng)
// Phù hợp cho hầu hết mọi tác vụ hiển thị UI:
viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
    viewModel.uiState.collect { updateViews(it) }
}

// 2. Lifecycle.State.RESUMED
// Chỉ dùng khi ứng dụng bắt buộc phải ở tiền cảnh tương tác trực tiếp
// Ví dụ: Nhận preview frame từ Camera2 API, quét mã QR, theo dõi chuyển động con quay
viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.RESUMED) {
    cameraStream.collect { renderFrame(it) }
}
```

---

### 4.3 Các Cạm bẫy phổ biến (Pitfalls & Anti-patterns)

#### Cạm bẫy 1: Dùng `lifecycleScope.launch` trần trụi để collect Flow
```kotlin
// ANTI-PATTERN: Rò rỉ tài nguyên nặng nề
class UserFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        // SAI LẦM: Coroutine này sẽ sống mãi từ onViewCreated cho tới khi Fragment bị DESTROY hoàn toàn!
        // Khi user chuyển tab, Fragment vào background nhưng flow vẫn bị collect ngầm!
        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.locationFlow.collect { updateMap(it) }
        }
    }
}

// GIẢI PHÁP ĐÚNG:
class UserFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.locationFlow.collect { updateMap(it) }
            }
        }
    }
}
```

#### Cạm bẫy 2: Dùng `lifecycleScope` thay vì `viewLifecycleOwner.lifecycleScope` trong Fragment
Trong Fragment, vòng đời của **View** ngắn hơn vòng đời của **Fragment Instance** (khi đưa vào BackStack, View bị destroy nhưng Fragment object vẫn còn). Nếu dùng `lifecycleScope` thay vì `viewLifecycleOwner.lifecycleScope`, bạn sẽ bị leak view hierarchy và có thể gặp lỗi crash `NullPointerException`.

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Xây dựng ứng dụng **Giám sát vị trí GPS Real-time (Live Location Tracker)**:
- Tự động bật GPS khi màn hình sáng.
- Tự động ngắt GPS khi người dùng bấm Home hoặc tắt màn hình để bảo vệ pin.
- Xoay màn hình không làm gián đoạn tín hiệu GPS nhờ `WhileSubscribed(5000)`.

### Bước 1: Khai báo Dependencies

```kotlin
// build.gradle.kts (Module: app)
dependencies {
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.4")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.8.4")
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.4")
    
    implementation(platform("androidx.compose:compose-bom:2024.06.00"))
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui")
}
```

---

### Bước 2: Data Layer (Cold Flow với `callbackFlow`)

```kotlin
data class UserLocation(val latitude: Double, val longitude: Double, val timestamp: Long)

interface LocationTracker {
    fun getLocationUpdates(): Flow<UserLocation>
}

class FakeLocationTracker(
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) : LocationTracker {

    override fun getLocationUpdates(): Flow<UserLocation> = callbackFlow {
        println("🛰️ GPS Hardware: KÍCH HOẠT VỆ TINH (Bắt đầu tốn pin)")

        // Giả lập callback nhận tọa độ GPS liên tục
        val timerJob = launch {
            var lat = 21.0285
            var lng = 105.8542
            while (isActive) {
                delay(1000) // Phát tọa độ mỗi 1 giây
                lat += 0.0001
                lng += 0.0001
                trySend(UserLocation(lat, lng, System.currentTimeMillis()))
            }
        }

        // Khối awaitClose: CHẠY KHI FLOW BỊ CANCEL
        awaitClose {
            println("🛑 GPS Hardware: TẮT VỆ TINH HOÀN TOÀN (Bảo vệ pin)")
            timerJob.cancel()
        }
    }.flowOn(ioDispatcher)
}
```

---

### Bước 3: ViewModel Layer (`stateIn` + `WhileSubscribed(5000)`)

```kotlin
sealed interface LocationUiState {
    data object Connecting : LocationUiState
    data class Active(val location: UserLocation) : LocationUiState
    data class Error(val message: String) : LocationUiState
}

class LocationViewModel(
    locationTracker: LocationTracker
) : ViewModel() {

    // Chuyển Cold Flow thành Hot StateFlow an toàn vòng đời tuyệt đối
    val locationState: StateFlow<LocationUiState> = locationTracker.getLocationUpdates()
        .map<UserLocation, LocationUiState> { LocationUiState.Active(it) }
        .catch { emit(LocationUiState.Error("Mất tín hiệu GPS")) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(stopTimeoutMillis = 5000),
            initialValue = LocationUiState.Connecting
        )
}
```

---

### Bước 4: UI Layer (Jetpack Compose với `collectAsStateWithLifecycle`)

```kotlin
@Composable
fun LocationTrackingScreen(
    viewModel: LocationViewModel,
    modifier: Modifier = Modifier
) {
    // CHUẨN MỰC GOOGLE: Tự động stop collect khi Activity/Screen rơi vào onStop!
    val state by viewModel.locationState.collectAsStateWithLifecycle()

    Scaffold(
        topBar = {
            TopAppBar(title = { Text("Live GPS Tracker") })
        }
    ) { innerPadding ->
        Box(
            modifier = modifier
                .fillMaxSize()
                .padding(innerPadding),
            contentAlignment = Alignment.Center
        ) {
            when (val currentState = state) {
                is LocationUiState.Connecting -> {
                    Column(horizontalAlignment = Alignment.CenterHorizontally) {
                        CircularProgressIndicator()
                        Spacer(modifier = Modifier.height(8.dp))
                        Text("Đang kết nối vệ tinh GPS...")
                    }
                }
                is LocationUiState.Error -> {
                    Text(text = currentState.message, color = MaterialTheme.colorScheme.error)
                }
                is LocationUiState.Active -> {
                    Card(
                        modifier = Modifier
                            .fillMaxWidth()
                            .padding(24.dp),
                        colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.primaryContainer)
                    ) {
                        Column(modifier = Modifier.padding(20.dp)) {
                            Text(text = "Tọa độ trực tiếp:", style = MaterialTheme.typography.titleMedium)
                            Spacer(modifier = Modifier.height(8.dp))
                            Text(text = "Vĩ độ (Lat): ${currentState.location.latitude}")
                            Text(text = "Kinh độ (Lng): ${currentState.location.longitude}")
                            Text(
                                text = "Cập nhật lúc: ${currentState.location.timestamp}",
                                style = MaterialTheme.typography.bodySmall
                            )
                        }
                    }
                }
            }
        }
    }
}
```

---

### Bước 5: Kiểm thử tự động Compose UI Test & Lifecycle Simulation

```kotlin
@RunWith(AndroidJUnit4::class)
class LocationTrackingScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun screen_displaysLocation_whenActiveStateEmitted() {
        val fakeFlow = MutableStateFlow<LocationUiState>(
            LocationUiState.Active(UserLocation(10.762622, 106.660172, 123456L))
        )
        
        composeTestRule.setContent {
            // Giả lập Compose UI
            val state by fakeFlow.collectAsStateWithLifecycle()
            // Render giao diện với state
            Text(text = if (state is LocationUiState.Active) "Vĩ độ: 10.762622" else "Loading")
        }

        // Xác nhận hiển thị đúng tọa độ
        composeTestRule.onNodeWithText("Vĩ độ: 10.762622").assertIsDisplayed()
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior/Lead Android

#### Q1: Tại sao `SharingStarted.WhileSubscribed(5000)` không áp dụng số 0ms mà lại là 5000ms?
**Trả lời chuẩn bản chất:**
Nếu đặt `stopTimeoutMillis = 0ms`, khi thiết bị xoay màn hình (Configuration Change), Activity cũ bị huỷ và Activity mới được tái tạo. Trong khoảng thời gian vài mili-giây ngắn ngủi này, số lượng subscriber của StateFlow rơi về 0. Điều này làm cho StateFlow lập tức dừng luồng dữ liệu upstream (ví dụ ngắt kết nối WebSocket hoặc ngừng truy vấn DB), sau đó vài mili-giây lại phải kết nối lại từ đầu khi Activity mới subscribe. Con số 5000ms là khoảng "thời gian hoãn" an toàn để chờ xem liệu người dùng có thực sự thoát khỏi màn hình hay chỉ đang xoay điện thoại.

#### Q2: Có thể dùng `flowWithLifecycle` thay thế `repeatOnLifecycle` không? Điểm khác biệt là gì?
**Trả lời chuẩn bản chất:**
`flowWithLifecycle` thực chất chỉ là một wrapper bọc ngoài `repeatOnLifecycle` nhằm hỗ trợ phong cách viết dạng toán tử chuỗi (operator chaining):
```kotlin
// Hai cách viết sau cho kết quả hoàn toàn tương đương:
// Cách 1: repeatOnLifecycle
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        flow.collect { ... }
    }
}

// Cách 2: flowWithLifecycle
viewLifecycleOwner.lifecycleScope.launch {
    flow.flowWithLifecycle(viewLifecycleOwner.lifecycle, Lifecycle.State.STARTED)
        .collect { ... }
}
```
Tuy nhiên, nếu bạn cần thu thập **nhiều Flow cùng một lúc** trên cùng một màn hình, dùng `repeatOnLifecycle` sẽ tối ưu hơn vì bạn chỉ cần khởi tạo 1 coroutine duy nhất cho tất cả các flows bên trong một block!

#### Q3: Điều gì xảy ra nếu Activity chuyển sang chế độ Multi-Window (Chia đôi màn hình)?
**Trả lời chuẩn bản chất:**
Trong chế độ Multi-window, một Activity có thể ở trạng thái `STARTED` nhưng không ở trạng thái `RESUMED` nếu người dùng đang tương tác với nửa màn hình còn lại.
- Nếu bạn collect với `minActiveState = STARTED`, giao diện vẫn tiếp tục cập nhật dữ liệu bình thường (đây là hành vi người dùng mong đợi khi họ đang nhìn vào màn hình của bạn).
- Nếu bạn collect với `minActiveState = RESUMED`, ứng dụng của bạn sẽ bị đóng băng (ngừng nhận dữ liệu) ngay khi người dùng chạm vào ứng dụng bên cạnh! Vì vậy, `STARTED` luôn là lựa chọn chuẩn mực nhất.

---

### 6.2 Lỗi Runtime thường gặp & Hướng xử lý

| Tình huống lỗi | Nguyên nhân gốc rễ | Giải pháp xử lý triệt để |
|---|---|---|
| **Pin tụt nhanh chóng mặt khi app ở background** | Dùng `collectAsState()` thay vì `collectAsStateWithLifecycle()` cho các stream GPS/Sensor/Socket. | Thay thế toàn bộ bằng `collectAsStateWithLifecycle()`. |
| **Crash: `IllegalStateException: Can't access the Fragment View's LifecycleOwner when getView() is null`** | Lắng nghe Flow thông qua `viewLifecycleOwner` trước khi `onCreateView()` hoặc sau khi `onDestroyView()`. | Chỉ khởi chạy việc collect Flow từ bên trong hàm `onViewCreated()`. |
| **Màn hình nhấp nháy, dữ liệu reload lại từ đầu mỗi khi xoay máy** | Dùng `SharingStarted.WhileSubscribed(0)` hoặc không cache state bằng `stateIn`. | Chuyển sang `SharingStarted.WhileSubscribed(5000)` trong ViewModel. |

---

*Bài trước: [03 — Flow Operators: map, flatMapLatest, combine, zip, debounce](03-flow-operators.md)*  
*Bài tiếp theo: [05 — Flow Exception Handling: catch, onCompletion, SupervisorJob](05-flow-exception-handling.md)*
