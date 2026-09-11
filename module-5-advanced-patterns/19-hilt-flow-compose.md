# Bài 19 — Hilt + Flow + Compose: Dependency Injection End-to-End

> **Module:** 5 — Advanced Patterns & Integration  
> **Mức độ:** Senior / Staff Android Architect  
> **Prerequisites:** Module 1 (Coroutines & Flow), Module 2 (Architecture), Module 3 (Compose Foundation)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

**Dependency Injection (DI - Tiêm phụ thuộc)** là một mẫu thiết kế phần mềm thực hiện nguyên lý **Đảo ngược điều khiển (Inversion of Control - IoC)** và nguyên tắc **Dependency Inversion (chữ D trong SOLID)**. Thay vì để một lớp tự khởi tạo các đối tượng mà nó phụ thuộc (Dependencies), các phụ thuộc này được cung cấp từ bên ngoài thông qua một bộ quản lý tập trung.

**Dagger Hilt** là thư viện Dependency Injection tiêu chuẩn chính thức của Google dành cho nền tảng Android. Hilt được xây dựng bên trên Dagger 2, loại bỏ hầu hết các đoạn mã lặp lại (boilerplate) của Dagger truyền thống bằng cách tích hợp trực tiếp vào **Vòng đời chuẩn của hệ điều hành Android (Android Lifecycle)**.

```
                    ┌────────────────────────────┐
                    │     SingletonComponent     │ (Application Lifecycle)
                    └─────────────┬──────────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │ ActivityRetainedComponent  │ (Survives Config Changes)
                    └─────────────┬──────────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │    ViewModelComponent      │ (ViewModel Lifecycle)
                    └─────────────┬──────────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │     ActivityComponent      │ (Activity Lifecycle)
                    └─────────────┬──────────────┘
                                  │
           ┌──────────────────────┴──────────────────────┐
           ▼                                             ▼
┌──────────────────────┐                      ┌──────────────────────┐
│  FragmentComponent   │                      │    ViewComponent     │
└──────────────────────┘                      └──────────────────────┘
```

### Các thuật ngữ cốt lõi:
- **Component (Thành phần chứa phụ thuộc):** Một đồ thị đối tượng (Object Graph) do Dagger sinh ra, chịu trách nhiệm lưu trữ và quản lý vòng đời của các dependency.
- **Scope (Phạm vi sống):** Annotation xác định vòng đời tồn tại của một instance đối tượng trong bộ nhớ. Nếu một đối tượng được đánh dấu bằng `@Singleton`, Dagger sẽ chỉ tạo duy nhất 1 instance và tái sử dụng nó trong suốt vòng đời của Application.
- **`@HiltAndroidApp`:** Annotation kích hoạt quá trình sinh mã của Hilt tại tầng `Application`, khởi tạo `SingletonComponent`.
- **`@AndroidEntryPoint`:** Annotation đánh dấu các điểm vào của Android (Activity, Fragment, View, Service) để Hilt có thể inject các dependency vào chúng.
- **`@Inject`:** Được sử dụng tại Constructor (`@Inject constructor(...)`) để báo cho Dagger biết cách tạo ra một đối tượng, hoặc tại Field (`@Inject lateinit var ...`) để yêu cầu Dagger gán giá trị.
- **`@Module` & `@InstallIn`:** Nơi định nghĩa cách cung cấp các đối tượng mà ta không thể thêm `@Inject constructor` (ví dụ: các lớp từ thư viện bên ngoài như Retrofit, RoomDatabase, OkHttpClient).
- **`@Provides` vs `@Binds`:** Hai phương thức cung cấp dependency trong Hilt Module:
  - `@Provides`: Dùng cho các đối tượng cần logic khởi tạo phức tạp hoặc từ bên thứ ba.
  - `@Binds`: Dùng để liên kết trực tiếp một Interface với một Implementation cụ thể.
- **`@Qualifier`:** Annotation tùy chỉnh dùng để phân biệt các instance có cùng kiểu dữ liệu (ví dụ: phân biệt giữa IO CoroutineDispatcher và Default CoroutineDispatcher).

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Cơ chế sinh mã lúc biên dịch (Compile-Time Code Generation)
Khác với các giải pháp DI dựa trên Reflection (như Spring) hoặc Service Locator giải quyết ở runtime (như Koin), Hilt hoạt động **100% ở thời điểm biên dịch (Compile-Time)** thông qua KSP (Kotlin Symbol Processing) hoặc KAPT.

