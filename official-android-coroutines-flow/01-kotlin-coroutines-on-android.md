# Bài 01 — Kotlin Coroutines trên Android (Bản Dịch & Hệ Thống Hóa Chuẩn 1:1)

> **Tài liệu tham chiếu gốc:** [Kotlin coroutines on Android — Android Developers](https://developer.android.com/kotlin/coroutines?hl=vi)  
> **Áp dụng:** Kotlin 2.0+, `kotlinx.coroutines:1.11.0`, AndroidX Lifecycle 2.8+  
> **Mục tiêu:** Nắm vững toàn bộ nội dung tài liệu chính thức của Google: định nghĩa coroutine, 4 tính năng then chốt, mô hình kiến trúc phân tầng trong ví dụ đăng nhập mẫu, quản lý luồng nền, nguyên lý Main-safety với `Dispatchers` và `withContext`, cấu hình Gradle, quản trị vòng đời với `viewModelScope`, và cơ chế xử lý ngoại lệ với `try-catch`.

---

## 1. Giới thiệu: Coroutine là gì?

**Coroutine** là một mẫu thiết kế (design pattern) cho cơ chế xử lý đồng thời mà bạn có thể sử dụng trên Android để đơn giản hoá mã nguồn thực thi không đồng bộ. Coroutine đã được bổ sung chính thức vào Kotlin từ phiên bản 1.3 và được kế thừa, phát triển dựa trên các khái niệm sẵn có trong nhiều ngôn ngữ lập trình hiện đại khác.

Trên Android, coroutine giúp giải quyết hai vấn đề then chốt:
1. **Quản lý các tác vụ chạy trong thời gian dài (long-running tasks)** có thể làm tắc nghẽn luồng chính và khiến ứng dụng của bạn bị treo, đơ giật.
2. **Đảm bảo tính an toàn cho luồng chính (main-safety)**, cho phép bạn gọi bất kỳ hàm tạm ngưng nào một cách an toàn trực tiếp từ luồng giao diện người dùng.

---

## 2. Bốn Tính Năng Then Chốt của Coroutines trên Android

Google khuyến nghị chính thức sử dụng Coroutines cho quy trình lập trình không đồng bộ trên Android nhờ 4 tính năng cốt lõi sau:

### 2.1. Dung lượng nhẹ (Lightweight)
Bạn có thể chạy hàng trăm nghìn coroutine trên một luồng đơn lẻ nhờ **tính năng hỗ trợ tạm ngưng (suspension)**:
- Thao tác tạm ngưng **không chặn (block) luồng** mà coroutine đang chạy.
- Hành vi tạm ngưng giúp giải phóng thread để thực thi các coroutine khác, giúp tiết kiệm bộ nhớ đáng kể so với việc phải duy trì các Thread Java/OS truyền thống (vốn tiêu tốn xấp xỉ 1MB stack bộ nhớ cho mỗi thread).

### 2.2. Giảm thiểu rò rỉ bộ nhớ (Fewer Memory Leaks)
Kotlin Coroutines áp dụng triết lý **Tính đồng thời có cấu trúc (Structured Concurrency)**:
- Mọi coroutine đều phải được khởi chạy bên trong một phạm vi xác định (`CoroutineScope`).
- Cơ chế này đảm bảo rằng các tác vụ trong một phạm vi sẽ không bị rò rỉ ra bên ngoài hoặc tiếp tục chạy ngầm vô ích khi phạm vi đó đã kết thúc hoặc bị hủy.

### 2.3. Hỗ trợ cơ chế hủy tích hợp sẵn (Built-in Cancellation Support)
Việc hủy bỏ một tác vụ đang chạy được **tự động lan truyền** theo toàn bộ hệ thống phân cấp công việc (`Job hierarchy`). Khi một coroutine cha bị hủy, tất cả các coroutine con khởi chạy bên trong nó cũng sẽ tự động bị hủy theo, giúp giải phóng tức thì tài nguyên mạng, bộ nhớ và pin.

### 2.4. Tích hợp sâu vào các thư viện Jetpack (Jetpack Integration)
Rất nhiều thư viện chính thức trong hệ sinh thái Android Jetpack bao gồm các tiện ích mở rộng cung cấp sẵn hỗ trợ toàn diện cho coroutines:
- Một số thành phần cung cấp sẵn `CoroutineScope` gắn liền với vòng đời của chính chúng (chẳng hạn như `viewModelScope` trong ViewModel, `lifecycleScope` trong LifecycleOwner).
- Các thư viện lưu trữ như Room Database cung cấp các hàm `suspend` và luồng phản ứng `Flow` cho các câu lệnh truy vấn quan sát.

---

## 3. Tổng Quan về Ví Dụ Kiến Trúc Mẫu

Căn cứ theo [Hướng dẫn về Kiến trúc Ứng dụng của Android (Guide to App Architecture)](https://developer.android.com/topic/architecture), ví dụ trong bài học này sẽ mô phỏng một quy trình thực tế: **Gửi một yêu cầu mạng lên máy chủ và trả về kết quả cho luồng chính**, tại đó ứng dụng có thể hiển thị kết quả cho người dùng.

Hệ thống được tổ chức phân tầng rõ ràng:
1. **Tầng Giao diện (UI Layer / Activity):** Hiển thị dữ liệu và gửi sự kiện người dùng bấm nút đăng nhập tới ViewModel.
2. **Thành phần Kiến trúc ViewModel (`LoginViewModel`):** Quản lý trạng thái giao diện, khởi chạy coroutine trong `viewModelScope`, và gọi tầng lưu trữ trên luồng chính.
3. **Tầng Lưu trữ (`LoginRepository`):** Điều phối yêu cầu đăng nhập, đảm bảo tính Main-safe và chuyển tiếp công việc tới tầng nguồn dữ liệu.
4. **Tầng Nguồn Dữ liệu (`LoginRemoteDataSource`):** Thực hiện kết nối mạng HTTP I/O trên luồng chạy ở chế độ nền.

```
Mô hình Phân tầng Kiến trúc Chuẩn:
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│ View (Activity) │ ────► │ LoginViewModel  │ ────► │ LoginRepository │ ────► │ RemoteDataSource│
│  (Main Thread)  │       │(viewModelScope) │       │   (Main-safe)   │       │   (IO Thread)   │
└─────────────────┘       └─────────────────┘       └─────────────────┘       └─────────────────┘
```

---

## 4. Thông Tin về Phần Phụ Thuộc (Dependency Setup)

Để sử dụng coroutine trong dự án Android, bạn cần thêm phần phụ thuộc thư viện vào tệp cấu hình Gradle của ứng dụng:

### Groovy DSL (`build.gradle` cấp module `app`)
```groovy
dependencies {
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.11.0'
    implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.8.7'
}
```

### Kotlin DSL (`build.gradle.kts` cấp module `app`)
```kotlin
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.11.0")
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.8.7")
}
```

---

## 5. Thực Thi trong Luồng ở Chế Độ Nền (Background Thread Execution)

### 5.1. Nguy cơ Tắc nghẽn Luồng Chính (ANR)
Việc tạo một yêu cầu mạng trên luồng chính sẽ khiến luồng này phải chờ đợi (chặn hoặc blocked) cho đến khi nhận được phản hồi từ máy chủ. 

Vì luồng chính đã bị chặn, hệ điều hành Android **không thể gọi phương thức `onDraw()`** để vẽ các khung hình tiếp theo trên màn hình. Hệ quả là ứng dụng của bạn sẽ bị treo, đứng hình, và nếu tình trạng này kéo dài quá 5 giây, hệ điều hành sẽ kích hoạt hộp thoại **ANR (Ứng dụng không phản hồi / Application Not Responding)** buộc người dùng phải đóng ứng dụng.

```
Rủi ro khi luồng chính bị chặn:
Luồng chính (Main Thread): ───[onDraw()]───► [BỊ CHẶN: Gọi mạng 5s...] ───► [ANR Dialog Xuất Hiện!]
                                                   ▲
                                                   │ Không thể gọi onDraw() -> Ứng dụng bị treo!
```

---

### 5.2. Mã Nguồn Thực Thi Mạng Truyền Thống

Hãy xem cách lớp `LoginRepository` dưới đây thực hiện một yêu cầu mạng đồng bộ bằng `HttpURLConnection`:

```kotlin
sealed class Result<out R> {
    data class Success<out T>(val data: T) : Result<T>()
    data class Error(val exception: Exception) : Result<Nothing>()
}

class LoginRepository(
    private val responseParser: LoginResponseParser
) {
    private const val loginUrl = "https://example.com/api/login"

    // Hàm này thực hiện blocking I/O đồng bộ
    fun makeLoginRequest(
        jsonBody: String
    ): Result<LoginResponse> {
        val url = URL(loginUrl)
        (url.openConnection() as? HttpURLConnection)?.run {
            requestMethod = "POST"
            setRequestProperty("Content-Type", "application/json; utf-8")
            setRequestProperty("Accept", "application/json")
            doOutput = true
            outputStream.write(jsonBody.toByteArray())

            val responseCode = responseCode
            return if (responseCode == HttpURLConnection.HTTP_OK) {
                val response = inputStream.bufferedReader().use { it.readText() }
                Result.Success(responseParser.parse(response))
            } else {
                Result.Error(IOException("Lỗi phản hồi mạng: $responseCode - $responseMessage"))
            }
        }
        return Result.Error(IOException("Không thể kết nối đến máy chủ"))
    }
}
```

Hàm `makeLoginRequest` ở trên là **đồng bộ và sẽ chặn luồng gọi**. Để tạo phản hồi cho yêu cầu đăng nhập, hàm này cấp phát bộ nhớ, đọc và ghi dữ liệu qua kết nối mạng, đồng thời phân tích cú pháp JSON phản hồi. Mỗi thao tác này đều có thể mất từ vài trăm mili-giây đến vài giây. Nếu gọi trực tiếp từ Main thread, giao diện người dùng chắc chắn sẽ bị đơ cứng.

---

## 6. Dùng Coroutine để Đảm Bảo An Toàn Cho Luồng Chính (Main-Safety)

### 6.1. Định nghĩa Hàm Main-Safe
> [!IMPORTANT]
> **Quy tắc của Google:**  
> Một hàm được coi là **Main-safe (an toàn cho luồng chính)** khi hàm này không chặn các bản cập nhật giao diện người dùng trên luồng chính.

Hàm `makeLoginRequest` ở mục trên **không an toàn cho luồng chính** vì lệnh gọi nó từ luồng chính sẽ chặn giao diện. Để biến một hàm thành main-safe, chúng ta sử dụng coroutine để di chuyển thao tác thực thi sang một luồng khác ở chế độ nền.

---

### 6.2. Các Bộ Điều Phối Luồng (Dispatchers)
Kotlin Coroutines sử dụng **Dispatchers** để xác định luồng nào sẽ thực thi công việc:

| Dispatcher | Luồng thực thi | Trường hợp sử dụng chuẩn trên Android |
| :--- | :--- | :--- |
| **`Dispatchers.Main`** | Luồng giao diện chính (Main Thread) | Sử dụng bộ điều phối này để chạy một coroutine trên luồng Android chính. Chỉ dùng cho việc tương tác với giao diện người dùng, gọi các hàm `suspend` nhẹ, và cập nhật LiveData / StateFlow. |
| **`Dispatchers.IO`** | Thread Pool mở rộng (tối đa 64 threads) | Bộ điều phối này được tối ưu hoá để thực hiện các thao tác vào/ra đĩa hoặc mạng bên ngoài luồng chính. Ví dụ: gọi API Retrofit, đọc/ghi tệp tin, truy vấn cơ sở dữ liệu Room. |
| **`Dispatchers.Default`** | Thread Pool theo số lõi CPU | Bộ điều phối này được tối ưu hoá để thực hiện các công việc tiêu tốn nhiều tài nguyên CPU bên ngoài luồng chính. Ví dụ: sắp xếp danh sách dữ liệu lớn, phân tích chuỗi JSON phức tạp, xử lý ảnh. |

---

### 6.3. Tối Ưu Hóa với `withContext`
Để đảm bảo hàm `makeLoginRequest` an toàn cho luồng chính, chúng ta chuyển việc thực thi sang `Dispatchers.IO` bằng hàm `withContext`:

```kotlin
class LoginRepository(
    private val responseParser: LoginResponseParser
) {
    private const val loginUrl = "https://example.com/api/login"

    // Hàm suspend này đảm bảo Main-safe tuyệt đối
    suspend fun makeLoginRequest(
        jsonBody: String
    ): Result<LoginResponse> = withContext(Dispatchers.IO) {
        // Toàn bộ khối mã này được chuyển sang chạy trên luồng của Dispatchers.IO
        val url = URL(loginUrl)
        (url.openConnection() as? HttpURLConnection)?.run {
            requestMethod = "POST"
            setRequestProperty("Content-Type", "application/json; utf-8")
            setRequestProperty("Accept", "application/json")
            doOutput = true
            outputStream.write(jsonBody.toByteArray())

            val responseCode = responseCode
            return@withContext if (responseCode == HttpURLConnection.HTTP_OK) {
                val response = inputStream.bufferedReader().use { it.readText() }
                Result.Success(responseParser.parse(response))
            } else {
                Result.Error(IOException("Lỗi mạng: $responseCode - $responseMessage"))
            }
        }
        return@withContext Result.Error(IOException("Không thể mở kết nối mạng"))
    }
}
```

#### Phân tích Hiệu Năng của `withContext`:
> [!NOTE]
> So với việc triển khai callback truyền thống, `withContext()` không tạo thêm chi phí (overhead) không cần thiết. Trong hầu hết các trường hợp, `withContext()` tối ưu hoá việc chuyển đổi ngữ cảnh bằng cách tái sử dụng luồng có sẵn trong thread pool của Dispatcher và tiếp tục (`resume`) lại coroutine trên luồng ban đầu mà không cần cấp phát thêm cấu trúc quản lý phức tạp.

---

### 6.4. Quản Lý Vòng Đời với `viewModelScope`

Thành phần `ViewModel` trong Android Architecture Components cung cấp sẵn một phạm vi coroutine có tên là **`viewModelScope`**. Khi ViewModel bị xóa khỏi bộ nhớ (onCleared), `viewModelScope` sẽ **tự động hủy bỏ tất cả các coroutine đang chạy bên trong nó**, giúp ngăn chặn triệt để tình trạng lãng phí tài nguyên mạng và rò rỉ bộ nhớ.

Bạn sử dụng coroutine builder `launch` để khởi chạy một coroutine mới:

```kotlin
class LoginViewModel(
    private val loginRepository: LoginRepository
) : ViewModel() {

    fun login(username: String, token: String) {
        // Tạo một coroutine mới gắn liền với vòng đời của ViewModel
        viewModelScope.launch {
            val jsonBody = "{ \"username\": \"$username\", \"token\": \"$token\" }"
            // makeLoginRequest là hàm suspend main-safe, có thể gọi trực tiếp từ Main thread
            val result = loginRepository.makeLoginRequest(jsonBody)
            
            // Xử lý kết quả trả về tại đây trên luồng chính
            when (result) {
                is Result.Success -> {
                    // Cập nhật trạng thái đăng nhập thành công lên giao diện
                }
                is Result.Error -> {
                    // Hiển thị thông báo lỗi lên giao diện
                }
            }
        }
    }
}
```

```
Quy trình thực thi tuần tự của Coroutine:
1. ViewModel gọi viewModelScope.launch trên Luồng chính (Main Thread).
2. Gọi makeLoginRequest(jsonBody) -> Coroutine TẠM NGƯNG (suspend), giải phóng Luồng chính.
3. withContext(Dispatchers.IO) đẩy công việc mạng sang luồng nền.
4. Yêu cầu mạng hoàn tất -> Coroutine TIẾP TỤC (resume) trở lại Luồng chính.
5. Kết quả được xử lý và cập nhật lên UI mà không bao giờ làm đơ màn hình!
```

---

## 7. Xử Lý Ngoại Lệ (Handling Exceptions)

Để xử lý các ngoại lệ mà tầng `Repository` có thể ném ra trong quá trình thực thi bất đồng bộ, hãy sử dụng tính năng hỗ trợ tích hợp sẵn của Kotlin: **khối lệnh `try-catch`**.

Trong ví dụ dưới đây, `LoginViewModel` bọc lệnh gọi mạng trong khối `try-catch`:

```kotlin
class LoginViewModel(
    private val loginRepository: LoginRepository
) : ViewModel() {

    fun makeLoginRequest(username: String, token: String) {
        viewModelScope.launch {
            val jsonBody = "{ \"username\": \"$username\", \"token\": \"$token\" }"
            val result = try {
                // Gọi hàm suspend trong khối try
                loginRepository.makeLoginRequest(jsonBody)
            } catch (e: Exception) {
                // Bắt và xử lý ngoại lệ bất kỳ (ví dụ: đứt cáp mạng, timeout)
                Result.Error(Exception("Lỗi mạng: ${e.message}"))
            }

            when (result) {
                is Result.Success -> {
                    // Thông báo đăng nhập thành công
                }
                is Result.Error -> {
                    // Thông báo lỗi cho người dùng
                }
            }
        }
    }
}
```

> [!CAUTION]
> **Lưu ý quan trọng về Hủy bỏ:**  
> Nếu bạn muốn bắt ngoại lệ cụ thể, hãy bắt các lớp con như `IOException`. Nếu bạn dùng `catch (e: Exception)` chung, hãy đảm bảo rằng bạn **không nuốt chửng `CancellationException`**, để cơ chế hủy coroutine của Android tiếp tục hoạt động chính xác.

---

## 8. Tài Nguyên Tham Khảo Mở Rộng từ Google

Để đào sâu hơn về Coroutines trên Android, bạn có thể tham khảo các tài liệu chính thức sau:
- **Cải thiện hiệu năng ứng dụng bằng coroutine của Kotlin:** Hướng dẫn quản lý tác vụ nền nặng và tối ưu hoá CPU.
- **Thực hành tốt nhất cho coroutine trên Android:** Bộ quy tắc thiết kế kiến trúc chuẩn do Google Engineering ban hành (xem chi tiết tại [Bài 02](02-coroutines-best-practices.md)).
- **Kiểm thử coroutine trên Android:** Hướng dẫn viết Unit Test cho coroutine với `TestDispatcher` (xem chi tiết tại [Bài 06](06-testing-coroutines-and-flow.md)).
- **Codelab Coroutines của Google:** Bài thực hành tương tác từng bước xây dựng ứng dụng với Coroutines và Lifecycle.
