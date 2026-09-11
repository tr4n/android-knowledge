# Bài 20 — WorkManager + Flow: Reliable Background Work & Progress Observation

> **Module:** 5 — Advanced Patterns & Integration  
> **Mức độ:** Senior / Staff Android Architect  
> **Prerequisites:** Module 1 (Coroutines & Flow), Module 2 (Architecture), Module 5 (Bài 19 - Hilt DI)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

**WorkManager** là giải pháp tiêu chuẩn chính thức của Google để xử lý **Tác vụ nền bền vững (Guaranteed Background Work)** trên hệ điều hành Android. Một tác vụ được gọi là "bền vững" (Persistent) khi nó được hệ thống đảm bảo sẽ thực thi thành công ngay cả khi người dùng tắt ứng dụng (App killed) hoặc thiết bị khởi động lại (Device reboot).

```
   ┌─────────────────────────────────────────────────────────────┐
   │                       WORK REQUEST                          │
   │   (OneTime / Periodic + Constraints: Network, Battery, etc) │
   └──────────────────────────────┬──────────────────────────────┘
                                  │
                                  ▼
   ┌─────────────────────────────────────────────────────────────┐
   │             INTERNAL SQLITE DATABASE (workdb)               │
   │     Lưu trữ bền vững trạng thái & tham số công việc         │
   └──────────────────────────────┬──────────────────────────────┘
                                  │
            Thỏa mãn điều kiện    │ Scheduler (JobScheduler API 23+)
                                  ▼
   ┌─────────────────────────────────────────────────────────────┐
   │                     COROUTINE WORKER                        │
   │        Thực thi suspend fun doWork(): Result                │
   │        setProgress(workDataOf("progress" to 85))            │
   └──────────────────────────────┬──────────────────────────────┘
                                  │
                 Reactive updates │ InvalidationTracker
                                  ▼
   ┌─────────────────────────────────────────────────────────────┐
   │                     FLOW<WORKINFO>                          │
   │      Quan sát tiến độ & trạng thái thời gian thực trong UI  │
   └─────────────────────────────────────────────────────────────┘
```

### Các thuật ngữ cốt lõi:
- **`CoroutineWorker`:** Lớp cơ sở trừu tượng hỗ trợ Kotlin Coroutines nguyên bản. Phương thức cốt lõi `suspend fun doWork(): Result` chạy ngầm trên `Dispatchers.Default` (hoặc Dispatcher tùy biến), cho phép gọi trực tiếp các `suspend function`.
- **`WorkRequest`:** Đại diện cho một yêu cầu thực thi công việc:
  - `OneTimeWorkRequest`: Tác vụ chỉ chạy một lần duy nhất (ví dụ: upload ảnh, gửi log).
  - `PeriodicWorkRequest`: Tác vụ lặp lại định kỳ (ví dụ: dọn dẹp cache mỗi 24 giờ). **Thời gian tối thiểu giữa 2 lần chạy là 15 phút**.