Khi bạn thêm annotation `@HiltAndroidApp` và `@AndroidEntryPoint`, Hilt thực hiện các bước sau:
1. **Tạo lớp cơ sở trung gian (Hilt Base Classes):**
   - Hilt tạo ra một lớp bytecode giả lập như `Hilt_MainActivity` kế thừa từ `ComponentActivity`.
   - Plugin `dagger.hilt.android.plugin` sử dụng kỹ thuật **Bytecode Transformation (ASM)** để đổi lớp cha của `MainActivity` từ `ComponentActivity` sang `Hilt_MainActivity`.
2. **Tạo Factory và Provider Classes:**
   - Với mỗi lớp có `@Inject constructor`, Dagger tạo ra một lớp tương ứng: `UserRepository_Impl_Factory.java`. Lớp này chịu trách nhiệm gọi `new UserRepository_Impl(...)` với các tham số được lấy từ Dagger Graph.
3. **Xác minh toàn bộ đồ thị phụ thuộc (Graph Validation):**
   - Dagger duyệt qua toàn bộ cây phụ thuộc. Nếu phát hiện bất kỳ phụ thuộc nào bị thiếu (Missing Binding), vòng lặp phụ thuộc (Cyclic Dependency), hoặc xung đột Scope (Scope Mismatch), quá trình biên dịch sẽ **ngay lập tức thất bại (Build Failure)** với thông báo lỗi chi tiết.

### 2.2 So sánh Bytecode: `@Binds` vs `@Provides`
Đây là một trong những điểm kiến trúc quan trọng nhất chứng minh đẳng cấp của một Senior Architect:

#### Cách viết thông thường với `@Provides`:
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DataModule {
    @Provides
    @Singleton
    fun provideUserRepository(impl: UserRepositoryImpl): UserRepository {
        return impl
    }
}
```
**Bytecode sinh ra:** Dagger tạo một class trung gian `DataModule_ProvideUserRepositoryFactory.java`. Mỗi lần resolve, Dagger phải khởi tạo Factory này và gọi phương thức ủy quyền, gây tốn bộ nhớ heap và thêm phương thức vào Dex file.

#### Cách viết tối ưu với `@Binds`:
```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class DataModule {
    @Binds
    @Singleton
    abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
}
```
**Bytecode sinh ra:** Vì là `abstract class` và `abstract fun`, Dagger **hoàn toàn không sinh ra bất kỳ class Factory trung gian nào**! Dagger cấu hình trực tiếp trong bảng chỉ mục nội bộ rằng bất cứ khi nào ai cần `UserRepository`, nó sẽ chuyển thẳng đến `UserRepositoryImpl_Factory`. Không tốn heap, không tốn phương thức Dex, hiệu năng khởi động ứng dụng đạt mức tối đa.

### 2.3 Cây phân cấp Hilt Component & Bảng ánh xạ Scope

```
Component                       Scope                       Vòng đời tồn tại
────────────────────────────────────────────────────────────────────────────────────────────
SingletonComponent              @Singleton                  Application khởi chạy -> App bị kill
  └── ActivityRetainedComponent @ActivityRetainedScoped     Activity tạo -> Survives Configuration Change (Xoay màn hình)
        └── ViewModelComponent  @ViewModelScoped            ViewModel tạo -> ViewModel.onCleared()
              └── ActivityComponent @ActivityScoped         Activity onCreate -> Activity onDestroy
                    ├── FragmentComponent @FragmentScoped   Fragment onAttach -> Fragment onDestroy
                    └── ViewComponent     @ViewScoped       View attach to Window -> Detach
