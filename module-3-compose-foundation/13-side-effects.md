# Bài 13 — Side Effects trong Compose: LaunchedEffect, DisposableEffect & SideEffect

> **Module:** 3 — Jetpack Compose Foundation  
> **Prerequisite:** [Bài 12 — Recomposition & Stability: @Stable, @Immutable](12-recomposition-stability.md)  
> **Official Docs:**
> - [Side-effects in Compose — Android Developers](https://developer.android.com/develop/ui/compose/side-effects)
> - [Effect Handlers API Reference](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

Trong kiến trúc khai báo của Jetpack Compose, một Composable function lý tưởng phải là **Idempotent (Xác định) và Pure (Không có tác dụng phụ)**: Nó chỉ nhận dữ liệu vào và phát ra các UI node.

Tuy nhiên, trong một ứng dụng thực tế, bạn bắt buộc phải tương tác với thế giới bên ngoài (gọi API mạng, hiển thị SnackBar, đăng ký lắng nghe GPS, ghi log analytics, chuyển màn hình). 

**Side-Effect (Tác dụng phụ)** trong Jetpack Compose là **bất kỳ thay đổi trạng thái nào xảy ra bên ngoài phạm vi của hàm `@Composable`**, hoặc bất kỳ thao tác nào phá vỡ tính thuần túy của hàm vẽ giao diện.

```
┌────────────────────────────────────────────────────────────────────────┐
│                      COMPOSE EFFECT HANDLERS SUITE                     │
│                                                                        │
│   1. LaunchedEffect(key1, key2...)                                     │
│      - Chạy một suspend function (Coroutine) gắn với Composition       │
│      - Tự động hủy và khởi chạy lại khi key thay đổi                   │
│                                                                        │
│   2. DisposableEffect(key1, key2...)                                   │
│      - Dành cho các tác vụ cần DỌN DẸP tài nguyên (Cleanup)            │
│      - Bắt buộc phải có khối onDispose { }                             │
│                                                                        │
│   3. SideEffect { }                                                    │
│      - Chạy đồng bộ SAU MỖI LẦN Recomposition thành công               │
│      - Đồng bộ state của Compose ra các đối tượng non-Compose          │
│                                                                        │
│   4. rememberUpdatedState(value)                                       │
│      - Bắt giữ giá trị mới nhất của tham số mà không restart effect    │
│                                                                        │
│   5. rememberCoroutineScope()                                          │
│      - Mở CoroutineScope gắn với Composition cho các Event Callbacks   │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Vòng đời của `LaunchedEffect` và cơ chế hủy bỏ (Cancellation)

Làm thế nào Compose quản lý vòng đời của Coroutine bên trong `LaunchedEffect`?

```
                 VÒNG ĐỜI COROUTINE TRONG LAUNCHEDEFFECT(KEY)

1. Node Enter Composition:
   LaunchedEffect(key) bắt đầu thực thi
          │
          ▼
   Tạo Coroutine mới trong ApplyCoroutineScope ──► Chạy suspend block
          │
2. Recomposition xảy ra:
   Giá trị 'key' có bị thay đổi so với frame trước không?
          │
          ├─────────────────────────┬─────────────────────────┐
          ▼                         ▼                         ▼
      [ KEY KHÔNG ĐỔI ]         [ KEY THAY ĐỔI ]      [ LEAVE COMPOSITION ]
          │                         │                         │
   Tiếp tục chạy bình thường,       ▼                         ▼
   KHÔNG khởi động lại!         HỦY COROUTINE CŨ          HỦY COROUTINE
                                (Cancel Job)              (Cancel Job)
                                    │                         │
                                    ▼                         ▼
                                SPAWN COROUTINE MỚI!      GIẢI PHÓNG HOÀN TOÀN!
```

- `LaunchedEffect` sử dụng chính cơ chế `key` để quyết định xem có nên khởi động lại (restart) coroutine hay không.
- Nếu bạn truyền `LaunchedEffect(Unit)` hoặc `LaunchedEffect(true)`, coroutine sẽ **chỉ chạy đúng 1 lần duy nhất** khi Composable xuất hiện trên màn hình và chỉ bị hủy khi Composable rời khỏi giao diện.

---

### 2.2 Cơ chế Cleanup của `DisposableEffect`: Khối `onDispose { }`

Nếu bạn cần đăng ký một BroadcastReceiver, một Sensor Listener, hoặc một WebSocket connection, bạn **không được phép dùng `LaunchedEffect`** vì nó không cung cấp hook dọn dẹp đồng bộ. Bạn bắt buộc phải dùng `DisposableEffect`:

```
              CƠ CHẾ HOẠT ĐỘNG CỦA DISPOSABLEEFFECT(KEY)

Enter Composition ──► Khởi chạy khối DisposableEffect block
                            │
                            ├── Đăng ký SensorEventListener
                            │
                            └── Cung cấp khối onDispose { }
                                      │
               ┌──────────────────────┴──────────────────────┐
               ▼                                             ▼
       [ KEY THAY ĐỔI ]                              [ LEAVE COMPOSITION ]
               │                                             │
               ▼                                             ▼
       Gọi onDispose { }                             Gọi onDispose { }
       (Hủy đăng ký listener cũ)                     (Hủy đăng ký vĩnh viễn)
               │
               ▼
       Chạy lại block mới!
```

> **Hợp đồng bất biến (Contract):** Trình biên dịch Compose bắt buộc câu lệnh cuối cùng bên trong lambda của `DisposableEffect` phải là lời gọi hàm `onDispose { ... }`. Nếu thiếu, code sẽ không thể biên dịch!

---

### 2.3 Giải quyết vấn nạn Stale Lambda với `rememberUpdatedState`

Đây là một trong những cơ chế tinh vi nhất của Compose. Hãy xem kịch bản lỗi kinh điển sau:

```kotlin
// KỊCH BẢN LỖI: Closure giữ tham chiếu cũ (Stale Closure)
@Composable
fun TimerComponent(onTimeout: () -> Unit) {
    // Key = Unit: Chỉ chạy 1 lần khi enter composition
    LaunchedEffect(Unit) {
        delay(10000) // Đợi 10 giây
        onTimeout()  // NGUY HIỂM: onTimeout ở đây bị ghim chặt ở giá trị ban đầu!
    }
}
```
Nếu trong 10 giây đó, component cha Recompose và truyền vào một lambda `onTimeout` mới với các biến tham chiếu mới, lambda cũ bên trong `LaunchedEffect` vẫn sẽ gọi phiên bản cũ (Stale Lambda Bug)!

#### Cơ chế của `rememberUpdatedState`:
```kotlin
@Composable
fun TimerComponent(onTimeout: () -> Unit) {
    // Tạo ra một State wrapper luôn cập nhật giá trị mới nhất ở MỌI frame:
    val currentOnTimeout by rememberUpdatedState(onTimeout)

    LaunchedEffect(Unit) {
        delay(10000)
        // Khi gọi, nó đọc .value từ State wrapper -> Luôn lấy lambda mới nhất
        // MÀ KHÔNG CẦN PHẢI RESTART COROUTINE!
        currentOnTimeout() 
    }
}
```

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Thảm họa khi chạy Side Effect trực tiếp trong thân Composable

Nhiều lập trình viên mới chuyển từ XML sang Compose thường mắc sai lầm:

```kotlin
// SAI LẦM CHÍ MẠNG (ANTI-PATTERN):
@Composable
fun UserProfileScreen(userId: String, viewModel: UserViewModel) {
    // CẤM: Gọi API trực tiếp trong thân hàm Composable!
    viewModel.fetchUserProfile(userId) 

    val user by viewModel.userState.collectAsState()
    Text(text = user.name)
}
```

#### Hậu quả thảm khốc:
1. Mỗi khi người dùng tương tác làm Recompose (hoặc frame rate 120fps chạy), hàm `UserProfileScreen` sẽ được gọi lại.
2. Lệnh `viewModel.fetchUserProfile()` sẽ bị **bắn liên tục hàng trăm lần mỗi phút** lên server!
3. Server bị quá tải DDoS, pin điện thoại cạn kiệt trong 15 phút, giao diện nhấp nháy điên cuồng.

#### Effect Handlers giải quyết triệt để:
Bọc trong `LaunchedEffect(userId)`: Request chỉ chạy đúng 1 lần. Nếu `userId` không đổi, dù màn hình có recompose 1000 lần thì request cũng không bao giờ bị gọi lại!

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Cây quyết định (Decision Tree): Chọn Effect Handler chuẩn xác

```
BẠN CẦN THỰC THI TÁC VỤ GÌ?
│
├── Cần chạy Suspend Function / Coroutine?
│   ├── Khởi chạy tự động theo vòng đời của màn hình? ──► LaunchedEffect(key)
│   └── Khởi chạy khi người dùng bấm nút (onClick)? ────► rememberCoroutineScope()
│
├── Cần đăng ký Listener / Callback và DỌN DẸP khi thoát? ─► DisposableEffect(key)
│   (BroadcastReceiver, Sensor, Location, WebSocket)
│
├── Cần đồng bộ State Compose ra ngoài sau mỗi frame? ────► SideEffect { }
│   (Ghi log analytics, cập nhật trạng thái Notification bar)
│
└── Truyền callback vào Effect dài hạn mà không restart? ─► rememberUpdatedState(callback)
```

---

### 4.2 Cạm bẫy phổ biến (Common Pitfalls & Anti-patterns)

#### Cạm bẫy 1: Lạm dụng `rememberCoroutineScope` trong thân Composable
`rememberCoroutineScope` **chỉ được dùng trong các Event Callbacks** (như `onClick`, `onSwipe`). Tuyệt đối không gọi `scope.launch { }` trực tiếp trong thân Composable:
```kotlin
// SAI:
@Composable
fun BadScreen() {
    val scope = rememberCoroutineScope()
    scope.launch { doWork() } // CẤM: Chạy lại ở mỗi lần Recompose!
}

// ĐÚNG: Dùng LaunchedEffect
@Composable
fun GoodScreen() {
    LaunchedEffect(Unit) { doWork() }
}
```

#### Cạm bẫy 2: Dùng `key = true` hoặc `key = Unit` khi Effect phụ thuộc vào biến số
Nếu bạn gọi API dựa trên `searchQuery` nhưng lại đặt `LaunchedEffect(Unit) { searchApi(searchQuery) }`, khi `searchQuery` đổi từ "A" sang "B", effect sẽ **không bao giờ chạy lại**. Bắt buộc phải truyền `LaunchedEffect(searchQuery)`.

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Xây dựng tính năng **Bộ đếm ngược OTP & Giám sát kết nối mạng (OTP Countdown & Network Monitor)**:
- Sử dụng `DisposableEffect` để đăng ký và hủy `ConnectivityManager.NetworkCallback`.
- Sử dụng `LaunchedEffect` đếm ngược 60 giây cho mã OTP.
- Áp dụng `rememberUpdatedState` để bảo đảm callback `onTimeout` luôn mới nhất mà không làm reset bộ đếm.
- Sử dụng `SideEffect` để ghi log Analytics sau mỗi lần màn hình Recompose thành công.

### Bước 1: Giám sát mạng với `DisposableEffect`

```kotlin
@Composable
fun NetworkConnectivityMonitor(
    onNetworkStatusChanged: (Boolean) -> Unit
) {
    val context = LocalContext.current
    // Cập nhật callback mới nhất bằng rememberUpdatedState
    val currentCallback by rememberUpdatedState(onNetworkStatusChanged)

    DisposableEffect(context) {
        val connectivityManager = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        
        val networkCallback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                currentCallback(true)
            }
            override fun onLost(network: Network) {
                currentCallback(false)
            }
        }

        val request = NetworkRequest.Builder()
            .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
            .build()

        connectivityManager.registerNetworkCallback(request, networkCallback)

        // BẮT BUỘC: Dọn dẹp listener khi Composable rời khỏi màn hình!
        onDispose {
            connectivityManager.unregisterNetworkCallback(networkCallback)
        }
    }
}
```

---

### Bước 2: Bộ đếm ngược OTP với `LaunchedEffect` & `rememberUpdatedState`

```kotlin
@Composable
fun OtpCountdownTimer(
    totalSeconds: Int = 60,
    onTimeout: () -> Unit,
    modifier: Modifier = Modifier
) {
    var secondsLeft by remember { mutableStateOf(totalSeconds) }
    
    // Đảm bảo lambda onTimeout luôn mới nhất mà KHÔNG kích hoạt lại LaunchedEffect
    val latestOnTimeout by rememberUpdatedState(onTimeout)

    LaunchedEffect(totalSeconds) {
        secondsLeft = totalSeconds
        while (secondsLeft > 0) {
            delay(1000L)
            secondsLeft--
        }
        latestOnTimeout() // Gọi callback khi hết giờ
    }

    Text(
        text = if (secondsLeft > 0) "Mã OTP hết hạn sau: ${secondsLeft}s" else "Mã OTP đã hết hạn!",
        style = MaterialTheme.typography.bodyMedium,
        color = if (secondsLeft > 10) MaterialTheme.colorScheme.onSurface else MaterialTheme.colorScheme.error,
        modifier = modifier
    )
}
```

---

### Bước 3: Màn hình tích hợp toàn diện (`SideEffect` Logging)

```kotlin
@Composable
fun OtpVerificationScreen(
    phoneNumber: String,
    onResendOtp: () -> Unit,
    onTimeoutExpired: () -> Unit,
    modifier: Modifier = Modifier
) {
    var isOnline by remember { mutableStateOf(true) }
    var renderCount by remember { mutableStateOf(0) }

    // 1. DisposableEffect giám sát mạng
    NetworkConnectivityMonitor { online ->
        isOnline = online
    }

    // 2. SideEffect chạy sau mỗi lần vẽ frame thành công
    SideEffect {
        renderCount++
        println("📊 Analytics: OtpVerificationScreen đã render thành công lần thứ: $renderCount")
    }

    Scaffold(
        topBar = { TopAppBar(title = { Text("Xác thực OTP") }) }
    ) { padding ->
        Column(
            modifier = modifier
                .fillMaxSize()
                .padding(padding)
                .padding(24.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center
        ) {
            // Cảnh báo mất mạng
            if (!isOnline) {
                Surface(
                    color = MaterialTheme.colorScheme.errorContainer,
                    shape = RoundedCornerShape(8.dp),
                    modifier = Modifier.fillMaxWidth().padding(bottom = 16.dp)
                ) {
                    Text(
                        text = "⚠️ Mất kết nối internet. Vui lòng kiểm tra lại mạng!",
                        color = MaterialTheme.colorScheme.onErrorContainer,
                        modifier = Modifier.padding(12.dp)
                    )
                }
            }

            Text(
                text = "Mã xác thực đã gửi tới số $phoneNumber",
                style = MaterialTheme.typography.titleMedium
            )

            Spacer(modifier = Modifier.height(16.dp))

            // 3. LaunchedEffect đếm ngược
            OtpCountdownTimer(
                totalSeconds = 60,
                onTimeout = onTimeoutExpired
            )

            Spacer(modifier = Modifier.height(24.dp))

            Button(
                onClick = onResendOtp,
                enabled = isOnline
            ) {
                Text("Gửi lại mã OTP")
            }
        }
    }
}
```

---

### Bước 4: Viết Compose UI Test cho Side Effects

```kotlin
@RunWith(AndroidJUnit4::class)
class OtpCountdownTimerTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun timer_countsDown_andCallsTimeout() {
        var timeoutCalled = false

        composeTestRule.setContent {
            OtpCountdownTimer(
                totalSeconds = 2,
                onTimeout = { timeoutCalled = true }
            )
        }

        // Ban đầu hiển thị 2s
        composeTestRule.onNodeWithText("Mã OTP hết hạn sau: 2s").assertIsDisplayed()

        // Tiến thời gian ảo của Compose
        composeTestRule.mainClock.advanceTimeBy(1050L)
        composeTestRule.onNodeWithText("Mã OTP hết hạn sau: 1s").assertIsDisplayed()

        composeTestRule.mainClock.advanceTimeBy(1050L)
        composeTestRule.onNodeWithText("Mã OTP đã hết hạn!").assertIsDisplayed()

        assertTrue(timeoutCalled)
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior/Staff Android Architect

#### Q1: Khi nào thì một Coroutine bên trong `rememberCoroutineScope` bị cancel?
**Trả lời chuẩn bản chất:**
CoroutineScope được tạo bởi `rememberCoroutineScope()` bị ràng buộc chặt chẽ với **điểm Composition nơi nó được gọi**. Khi Composable chứa scope đó rời khỏi cây giao diện (Leave Composition), Compose Runtime sẽ tự động gọi `cancel()` trên CoroutineScope này, hủy toàn bộ các coroutines con đang chạy bên trong nó.

#### Q2: `produceState` hoạt động thế nào dưới tầng thấp (Under the hood)?
**Trả lời chuẩn bản chất:**
`produceState` thực chất chỉ là một cú pháp đường (syntactic sugar) kết hợp giữa `remember { mutableStateOf(initialValue) }` và `LaunchedEffect(key)`. Nó tạo ra một `MutableState` và khởi chạy một coroutine để lắng nghe các API dựa trên callback hoặc RxJava/Flow bên ngoài, sau đó gán kết quả vào `value`.

#### Q3: Tại sao Side Effect có thể bị chạy 2 lần khi bật chế độ StrictMode hoặc Android Studio Preview?
**Trả lời chuẩn bản chất:**
Trong môi trường phát triển (Debug / Tooling Preview), Compose Runtime có thể áp dụng cơ chế xác minh tính Idempotent bằng cách chạy thử một Composable nhiều lần để phát hiện các tác dụng phụ rò rỉ. Nếu bạn viết code thuần túy tuân thủ hợp đồng của `DisposableEffect` (có cleanup đầy đủ) và `LaunchedEffect`, việc hệ thống kích hoạt rồi dọn dẹp sẽ diễn ra an toàn 100% mà không để lại bất kỳ side-effect nào.

---

### 6.2 Lỗi Runtime thường gặp & Hướng xử lý

| Tình huống lỗi | Nguyên nhân gốc rễ | Cách xử lý triệt để |
|---|---|---|
| **Memory Leak BroadcastReceiver / Sensor** | Dùng `LaunchedEffect` để đăng ký listener mà không có cơ chế hủy khi thoát. | Chuyển sang dùng `DisposableEffect` và bắt buộc hủy đăng ký trong khối `onDispose { }`. |
| **Coroutine bị hủy và khởi động lại liên tục** | Truyền đối tượng Unstable hoặc đối tượng được tạo mới ở mỗi frame vào tham số `key` của `LaunchedEffect`. | Sử dụng các kiểu dữ liệu nguyên thủy hoặc các giá trị Stable làm `key`. |
| **Stale Callback Bug (Gọi logic cũ sau khi Recompose)** | Truyền lambda callback vào `LaunchedEffect(Unit)` dài hạn mà không bọc qua `rememberUpdatedState`. | Bọc callback bằng `val currentCallback by rememberUpdatedState(callback)`. |

---

*Bài trước: [12 — Recomposition & Stability: @Stable, @Immutable](12-recomposition-stability.md)*  
*Module tiếp theo: [Module 4 — Jetpack Compose Advanced](../module-4-compose-advanced/14-navigation.md)*