- **`Constraints` (Ràng buộc hệ thống):** Các điều kiện mà thiết bị phải thỏa mãn thì tác vụ mới được phép chạy: `NetworkType.CONNECTED` (có mạng), `NetworkType.UNMETERED` (phải là WiFi), `setRequiresBatteryNotLow(true)` (pin không yếu), `setRequiresCharging(true)` (đang cắm sạc).
- **`WorkInfo` & `WorkInfo.State`:** Đối tượng phản ánh trạng thái hiện tại của công việc: `ENQUEUED` (đang chờ), `RUNNING` (đang chạy), `SUCCEEDED` (thành công), `FAILED` (thất bại), `BLOCKED` (bị nghẽn bởi task trước), `CANCELLED` (đã hủy).
- **`Expedited Work`:** Cơ chế ưu tiên thực thi ngay lập tức cho các tác vụ quan trọng khẩn cấp (như đồng bộ tin nhắn đến, xử lý thanh toán), tuân thủ cơ chế hạn mức (Quota) của Android 12+.
- **`@HiltWorker` & `@AssistedInject`:** Cặp annotation cho phép tích hợp Dependency Injection với Hilt vào Worker mà không vi phạm quy tắc khởi tạo của Android Framework.

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Kiến trúc Lưu trữ Bền vững (SQLite Persistence Layer)
Tại sao WorkManager có thể sống sót qua việc khởi động lại thiết bị (Reboot)?
- Khi bạn gọi `workManager.enqueue(workRequest)`, WorkManager **không bao giờ** khởi chạy tiến trình trực tiếp trên RAM ngay lập tức.
- Thay vào đó, nó ghi toàn bộ thông tin của `WorkSpec` (ID, ClassName, Constraints, InputData, Trạng thái `ENQUEUED`) vào một cơ sở dữ liệu SQLite cục bộ được quản lý bởi Room Database nội bộ (`androidx.work.workdb`).
- Sau khi commit thành công vào SQLite, WorkManager đăng ký điều kiện với **JobScheduler** của Android OS (trên API 23+).
- Khi thiết bị khởi động lại, BroadcastReceiver `RescheduleReceiver` của WorkManager nhận sự kiện `ACTION_BOOT_COMPLETED`, đọc lại SQLite và nạp lại các Job vào JobScheduler. **Không bao giờ có chuyện mất mát tác vụ!**

### 2.2 Cơ chế Reactive Flow qua `getWorkInfoByIdFlow()`
Phương thức `workManager.getWorkInfoByIdFlow(uuid)` trả về một `Flow<WorkInfo>`.
- Bên dưới, WorkManager sử dụng cơ chế `InvalidationTracker` của Room Database trên bảng `WorkSpec`.
- Mỗi khi `CoroutineWorker` gọi `setProgress(data)` hoặc khi trạng thái công việc chuyển từ `RUNNING` sang `SUCCEEDED`, Room InvalidationTracker phát hiện có bản ghi bị thay đổi trong SQLite và kích hoạt `Flow` phát ra giá trị `WorkInfo` mới nhất tới ViewModel và UI.

### 2.3 Sơ đồ Chuyển đổi Trạng thái (Work Life Cycle FSM)

```
                       [ Enqueue ]
                            │
                            ▼
                     ┌──────────────┐
                     │   ENQUEUED   │◄───────────────────────┐
                     └──────┬───────┘                        │
                            │ Constraints Satisfied          │
                            ▼                                │ Result.retry()
                     ┌──────────────┐                        │
       ┌────────────►│   RUNNING    ├────────────────────────┘
       │             └──────┬───────┘
       │                    │
       │    ┌───────────────┼───────────────┐
       │    │               │               │
       │    ▼               ▼               ▼
       │ ┌───────────┐ ┌───────────┐ ┌───────────┐
       │ │ SUCCEEDED │ │  FAILED   │ │ CANCELLED │
       │ └───────────┘ └───────────┘ └───────────┘
       │   (Terminal)    (Terminal)    (Terminal)
       │
       └── Chained Next Work
```