```

**Quy tắc đồ thị (Graph Lookup Rule):**
Một component con có quyền truy cập vào tất cả các binding của component cha của nó. Tuy nhiên, component cha **không thể** truy cập các binding của component con.
*Ví dụ:* Một class được đánh dấu `@ViewModelScoped` có thể inject các dependency từ `SingletonComponent` (như Retrofit, Database), nhưng một `@Singleton` Repository không bao giờ được phép inject một `@ViewModelScoped` hay `@ActivityScoped` class.

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Nỗi đau của các phương pháp cũ

#### 1. Tự viết thủ công (Manual Dependency Injection):
```kotlin
// ❌ ANTI-PATTERN: Thủ công tạo Dependency Tree
class MyApplication : Application() {
    val database by lazy { AppDatabase.create(this) }
    val apiService by lazy { RetrofitClient.create() }
    val userRepository by lazy { UserRepositoryImpl(apiService, database.userDao()) }
}
```
- Phải truyền Context lung tung, rất dễ gây ra **Memory Leak**.
- Constructor bùng nổ (Constructor Explosion): Khi một Repository cần thêm một phụ thuộc mới, bạn phải sửa lại hàng chục ViewModel đang sử dụng nó.

#### 2. Thư viện Koin (Service Locator giải quyết ở Runtime):
```kotlin
// Koin: Trông rất ngắn gọn nhưng tiềm ẩn rủi ro production
val appModule = module {
    single<UserRepository> { UserRepositoryImpl(get(), get()) }
}
```
- **Không có Compile-Time Verification:** Nếu lập trình viên quên khai báo một dependency hoặc truyền sai thứ tự tham số `get()`, ứng dụng vẫn build thành công 100%. Khi người dùng click vào màn hình đó trên thiết bị thật, ứng dụng sẽ lập tức crash với `NoBeanDefFoundException`!
- **Reflection Overhead:** Koin phải resolve phụ thuộc thông qua lookup runtime, làm chậm thời gian render màn hình đầu tiên (Cold Start).

### 3.2 Giá trị vượt trội của Hilt
- **An toàn tuyệt đối lúc biên dịch (Compile-Time Safety):** Mọi thiếu sót về binding đều khiến build fail ngay lập tức tại máy dev hoặc CI/CD server.
- **Tích hợp sâu sắc với Android Architecture Components:** Hilt tự động xử lý việc inject vào ViewModel (`@HiltViewModel`), Compose (`hiltViewModel()`), WorkManager (`@HiltWorker`), và SavedStateHandle.
- **Tách biệt rạch ròi môi trường Testing:** Hilt cho phép ghi đè (override) module cực kỳ thanh lịch bằng `@TestInstallIn` mà không cần sửa một dòng code nào trong production.

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Do's and Don'ts từ Google Android Architecture

#### DO:
1. **Luôn sử dụng `@Binds` thay vì `@Provides` bất cứ khi nào có thể:** Chỉ dùng `@Provides` khi khởi tạo các thư viện bên ngoài (Retrofit, Room, OkHttp, DataStore) hoặc khi cần Builder logic.
2. **Inject Coroutine Dispatchers thông qua Custom Qualifiers:** Không bao giờ hardcode `Dispatchers.IO` hay `Dispatchers.Default` trong ViewModel hay Repository. Luôn inject chúng qua Qualifiers để dễ dàng thay thế bằng `StandardTestDispatcher` trong Unit Test.
3. **Giữ ViewModel độc lập với Android Framework:** `@HiltViewModel` chỉ nên nhận các Interface (UseCases, Repositories, Dispatchers), không bao giờ inject `Activity`, `Context`, hay `View` vào ViewModel.
4. **Sử dụng `@ViewModelScoped` một cách hợp lý:** Chỉ gắn `@ViewModelScoped` khi đối tượng đó thực sự cần sống cùng vòng đời với một ViewModel cụ thể (ví dụ: một StateHolder nội bộ). Các Repository truy cập mạng hoặc database phải gắn `@Singleton`.

#### DON'T:
1. **KHÔNG lạm dụng `@Singleton` cho mọi thứ (Over-scoping):** Việc gắn `@Singleton` vào mọi class sẽ giữ chúng sống vĩnh viễn trên bộ nhớ Heap của Application, dẫn đến việc bộ thu gom rác (Garbage Collector) không thể giải phóng tài nguyên, làm tăng nguy cơ `OutOfMemoryError`.
2. **KHÔNG inject `@ActivityScoped` vào một `@Singleton`:** Đây là lỗi **Scope Mismatch** kinh điển. Nó sẽ giữ tham chiếu tới Activity đã bị hủy sau khi xoay màn hình, gây rò rỉ bộ nhớ (Memory Leak) nghiêm trọng.
3. **KHÔNG tạo nhiều Hilt Component tùy biến nếu không cần thiết:** Tận dụng tối đa các Component dựng sẵn của Hilt.

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Chúng ta sẽ xây dựng một kiến trúc chuẩn Hilt kết hợp Kotlin Flow và Jetpack Compose từ tầng Data, Domain đến UI, kèm bộ kiểm thử Integration Test hoàn chỉnh.

### Bước 1: Khởi tạo Hilt trong Application & Custom Qualifiers

Tạo tệp `di/CoroutinesQualifiers.kt`:

```kotlin
package com.example.hiltflow.di

