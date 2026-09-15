# Bài 06 — Kiểm Thử Coroutines và Flow trên Android (Testing Coroutines and Flow)

> **Tài liệu tham chiếu gốc:**
> - [Testing coroutines on Android — Android Developers](https://developer.android.com/kotlin/coroutines/test?hl=vi)
> - [Testing Kotlin flows on Android — Android Developers](https://developer.android.com/kotlin/flow/test?hl=vi)
>
> **Áp dụng:** `kotlinx-coroutines-test:1.11.0`, CashApp `turbine:1.2.0`, JUnit4 / JUnit5  
> **Mục tiêu:** Nắm vững toàn bộ kỹ thuật kiểm thử tự động (Unit Test) cho Coroutines và Flow theo chuẩn Google: hàm kiểm thử `runTest`, hai bộ điều phối `StandardTestDispatcher` và `UnconfinedTestDispatcher`, thiết lập `MainDispatcherRule`, kỹ thuật chèn TestDispatcher / TestScope, kiểm thử luồng với thư viện Turbine, và xử lý kiểm thử `StateFlow` sinh bởi `stateIn`.

---

## 1. Thiết Lập Thư Viện Kiểm Thử (Dependencies)

Thêm các thư viện sau vào khối `dependencies` trong tệp `build.gradle.kts`:

```kotlin
dependencies {
    // 1. Thư viện kiểm thử Coroutines chính thức của JetBrains / Google
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.11.0")

    // 2. Thư viện kiểm thử Flow chuyên dụng (Turbine - CashApp)
    testImplementation("app.cash.turbine:turbine:1.2.0")

    // 3. Thư viện kiểm thử JUnit và Assertions
    testImplementation("junit:junit:4.13.2")
    testImplementation("com.google.truth:truth:1.4.2")
}
```

---

## 2. Gọi Hàm Tạm Ngưng trong Kiểm Thử: Hàm `runTest`

Trong mã kiểm thử đơn vị, bạn không thể gọi trực tiếp một hàm `suspend` từ một hàm test thông thường vì test runner chạy đồng bộ. 

Google khuyến nghị sử dụng **`runTest`** làm coroutine builder cho mọi bài kiểm thử:

```kotlin
class UserRepositoryTest {

    @Test
    fun fetchUser_returnsSuccess() = runTest {
        // Bên trong runTest là một TestScope
        // Bạn có thể gọi bất kỳ hàm suspend nào một cách an toàn
        val repository = UserRepository(fakeDataSource)
        val user = repository.fetchUserData()

        assertThat(user.name).isEqualTo("Alice")
    }
}
```

### Cơ chế Bỏ qua Thời gian Ảo (Virtual Time Skipping):
> [!NOTE]
> `runTest` tự động điều khiển thời gian ảo. Bất kỳ lệnh `delay(5000)` nào bên trong `runTest` sẽ được **tua nhanh ngay lập tức (skip delay)** trong 0ms thực tế trên đồng hồ CPU, giúp bài test chạy tức thì mà không phải chờ đợi lãng phí thời gian.

---

## 3. Các Bộ Điều Phối Kiểm Thử (Test Dispatchers)

Thư viện `kotlinx-coroutines-test` cung cấp hai triển khai của `TestDispatcher`:

### 3.1. `StandardTestDispatcher` (Bộ điều phối kiểm thử tiêu chuẩn)
Khi bạn khởi chạy coroutine trên `StandardTestDispatcher`, coroutine đó **không chạy ngay lập tức** mà được đưa vào hàng đợi của bộ lập lịch (scheduler):
- Bạn có toàn quyền kiểm soát thời điểm coroutine chạy bằng cách gọi:
  - `advanceUntilIdle()`: Chạy toàn bộ các coroutine đang chờ trong hàng đợi cho đến khi rảnh rỗi.
  - `advanceTimeBy(millis)`: Tua thời gian ảo lên một khoảng cụ thể.
  - `runCurrent()`: Chạy các tác vụ đang được lên lịch ở thời điểm hiện tại.

### 3.2. `UnconfinedTestDispatcher` (Bộ điều phối kiểm thử tự do)
Khi khởi chạy một coroutine mới trên `UnconfinedTestDispatcher`, nó sẽ **thực thi ngay lập tức một cách háo hức (eagerly)** trên luồng hiện tại cho đến điểm tạm ngưng đầu tiên:
- Phù hợp cho các bài kiểm thử đơn giản mà bạn không cần kiểm soát thứ tự thực thi chi tiết theo thời gian ảo.

| Tiêu chí | `StandardTestDispatcher` | `UnconfinedTestDispatcher` |
| :--- | :--- | :--- |
| **Hành vi khởi chạy** | Xếp vào hàng đợi, chờ lệnh thực thi | Chạy ngay lập tức (Eager execution) |
| **Kiểm soát thời gian** | Thủ công qua `advanceUntilIdle()`, `advanceTimeBy()` | Tự động chạy đến điểm suspend tiếp theo |
| **Mặc định trong `runTest`** | Được sử dụng làm dispatcher mặc định | Phải cấu hình thủ công |

---

## 4. Đặt Bộ Điều Phối Chính (Setting the Main Dispatcher)

Trong môi trường Unit Test trên máy tính cục bộ (JVM), không hề có vòng lặp sự kiện Android Looper, do đó việc truy cập `Dispatchers.Main` sẽ ném ra lỗi `IllegalStateException: Module with the Main dispatcher had failed to initialize`.

Để khắc phục, Google khuyến nghị tạo một **JUnit Test Rule** có thể tái sử dụng để thay thế `Dispatchers.Main` bằng một `TestDispatcher`:

```kotlin
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.test.*
import org.junit.rules.TestWatcher
import org.junit.runner.Description

class MainDispatcherRule(
    val testDispatcher: TestDispatcher = UnconfinedTestDispatcher()
) : TestWatcher() {

    override fun starting(description: Description) {
        // Thay thế Dispatchers.Main bằng testDispatcher trước khi bài test chạy
        Dispatchers.setMain(testDispatcher)
    }

    override fun finished(description: Description) {
        // Khôi phục lại Dispatchers.Main ban đầu sau khi bài test kết thúc
        Dispatchers.resetMain()
    }
}
```

---

## 5. Kiểm Thử ViewModel Sử Dụng Coroutines

Dưới đây là một bài kiểm thử hoàn chỉnh cho `LoginViewModel`, áp dụng cả `MainDispatcherRule` và `StandardTestDispatcher`:

```kotlin
class LoginViewModelTest {

    // 1. Áp dụng MainDispatcherRule để mock Dispatchers.Main
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule(StandardTestDispatcher())

    private val fakeRepository = FakeLoginRepository()

    @Test
    fun login_success_updatesUiStateToSuccess() = runTest {
        val viewModel = LoginViewModel(fakeRepository)

        // Thực hiện hành vi đăng nhập
        viewModel.login("admin", "123456")

        // Ban đầu trạng thái là Loading
        assertThat(viewModel.loginUiState.value).isEqualTo(LoginUiState.Loading)

        // Tua nhanh cho đến khi tất cả các coroutine trong viewModelScope chạy xong
        advanceUntilIdle()

        // Trạng thái đã được cập nhật thành Success
        val currentState = viewModel.loginUiState.value
        assertThat(currentState).isInstanceOf(LoginUiState.Success::class.java)
    }
}
```

---

## 6. Kiểm Thử Kotlin Flow

Kiểm thử luồng dữ liệu bất đồng bộ thường gặp thách thức vì Flow phát nhiều giá trị theo thời gian.

### 6.1. Phương Pháp Cơ Bản: Gom Luồng với `toList()` hoặc `first()`
Đối với các Cold Flow hữu hạn, bạn có thể gọi `first()` hoặc thu thập thành `List`:

```kotlin
@Test
fun repository_emitsFavorites() = runTest {
    val repository = NewsRepository(fakeRemoteDataSource, fakeUserData)
    
    // Lấy giá trị đầu tiên phát ra từ Flow
    val firstItem = repository.favoriteLatestNews.first()
    assertThat(firstItem).isNotEmpty()
}
```

---

### 6.2. Tiêu Chuẩn Vàng: Kiểm Thử Flow Bằng Thư Viện Turbine

Google khuyến nghị sử dụng thư viện **Turbine** của CashApp để kiểm thử các luồng phức tạp một cách trực quan, rõ ràng và có thứ tự:

```kotlin
import app.cash.turbine.test

@Test
fun streamNews_emitsLoadingThenSuccess() = runTest {
    val repository = NewsRepository(fakeRemoteDataSource)

    // Khối test {} của Turbine mở một kênh lắng nghe độc lập
    repository.favoriteLatestNews.test {
        // 1. Chờ phần tử đầu tiên phát ra
        val item1 = awaitItem()
        assertThat(item1).hasSize(2)

        // 2. Kích hoạt thay đổi từ phía nguồn dữ liệu
        fakeRemoteDataSource.emitNewArticle(newArticle)

        // 3. Chờ phần tử thứ hai phát ra sau khi có cập nhật
        val item2 = awaitItem()
        assertThat(item2).hasSize(3)

        // 4. Xác nhận không còn sự kiện nào phát ra thêm và luồng hoàn thành
        cancelAndIgnoreRemainingEvents()
    }
}
```

---

## 7. Kiểm Thử `StateFlow` Sinh Bởi `stateIn`

### Vấn Đề với `SharingStarted.WhileSubscribed(5000)`
Khi bạn biến đổi Cold Flow thành Hot StateFlow bằng:
```kotlin
val uiState = repository.dataFlow.stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(5000),
    initialValue = UiState.Loading
)
```
Trong môi trường kiểm thử, nếu bạn chỉ đọc `viewModel.uiState.value` mà không có bất kỳ bên nào gọi `collect`, số lượng subscriber bằng `0`. Do đó, **luồng upstream sẽ không bao giờ được kích hoạt** và state mãi mãi dừng ở `Loading`!

### Giải Pháp: Sử Dụng Turbine hoặc `backgroundScope`
1. **Dùng Turbine:** Gọi `viewModel.uiState.test { ... }`. Turbine sẽ tự đóng vai một subscriber, kích hoạt upstream chạy và cho phép bạn kiểm tra từng bước chuyển trạng thái.
2. **Dùng `backgroundScope`:**
```kotlin
@Test
fun stateIn_withSubscriber_emitsSuccess() = runTest {
    val viewModel = MyViewModel(fakeRepository)

    // Đăng ký một subscriber chạy ngầm trong backgroundScope của bài test
    backgroundScope.launch(UnconfinedTestDispatcher(testScheduler)) {
        viewModel.uiState.collect()
    }

    // Lúc này subscriber > 0, WhileSubscribed kích hoạt upstream!
    assertThat(viewModel.uiState.value).isEqualTo(UiState.Success(data))
}
```

---

## 8. Tóm Tắt Quy Trình Kiểm Thử Chuẩn Google

1. **Luôn bao bọc hàm test bằng `runTest`** để tận dụng cơ chế bỏ qua độ trễ thời gian ảo.
2. **Luôn sử dụng `MainDispatcherRule`** để mock `Dispatchers.Main` trên JVM Unit Tests.
3. **Inject `CoroutineDispatcher`** vào Repository/ViewModel thay vì hardcode.
4. **Sử dụng Turbine (`flow.test { ... }`)** làm giải pháp số 1 để kiểm tra chuỗi phát sự kiện của Flow.
5. **Đăng ký subscriber** qua Turbine hoặc `backgroundScope` khi kiểm thử `StateFlow` sử dụng `WhileSubscribed`.