### 2.4 Hạn chế của Doze Mode & Giải pháp Expedited Work
Từ Android 6.0, hệ điều hành áp dụng **Doze Mode** (chế độ ngủ sâu khi thiết bị không sạc và đứng yên). Các Background Service thông thường sẽ bị ngắt mạng và dừng CPU.
- WorkManager tự động trì hoãn các tác vụ cho đến khi thiết bị bước vào **Maintenance Window** của Doze Mode.
- Nếu tác vụ cực kỳ cấp bách (High priority), nhà phát triển kích hoạt **Expedited Work** qua `setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)`. Trên Android 12+, hệ điều hành cấp phát một hạn mức thời gian thực thi ngay lập tức (Execution Quota) mà không cần chờ Maintenance Window.

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Sự suy tàn của Background Service truyền thống
Trong các phiên bản Android cũ, lập trình viên thường dùng `Service` hoặc `IntentService` để đồng bộ dữ liệu ngầm. Tuy nhiên:
1. **Android 8.0 (API 26) - Background Execution Limits:** Cấm hoàn toàn việc gọi `startService()` khi ứng dụng đang ở background. Ứng dụng sẽ crash với `IllegalStateException: Not allowed to start service Intent`.
2. **Android 14 (API 34) - Strict Foreground Service Types:** Nếu sử dụng Foreground Service, bắt buộc phải khai báo chính xác loại hình (`dataSync`, `mediaPlayback`, `camera`) trong Manifest và xin runtime permission. Nếu sử dụng sai mục đích, Google Play Store sẽ từ chối duyệt ứng dụng!
3. **`viewModelScope` bị hủy:** Nếu người dùng chuyển sang ứng dụng khác và Android thiếu RAM, Activity/Process sẽ bị hủy. Mọi tác vụ chạy bằng Coroutine trong `viewModelScope` hoặc `lifecycleScope` sẽ bị ngắt giữa chừng, làm hỏng dữ liệu hoặc mất file tải lên.

### 3.2 Giá trị vượt trội của WorkManager
- **100% Guaranteed Execution:** Bất chấp pin yếu, mất mạng, thiết bị sập nguồn hay app bị người dùng vuốt đóng khỏi màn hình đa nhiệm (Recent Apps), tác vụ vẫn được tiếp tục khi điều kiện môi trường hội tụ đủ.
- **Tiết kiệm pin thông minh:** Hệ thống tự động gom (batch) các tác vụ từ nhiều ứng dụng khác nhau để đánh thức CPU và Radio cùng một lúc, kéo dài tuổi thọ pin của thiết bị.
- **Quan sát trạng thái phản ứng (Reactive Observation):** Tích hợp hoàn hảo với Kotlin Flow và Compose UI để hiển thị tiến trình mượt mà.

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Bảng so sánh lựa chọn giải pháp xử lý bất đồng bộ trong Android

| Kịch bản nghiệp vụ | Giải pháp chuẩn mực | Lý do lựa chọn |
|---|---|---|
| Gọi API lấy dữ liệu hiển thị lên màn hình hiện tại | Kotlin Coroutines (`viewModelScope`) | Tác vụ gắn liền với UI; cần hủy ngay khi người dùng thoát màn hình để giải phóng RAM. |
| Người dùng bấm nút "Lưu bài viết vào yêu thích" | Room `@Insert` với Coroutine | Thao tác cực nhanh (vài millisecond), hoàn thành trước khi màn hình kịp đóng. |
| Tải ảnh/video dung lượng lớn (100MB+) lên Server | **WorkManager (`OneTimeWorkRequest`)** | Cần đảm bảo upload thành công dù người dùng tắt app hoặc mất mạng giữa chừng. |
| Dọn dẹp cache, đồng bộ hóa dữ liệu danh mục hàng ngày | **WorkManager (`PeriodicWorkRequest`)** | Tác vụ định kỳ lặp lại, cần ràng buộc sạc pin và kết nối WiFi. |
| Phát nhạc nền, điều hướng GPS theo thời gian thực | Foreground Service | Tác vụ tương tác trực tiếp liên tục với người dùng, cần thông báo thường trực trên thanh trạng thái. |

### 4.2 Do's and Don'ts từ Google Android Architecture