import javax.inject.Qualifier

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class IoDispatcher

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class DefaultDispatcher

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class MainDispatcher
```

Tạo tệp `di/DispatchersModule.kt`:

```kotlin
package com.example.hiltflow.di

import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import kotlinx.coroutines.CoroutineDispatcher
import kotlinx.coroutines.Dispatchers
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object DispatchersModule {

    @Provides
    @Singleton
    @IoDispatcher
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO

    @Provides
    @Singleton
    @DefaultDispatcher
    fun provideDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default

    @Provides
    @Singleton
    @MainDispatcher
    fun provideMainDispatcher(): CoroutineDispatcher = Dispatchers.Main
}
```

Khai báo Application trong `App.kt` và thêm vào `AndroidManifest.xml`:

```kotlin
package com.example.hiltflow

import android.app.Application
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp
class MainApplication : Application()
```

### Bước 2: Tầng Data & Domain với `@Binds` và `@Singleton`

Tạo Interface và Data Model trong tệp `domain/UserModels.kt`:

```kotlin
package com.example.hiltflow.domain

import kotlinx.coroutines.flow.Flow

data class UserProfile(
    val id: String,
    val name: String,
    val email: String,
    val isVip: Boolean
)

interface UserRepository {
    fun getUserProfileFlow(userId: String): Flow<UserProfile>
    suspend fun updateVipStatus(userId: String, isVip: Boolean)
}
```

Triển khai Repository trong tệp `data/UserRepositoryImpl.kt`:

```kotlin
package com.example.hiltflow.data

import com.example.hiltflow.di.IoDispatcher
import com.example.hiltflow.domain.UserProfile
import com.example.hiltflow.domain.UserRepository
import kotlinx.coroutines.CoroutineDispatcher
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.flowOn
import kotlinx.coroutines.withContext
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class UserRepositoryImpl @Inject constructor(
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher
    // Trong thực tế, inject Room Dao hoặc Retrofit ApiService tại đây
) : UserRepository {

    // Giả lập Local Cache phản ứng bằng MutableStateFlow
    private val _userState = MutableStateFlow(
        UserProfile(
            id = "user_99",
            name = "Alex Nguyen",
            email = "alex.nguyen@enterprise.com",
            isVip = false
        )
    )

    override fun getUserProfileFlow(userId: String): Flow<UserProfile> {
        return _userState.asStateFlow().flowOn(ioDispatcher)
    }

    override suspend fun updateVipStatus(userId: String, isVip: Boolean) {
        withContext(ioDispatcher) {
            delay(400) // Giả lập I/O latency
            val current = _userState.value
            if (current.id == userId) {
                _userState.value = current.copy(isVip = isVip)
            }
        }
    }
}
```

Khai báo Module liên kết Interface với Implementation qua `@Binds` trong `di/RepositoryModule.kt`:

```kotlin
package com.example.hiltflow.di

import com.example.hiltflow.data.UserRepositoryImpl
import com.example.hiltflow.domain.UserRepository
import dagger.Binds
import dagger.Module
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    // Tối ưu hóa bytecode: Abstract function không sinh class trung gian
    @Binds
    @Singleton
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl
    ): UserRepository
}
```

### Bước 3: Triển khai `@HiltViewModel`

Tạo tệp `ui/ProfileViewModel.kt`:

```kotlin
package com.example.hiltflow.ui

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.hiltflow.domain.UserProfile
import com.example.hiltflow.domain.UserRepository
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asSharedFlow
import kotlinx.coroutines.flow.catch
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch
import javax.inject.Inject

sealed interface ProfileUiState {
    data object Loading : ProfileUiState
    data class Success(val profile: UserProfile) : ProfileUiState
    data class Error(val message: String) : ProfileUiState
}

sealed interface ProfileUiEffect {
    data class ShowSnackbar(val text: String) : ProfileUiEffect
}

@HiltViewModel
class ProfileViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {

    // One-time effects
    private val _effect = MutableSharedFlow<ProfileUiEffect>()
    val effect = _effect.asSharedFlow()

    // Chuyển hóa Flow từ Repository thành StateFlow sẵn sàng cho Compose
    val uiState: StateFlow<ProfileUiState> = userRepository.getUserProfileFlow("user_99")
        .map<UserProfile, ProfileUiState> { profile ->
            ProfileUiState.Success(profile)
        }
        .catch { throwable ->
            emit(ProfileUiState.Error(throwable.message ?: "Đã xảy ra lỗi không xác định"))
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = ProfileUiState.Loading
        )

    fun toggleVipStatus() {
        val currentState = uiState.value
        if (currentState is ProfileUiState.Success) {
            val newStatus = !currentState.profile.isVip
            viewModelScope.launch {
                userRepository.updateVipStatus(currentState.profile.id, newStatus)
                val msg = if (newStatus) "Chúc mừng! Bạn đã nâng cấp lên VIP." else "Đã hủy gói VIP."
                _effect.emit(ProfileUiEffect.ShowSnackbar(msg))
            }
        }
    }
}
```

### Bước 4: Tích hợp với Jetpack Compose & Navigation

Tạo Activity chứa `@AndroidEntryPoint` trong `ui/MainActivity.kt`:

```kotlin
package com.example.hiltflow.ui

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import dagger.hilt.android.AndroidEntryPoint

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                Surface(modifier = Modifier.fillMaxSize()) {
                    ProfileRoute()
                }
            }
        }
    }
}

@Composable
fun ProfileRoute(
    // hiltViewModel() tự động tìm kiếm ViewModelFactory được gắn với Hilt Component
    viewModel: ProfileViewModel = hiltViewModel()
) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }

    LaunchedEffect(Unit) {
        viewModel.effect.collect { effect ->
            when (effect) {
                is ProfileUiEffect.ShowSnackbar -> {
                    snackbarHostState.showSnackbar(effect.text)
                }
            }
        }
    }

    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { padding ->
        Box(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding),
            contentAlignment = Alignment.Center
        ) {
            when (val current = state) {
                is ProfileUiState.Loading -> {
                    CircularProgressIndicator()
                }
                is ProfileUiState.Error -> {
                    Text(text = "Lỗi: ${current.message}", color = MaterialTheme.colorScheme.error)
                }
                is ProfileUiState.Success -> {
                    ProfileContent(
                        profile = current.profile,
                        onToggleVip = viewModel::toggleVipStatus
                    )
                }
            }
        }
    }
}

@Composable
fun ProfileContent(
    profile: com.example.hiltflow.domain.UserProfile,
    onToggleVip: () -> Unit
) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(24.dp),
        elevation = CardDefaults.cardElevation(defaultElevation = 6.dp)
    ) {
        Column(
            modifier = Modifier.padding(24.dp),
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            Text(text = profile.name, style = MaterialTheme.typography.headlineMedium)
            Spacer(modifier = Modifier.height(8.dp))
            Text(text = profile.email, style = MaterialTheme.typography.bodyMedium)
            Spacer(modifier = Modifier.height(16.dp))

            Surface(
                color = if (profile.isVip) MaterialTheme.colorScheme.primaryContainer else MaterialTheme.colorScheme.surfaceVariant,
                shape = MaterialTheme.shapes.medium
            ) {
                Text(
                    text = if (profile.isVip) "🌟 HỘI VIÊN VIP" else "Tài khoản Thường",
                    modifier = Modifier.padding(horizontal = 16.dp, vertical = 8.dp),
                    style = MaterialTheme.typography.labelLarge
                )
            }

            Spacer(modifier = Modifier.height(24.dp))
            Button(onClick = onToggleVip) {
                Text(if (profile.isVip) "Hạ cấp về Thường" else "Nâng cấp lên VIP")
            }
        }
    }
}
```

### Bước 5: Viết Hilt Integration Test với `@TestInstallIn`

Tạo tệp test `ProfileIntegrationTest.kt` trong `androidTest`:

```kotlin
package com.example.hiltflow