#### DO:
1. **Luôn dùng `CoroutineWorker`:** Kế thừa `CoroutineWorker` thay vì `Worker` (chạy trên thread pool cũ) hoặc `ListenableWorker` (dùng Guava ListenableFuture phức tạp).
2. **Báo cáo tiến độ thời gian thực với `setProgress()`:** Cho phép UI lắng nghe phần trăm hoàn thành thông qua `workInfo.progress`.
3. **Sử dụng `ExistingWorkPolicy.KEEP` hoặc `REPLACE`:** Khi enqueue Unique Work, luôn chỉ định rõ chính sách để ngăn chặn việc người dùng bấm liên tục tạo ra hàng chục tác vụ trùng lặp.
4. **Kiểm tra `isStopped` trong vòng lặp dài:** Nếu hệ điều hành hủy worker vì vi phạm ràng buộc (ví dụ: người dùng tắt WiFi), hãy kiểm tra điều kiện để thoát vòng lặp an toàn và dọn dẹp file tạm.

#### DON'T:
1. **KHÔNG dùng WorkManager cho các tác vụ cần chạy chính xác từng giây:** WorkManager được thiết kế cho *deferrable work* (tác vụ có thể trì hoãn). Để đặt báo thức chính xác lúc 07:00:00 AM, bắt buộc phải dùng `AlarmManager.setExactAndAllowWhileIdle()`.
2. **KHÔNG đặt chu kỳ `PeriodicWorkRequest` dưới 15 phút:** Android OS đặt giới hạn cứng (Hard Limit) 15 phút. Nếu bạn đặt `PeriodicWorkRequestBuilder(5, TimeUnit.MINUTES)`, hệ thống sẽ tự động ép lên 15 phút.
3. **KHÔNG truyền các đối tượng phức tạp lớn qua `Data`:** `workDataOf()` có giới hạn dung lượng **10KB (MAX_DATA_BYTES)**. Chỉ truyền ID, file path hoặc primitive types; không truyền ảnh base64 hoặc list đối tượng khổng lồ.

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Chúng ta sẽ xây dựng tính năng: **Nén ảnh & Tải lên máy chủ ngầm (Background Image Processor)** với `@HiltWorker`, báo cáo tiến độ qua `Flow<WorkInfo>` và hiển thị trên Jetpack Compose.

### Bước 1: Khởi tạo Hilt Custom WorkManager Configuration

Tạo tệp `di/WorkerModule.kt`:

```kotlin
package com.example.workmanager.di

import android.content.Context
import androidx.hilt.work.HiltWorkerFactory
import androidx.work.Configuration
import androidx.work.WorkManager
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object WorkerModule {

    @Provides
    @Singleton
    fun provideWorkManager(
        @ApplicationContext context: Context
    ): WorkManager {
        return WorkManager.getInstance(context)
    }
}
```

Cấu hình Application kế thừa `Configuration.Provider` để Hilt kiểm soát việc khởi tạo Worker:

```kotlin
package com.example.workmanager

import android.app.Application
import androidx.hilt.work.HiltWorkerFactory
import androidx.work.Configuration
import dagger.hilt.android.HiltAndroidApp
import javax.inject.Inject

@HiltAndroidApp
class WorkApplication : Application(), Configuration.Provider {

    @Inject
    lateinit var workerFactory: HiltWorkerFactory

    override val workManagerConfiguration: Configuration
        get() = Configuration.Builder()
            .setWorkerFactory(workerFactory)
            .setMinimumLoggingLevel(android.util.Log.DEBUG)
            .build()
}
```

### Bước 2: Triển khai `CoroutineWorker` với `@HiltWorker`

Tạo tệp `worker/ImageProcessingWorker.kt`:

```kotlin
package com.example.workmanager.worker

import android.content.Context
import androidx.hilt.work.HiltWorker
import androidx.work.CoroutineWorker
import androidx.work.WorkerParameters
import androidx.work.workDataOf
import dagger.assisted.Assisted
import dagger.assisted.AssistedInject
import kotlinx.coroutines.delay
import java.io.File

@HiltWorker
class ImageProcessingWorker @AssistedInject constructor(
    @Assisted private val appContext: Context,
    @Assisted private val workerParams: WorkerParameters
    // Có thể inject thêm NetworkRepository hoặc Database tại đây!
) : CoroutineWorker(appContext, workerParams) {

    companion object {
        const val KEY_IMAGE_PATH = "key_image_path"
        const val KEY_OUTPUT_PATH = "key_output_path"
        const val PROGRESS_PERCENT = "progress_percent"
    }

    override suspend fun doWork(): Result {
        val inputPath = inputData.getString(KEY_IMAGE_PATH)
            ?: return Result.failure(workDataOf("error" to "Missing input image path"))

        return try {
            // Giả lập tiến trình nén ảnh nặng qua 5 giai đoạn
            for (progress in 20..100 step 20) {
                // Kiểm tra xem hệ điều hành có yêu cầu dừng worker không (vd: mất kết nối mạng)
                if (isStopped) {
                    cleanUpTemporaryFiles()
                    return Result.retry()
                }

                delay(600) // Giả lập xử lý nén CPU-intensive

                // Gửi tiến độ thời gian thực về WorkManager SQLite
                setProgress(workDataOf(PROGRESS_PERCENT to progress))
            }

            val fakeCompressedPath = "/data/user/0/com.example/files/compressed_${System.currentTimeMillis()}.jpg"

            // Thành công: Trả về kết quả đầu ra
            Result.success(workDataOf(KEY_OUTPUT_PATH to fakeCompressedPath))
        } catch (e: Exception) {
            if (runAttemptCount < 3) {
                Result.retry() // Tự động thử lại với Exponential Backoff
            } else {
                Result.failure(workDataOf("error" to (e.message ?: "Unknown processing failure")))
            }
        }
    }

    private fun cleanUpTemporaryFiles() {
        // Xóa sạch file nháp khi worker bị hủy
    }
}
```

### Bước 3: Triển khai ViewModel quản lý Enqueue và Observe Flow

Tạo tệp `ui/ImageProcessingViewModel.kt`:

```kotlin
package com.example.workmanager.ui

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import androidx.work.*
import com.example.workmanager.worker.ImageProcessingWorker
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import java.util.UUID
import javax.inject.Inject

sealed interface ProcessingUiState {
    data object Idle : ProcessingUiState
    data class Processing(val progress: Int) : ProcessingUiState
    data class Success(val outputPath: String) : ProcessingUiState
    data class Failed(val reason: String) : ProcessingUiState
}

@HiltViewModel
class ImageProcessingViewModel @Inject constructor(
    private val workManager: WorkManager
) : ViewModel() {

    private val _currentWorkId = MutableStateFlow<UUID?>(null)

    // Chuyển hóa WorkManager Flow thành UI State phản ứng
    val uiState: StateFlow<ProcessingUiState> = _currentWorkId
        .flatMapLatest { workId ->
            if (workId == null) {
                flowOf(ProcessingUiState.Idle)
            } else {
                workManager.getWorkInfoByIdFlow(workId).map { workInfo ->
                    mapWorkInfoToState(workInfo)
                }
            }
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = ProcessingUiState.Idle
        )

    fun startCompressingImage(rawImagePath: String) {
        // 1. Ràng buộc hệ điều hành
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .setRequiresBatteryNotLow(true)
            .build()

        // 2. Dữ liệu đầu vào
        val inputData = workDataOf(ImageProcessingWorker.KEY_IMAGE_PATH to rawImagePath)

        // 3. Khởi tạo OneTimeWorkRequest
        val workRequest = OneTimeWorkRequestBuilder<ImageProcessingWorker>()
            .setConstraints(constraints)
            .setInputData(inputData)
            .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, WorkRequest.MIN_BACKOFF_MILLIS, java.util.concurrent.TimeUnit.MILLISECONDS)
            .addTag("TAG_IMAGE_PROCESSOR")
            .build()

        // 4. Enqueue dưới dạng Unique Work để chống click đúp
        val uniqueWorkName = "UNIQUE_IMAGE_COMPRESS"
        workManager.enqueueUniqueWork(
            uniqueWorkName,
            ExistingWorkPolicy.REPLACE,
            workRequest
        )

        _currentWorkId.value = workRequest.id
    }

    fun cancelCurrentWork() {
        _currentWorkId.value?.let { id ->
            workManager.cancelWorkById(id)
        }
    }

    private fun mapWorkInfoToState(workInfo: WorkInfo?): ProcessingUiState {
        if (workInfo == null) return ProcessingUiState.Idle

        return when (workInfo.state) {
            WorkInfo.State.ENQUEUED -> ProcessingUiState.Processing(progress = 0)
            WorkInfo.State.RUNNING -> {
                val progress = workInfo.progress.getInt(ImageProcessingWorker.PROGRESS_PERCENT, 0)
                ProcessingUiState.Processing(progress = progress)
            }
            WorkInfo.State.SUCCEEDED -> {
                val output = workInfo.outputData.getString(ImageProcessingWorker.KEY_OUTPUT_PATH).orEmpty()
                ProcessingUiState.Success(output)
            }
            WorkInfo.State.FAILED -> {
                val err = workInfo.outputData.getString("error") ?: "Quá trình xử lý thất bại"
                ProcessingUiState.Failed(err)
            }
            WorkInfo.State.CANCELLED -> {
                ProcessingUiState.Failed("Tác vụ đã bị người dùng hủy")
            }
            WorkInfo.State.BLOCKED -> ProcessingUiState.Processing(progress = 0)
        }
    }
}
```