import androidx.compose.ui.test.assertIsDisplayed
import androidx.compose.ui.test.junit4.createAndroidComposeRule
import androidx.compose.ui.test.onNodeWithText
import androidx.compose.ui.test.performClick
import com.example.hiltflow.di.RepositoryModule
import com.example.hiltflow.domain.UserProfile
import com.example.hiltflow.domain.UserRepository
import com.example.hiltflow.ui.MainActivity
import dagger.Binds
import dagger.Module
import dagger.hilt.android.testing.HiltAndroidRule
import dagger.hilt.android.testing.HiltAndroidTest
import dagger.hilt.android.testing.UninstallModules
import dagger.hilt.components.SingletonComponent
import dagger.hilt.testing.TestInstallIn
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableStateFlow
import org.junit.Before
import org.junit.Rule
import org.junit.Test
import javax.inject.Inject
import javax.inject.Singleton

// Fake Repository phục vụ Test tự động
@Singleton
class FakeUserRepository @Inject constructor() : UserRepository {
    private val state = MutableStateFlow(
        UserProfile("test_id", "Test User", "test@domain.com", isVip = false)
    )

    override fun getUserProfileFlow(userId: String): Flow<UserProfile> = state

    override suspend fun updateVipStatus(userId: String, isVip: Boolean) {
        state.value = state.value.copy(isVip = isVip)
    }
}

// Thay thế hoàn toàn RepositoryModule gốc trong môi trường Test
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [RepositoryModule::class]
)
abstract class FakeRepositoryModule {
    @Binds
    @Singleton
    abstract fun bindFakeUserRepository(impl: FakeUserRepository): UserRepository
}

@HiltAndroidTest
class ProfileIntegrationTest {

    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val composeTestRule = createAndroidComposeRule<MainActivity>()

    @Before
    fun init() {
        hiltRule.inject()
    }

    @Test
    fun testVipToggleUpdatesUiCorrectly() {
        // Kiểm tra hiển thị ban đầu
        composeTestRule.onNodeWithText("Test User").assertIsDisplayed()
        composeTestRule.onNodeWithText("Tài khoản Thường").assertIsDisplayed()

        // Nhấn nút nâng cấp
        composeTestRule.onNodeWithText("Nâng cấp lên VIP").performClick()

        // Kiểm tra UI lập tức cập nhật trạng thái mới
        composeTestRule.onNodeWithText("🌟 HỘI VIÊN VIP").assertIsDisplayed()
        composeTestRule.onNodeWithText("Hạ cấp về Thường").assertIsDisplayed()
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior / Staff Android Architect

#### Câu hỏi 1: `@Binds` khác `@Provides` như thế nào về mặt Bytecode và Hiệu năng Khởi động ứng dụng (Cold Start)?
**Trả lời:**
- Khi sử dụng `@Provides`, Dagger buộc phải sinh ra một lớp Factory riêng biệt (`MyModule_ProvideSomethingFactory.class`). Trong một ứng dụng lớn với hàng ngàn dependency, việc sinh ra hàng ngàn lớp Factory làm tăng dung lượng file DEX (Dex Method Count), làm nặng quá trình nạp class (Class Loading) vào RAM khi ứng dụng khởi động.
- Khi sử dụng `@Binds`, phương thức là `abstract` và không chứa code thực thi. Dagger tận dụng thông tin này ở compile-time để gán thẳng liên kết trong dependency graph. **Không có bất kỳ class trung gian nào được sinh ra**. Do đó, `@Binds` giúp giảm kích thước APK và tăng tốc độ Cold Start đáng kể.

#### Câu hỏi 2: Làm thế nào để tổ chức Hilt trong dự án Multi-Module Architecture quy mô lớn?
**Trả lời:**
- Áp dụng cấu trúc **API / Impl Separation**:
  - `feature:profile:api`: Chứa Model, Interface `ProfileRepository`, `ProfileNavigationRoute`. Module này không cài đặt Hilt Module.
  - `feature:profile:impl`: Chứa `ProfileRepositoryImpl`, `ProfileViewModel`, `ProfileScreen`. Cài đặt Hilt Module với `@InstallIn(SingletonComponent::class)` để `@Binds` interface từ module `:api` sang implementation ở module `:impl`.
  - Ứng dụng chính `:app` chỉ cần phụ thuộc vào `:api` và `:impl`. Nhờ vậy, các feature khác muốn giao tiếp với Profile chỉ cần phụ thuộc vào `:api`, giúp thời gian biên dịch (Build Time) độc lập và nhanh chóng hơn nhờ Gradle Caching.

#### Câu hỏi 3: Tại sao gọi `hiltViewModel()` trong Navigation Compose lại khác với `viewModel()` thông thường?
**Trả lời:**
- `viewModel()` thông thường gắn chặt vòng đời của ViewModel vào `LocalViewModelStoreOwner` gần nhất (thường là cả Activity hoặc NavBackStackEntry hiện tại).
- `hiltViewModel()` trong gói `androidx.hilt.navigation.compose` kiểm tra xem NavBackStackEntry đó có được hỗ trợ bởi Hilt Navigation Factory hay không. Nếu màn hình đó là một điểm đến trong NavHost, `hiltViewModel()` tự động scoped ViewModel vào đúng `NavBackStackEntry` của route đó. Khi người dùng pop màn hình khỏi backstack, `ViewModel.onCleared()` sẽ được gọi ngay lập tức để giải phóng tài nguyên.

### 6.2 Bảng gỡ rối các lỗi thực tế (Troubleshooting Matrix)

| Lỗi Compile / Runtime | Nguyên nhân gốc rễ (Root Cause) | Giải pháp triệt để (Solution) |
|---|---|---|
| `IllegalStateException: Hilt ViewModel cannot be created without @AndroidEntryPoint` | Composable chứa `hiltViewModel()` được gắn vào một `Activity` hoặc `Fragment` quên không đánh dấu `@AndroidEntryPoint` | Thêm `@AndroidEntryPoint` vào Activity/Fragment chứa giao diện Composable đó. |
| `[Dagger/MissingBinding]: cannot be provided without an @Provides- or @Inject-annotated constructor` | Quên `@Inject constructor()` trên class triển khai, hoặc quên khai báo `@Binds`/`@Provides` trong `@Module` có `@InstallIn` | Kiểm tra lại xem Interface đã được map với Implementation trong một Module phù hợp hay chưa. |
| `[Dagger/IncompatiblyScopedBindings]: scoped with @ActivityScoped may not be referenced from @Singleton` | Tiêm một dependency có vòng đời ngắn (`@ActivityScoped`) vào một đối tượng sống lâu (`@Singleton`) gây rò rỉ bộ nhớ | Điều chỉnh lại phạm vi Scope: Chỉ inject các dependency có vòng đời bằng hoặc dài hơn đối tượng nhận. |
| Crash khi chạy UI Test: `java.lang.IllegalStateException: The component was not created` | Quên thêm `HiltAndroidRule` hoặc sắp xếp sai thứ tự `@get:Rule` (`order = 0`) | Đặt `HiltAndroidRule(this)` với `order = 0` trước `ComposeTestRule` (`order = 1`). |

---

## 7. Tổng kết

Dagger Hilt loại bỏ sự bất an của các runtime-crashes bằng cách cung cấp **Đồ thị phụ thuộc an toàn tuyệt đối lúc biên dịch (Compile-Time Verified Graph)**. Khi kết hợp với Kotlin Flow (`stateIn`) và Jetpack Compose (`hiltViewModel()`), bạn tạo ra một nền tảng kiến trúc vững chắc, dễ kiểm thử và sẵn sàng mở rộng cho các hệ thống phần mềm doanh nghiệp hàng đầu.

*Bài tiếp theo: [Bài 20 — WorkManager + Flow: Reliable Background Work & Progress Observation](20-workmanager-flow.md)*