### Bước 4: Jetpack Compose UI hiển thị Tiến độ qua Flow

Tạo tệp `ui/ImageProcessingScreen.kt`:

```kotlin
package com.example.workmanager.ui

import androidx.compose.animation.AnimatedVisibility
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import androidx.lifecycle.compose.collectAsStateWithLifecycle

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ImageProcessingScreen(
    viewModel: ImageProcessingViewModel = hiltViewModel()
) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()

    Scaffold(
        topBar = {
            TopAppBar(title = { Text("WorkManager + Flow Tiến độ") })
        }
    ) { padding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
                .padding(24.dp),
            verticalArrangement = Arrangement.Center,
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            when (val current = state) {
                is ProcessingUiState.Idle -> {
                    Text("Chưa có tác vụ nào đang chạy", style = MaterialTheme.typography.bodyLarge)
                    Spacer(modifier = Modifier.height(24.dp))
                    Button(
                        onClick = { viewModel.startCompressingImage("/storage/emulated/0/DCIM/photo.raw") }
                    ) {
                        Text("Bắt đầu nén ảnh nền (Guaranteed Work)")
                    }
                }

                is ProcessingUiState.Processing -> {
                    Text("Đang xử lý ảnh ngầm: ${current.progress}%", style = MaterialTheme.typography.titleMedium)
                    Spacer(modifier = Modifier.height(16.dp))
                    LinearProgressIndicator(
                        progress = { current.progress / 100f },
                        modifier = Modifier
                            .fillMaxWidth()
                            .height(8.dp)
                    )
                    Spacer(modifier = Modifier.height(24.dp))
                    OutlinedButton(
                        onClick = { viewModel.cancelCurrentWork() },
                        colors = ButtonDefaults.outlinedButtonColors(contentColor = MaterialTheme.colorScheme.error)
                    ) {
                        Text("Hủy tác vụ")
                    }
                }

                is ProcessingUiState.Success -> {
                    Text(
                        "✅ Nén ảnh thành công!",
                        style = MaterialTheme.typography.titleLarge,
                        color = MaterialTheme.colorScheme.primary
                    )
                    Spacer(modifier = Modifier.height(8.dp))
                    Text(
                        "File đầu ra: ${current.outputPath}",
                        style = MaterialTheme.typography.bodySmall
                    )
                    Spacer(modifier = Modifier.height(24.dp))
                    Button(
                        onClick = { viewModel.startCompressingImage("/storage/emulated/0/DCIM/photo2.raw") }
                    ) {
                        Text("Nén ảnh khác")
                    }
                }

                is ProcessingUiState.Failed -> {
                    Text(
                        "❌ Lỗi: ${current.reason}",
                        style = MaterialTheme.typography.titleMedium,
                        color = MaterialTheme.colorScheme.error
                    )
                    Spacer(modifier = Modifier.height(24.dp))
                    Button(
                        onClick = { viewModel.startCompressingImage("/storage/emulated/0/DCIM/photo.raw") }
                    ) {
                        Text("Thử lại")
                    }
                }
            }
        }
    }
}
```

### Bước 5: Unit Test CoroutineWorker với `WorkManagerTestInitHelper`

Tạo tệp test `ImageProcessingWorkerTest.kt`:

```kotlin
package com.example.workmanager

import android.content.Context
import androidx.test.core.app.ApplicationProvider
import androidx.work.ListenableWorker
import androidx.work.testing.TestListenableWorkerBuilder
import androidx.work.workDataOf
import com.example.workmanager.worker.ImageProcessingWorker
import kotlinx.coroutines.runBlocking
import org.junit.Assert.assertEquals
import org.junit.Assert.assertTrue
import org.junit.Before
import org.junit.Test
import org.junit.runner.RunWith
import org.robolectric.RobolectricTestRunner

@RunWith(RobolectricTestRunner::class)
class ImageProcessingWorkerTest {

    private lateinit var context: Context

    @Before
    fun setUp() {
        context = ApplicationProvider.getApplicationContext()
    }

    @Test
    fun `worker with missing input path should return Result failure`() = runBlocking {
        // Given: Không truyền KEY_IMAGE_PATH
        val worker = TestListenableWorkerBuilder<ImageProcessingWorker>(context)
            .build()

        // When
        val result = worker.doWork()

        // Then
        assertTrue(result is ListenableWorker.Result.Failure)
    }

    @Test
    fun `worker with valid input path should complete successfully and return output path`() = runBlocking {
        // Given
        val input = workDataOf(ImageProcessingWorker.KEY_IMAGE_PATH to "/path/to/img.png")
        val worker = TestListenableWorkerBuilder<ImageProcessingWorker>(context)
            .setInputData(input)
            .build()

        // When
        val result = worker.doWork()

        // Then
        assertTrue(result is ListenableWorker.Result.Success)
        val outputData = (result as ListenableWorker.Result.Success).outputData
        val outputPath = outputData.getString(ImageProcessingWorker.KEY_OUTPUT_PATH)
        assertTrue(outputPath?.contains("compressed_") == true)
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior / Staff Android Architect

#### Câu hỏi 1: Khi nào nên sử dụng Kotlin Coroutines thuần túy và khi nào bắt buộc phải dùng WorkManager?
**Trả lời:**
- Quy tắc vàng dựa trên **Vòng đời (Lifecycle) và Tính bền vững (Durability)**:
  - Nếu tác vụ **phụ thuộc vào màn hình hiện tại** (như hiển thị Search suggestions, tải danh sách bạn bè khi mở tab): Sử dụng `viewModelScope.launch`. Khi người dùng thoát màn hình hoặc đóng app, tác vụ bị hủy ngay lập tức, tiết kiệm tài nguyên.
  - Nếu tác vụ **bắt buộc phải hoàn thành bất kể người dùng làm gì** (như hoàn tất upload video 200MB, gửi hóa đơn thanh toán lên hệ thống, sao lưu nhật ký chat): Bắt buộc phải sử dụng `WorkManager`. Hệ thống đảm bảo sống sót qua process kill và thiết bị khởi động lại.

#### Câu hỏi 2: Tại sao `PeriodicWorkRequest` lại có thời gian chu kỳ tối thiểu là 15 phút (không thể đặt 5 phút)?
**Trả lời:**
- Đây là giới hạn bảo vệ pin và CPU được thiết kế có chủ đích bởi Google Android OS Team. Việc đánh thức CPU, bật Radio (4G/5G/WiFi) mỗi 5 phút sẽ khiến thiết bị không bao giờ bước vào trạng thái ngủ sâu (Deep Sleep / Doze Mode), làm tụt pin nhanh chóng.
- Nếu dự án thực sự cần chu kỳ dưới 15 phút (ví dụ: VoIP Heartbeat hoặc BLE Tracking), bạn phải sử dụng **Foreground Service** (với notification thường trực) hoặc nhận **FCM High-Priority Push Notification** từ server để đánh thức app.

#### Câu hỏi 3: Làm thế nào để nối chuỗi (Work Chaining) nhiều Worker và truyền dữ liệu tuần tự từ Worker này sang Worker tiếp theo?
**Trả lời:**
- Sử dụng API `beginWith().then()` của WorkManager:
  ```kotlin
  workManager.beginWith(downloadWorkerRequest)
      .then(filterWorkerRequest)
      .then(uploadWorkerRequest)
      .enqueue()
  ```
- Dữ liệu trả về qua `Result.success(outputData)` của `downloadWorkerRequest` sẽ tự động trở thành `inputData` của `filterWorkerRequest` thông qua lớp trung gian `InputMerger` (mặc định là `OverwritingInputMerger`).

### 6.2 Bảng gỡ rối các lỗi thực tế (Troubleshooting Matrix)

| Vấn đề / Lỗi thực tế | Nguyên nhân gốc rễ (Root Cause) | Giải pháp triệt để (Solution) |
|---|---|---|
| **Worker không bao giờ chạy (Mãi ở trạng thái ENQUEUED)** | Đặt ràng buộc (`Constraints`) quá khắt khe (ví dụ: `setRequiresCharging(true)` hoặc `NetworkType.UNMETERED` trong khi thiết bị đang dùng 4G và không cắm sạc) | Kiểm tra lại các điều kiện Constraints; hạ xuống `NetworkType.CONNECTED` nếu không nhất thiết cần WiFi. |
| **Crash: `IllegalStateException: WorkManager is not initialized properly`** | Khởi tạo WorkManager bằng `getInstance(context)` nhưng chưa tắt trình khởi tạo mặc định trong Manifest khi dùng Custom Configuration | Thêm thẻ `<provider>` gỡ bỏ `WorkManagerInitializer` mặc định trong `AndroidManifest.xml`. |
| **Worker bị chạy lặp lại nhiều lần (Duplicate Execution)** | Sử dụng `workManager.enqueue()` thông thường mỗi khi người dùng click, tạo ra nhiều instance cùng lúc | Thay bằng `workManager.enqueueUniqueWork("unique_tag", ExistingWorkPolicy.KEEP, request)`. |
| **Data vượt quá 10KB bị crash `IllegalStateException: Data cannot occupy more than 10240 bytes`** | Truyền Bitmap, Byte Array hoặc String JSON khổng lồ vào `workDataOf()` | Lưu dữ liệu lớn vào Room Database hoặc Internal File Storage, chỉ truyền File Path hoặc Record ID qua `Data`. |

---

## 7. Tổng kết

WorkManager kết hợp Kotlin Flow và Jetpack Compose cung cấp một kiến trúc xử lý tác vụ nền **Bền vững - Phản ứng - Tiết kiệm năng lượng**. Với sự bảo vệ của SQLite persistence và cơ chế quan sát thời gian thực `getWorkInfoByIdFlow()`, ứng dụng của bạn luôn đảm bảo tính toàn vẹn dữ liệu ở cấp độ doanh nghiệp cao nhất.

*Bài tiếp theo: [Bài 21 — Room + Flow: Reactive Database, InvalidationTracker & Migrations](21-room-flow.md)*
