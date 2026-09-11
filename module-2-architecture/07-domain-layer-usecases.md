# Bài 07 — Domain Layer & Use Cases với Flow

> **Module:** 2 — Architecture & ViewModel  
> **Prerequisite:** [Bài 06 — ViewModel + Flow + Compose: Kiến trúc UDF End-to-End](06-viewmodel-flow-compose-udf.md)  
> **Official Docs:**
> - [Domain Layer — Android Developers](https://developer.android.com/topic/architecture/domain-layer)
> - [Modern Android Architecture: Domain Layer Guide](https://developer.android.com/topic/architecture/recommendations#domain-layer)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

Trong hướng dẫn kiến trúc chính thức của Google (**Google Guide to App Architecture**), **Domain Layer (Tầng nghiệp vụ)** là một tầng tùy chọn (**optional**) nằm kẹp giữa **UI Layer** (ViewModel) và **Data Layer** (Repository).

```
┌────────────────────────────────────────────────────────────────────────┐
│                      CLEAN ARCHITECTURE HIERARCHY                      │
│                                                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ UI Layer: Activity / Jetpack Compose / ViewModel               │   │
│   └───────────────────────────────┬────────────────────────────────┘   │
│                                   │ Phụ thuộc (Depends on)             │
│                                   ▼                                    │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ Domain Layer (Optional): Use Cases / Interactors               │   │
│   │  - 100% Pure Kotlin (Không import bất kỳ Android SDK nào)      │   │
│   │  - Đóng gói Business Logic phức tạp hoặc tái sử dụng           │   │
│   └───────────────────────────────┬────────────────────────────────┘   │
│                                   │ Phụ thuộc (Depends on)             │
│                                   ▼                                    │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ Data Layer: Repositories / DataSources (Room, Retrofit, Ktor)  │   │
│   └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

### 1.1 Khái niệm Use Case (hoặc Interactor)
- **Use Case** là một lớp đối tượng đại diện cho **một hành động nghiệp vụ duy nhất** (Single Business Action) mà người dùng hoặc hệ thống có thể thực hiện trong ứng dụng.
- **Nguyên lý Đơn nhiệm (Single Responsibility Principle - SRP):** Mỗi Use Case chỉ chịu trách nhiệm cho đúng một tác vụ logic (ví dụ: `FormatDateUseCase`, `GetNewsWithBookmarksUseCase`, `ApplyDiscountCouponUseCase`).

---

### 1.2 Phân định rạch ròi: Business Logic vs UI Logic vs Data Logic

Rất nhiều kỹ sư Android nhầm lẫn giữa 3 loại logic này, dẫn đến việc đặt code sai tầng:

| Loại Logic | Định nghĩa | Nơi thực thi chuẩn mực | Ví dụ thực tế |
|---|---|---|---|
| **Business Logic** | Các quy tắc nghiệp vụ mang lại giá trị cho doanh nghiệp, độc lập hoàn toàn với nền tảng Android. | **Domain Layer (Use Case)** | Công thức tính thuế VAT; VIP Member được giảm thêm 5%; nếu giỏ hàng > 500k thì miễn phí vận chuyển. |
| **UI Logic** | Cách thức hiển thị dữ liệu lên màn hình hoặc điều hướng giao diện. | **UI Layer (ViewModel / Composable)** | Hiển thị Dialog xác nhận; đổi màu nút bấm khi form hợp lệ; chuyển tab khi click; format chuỗi hiển thị. |
| **Data Logic** | Cách thức lưu trữ, đồng bộ và truy xuất dữ liệu từ các nguồn khác nhau. | **Data Layer (Repository)** | Lưu cache vào Room Database; gọi Retrofit API; xử lý token refresh khi gặp lỗi 401; phân trang Paging. |

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Use Case đóng vai trò Orchestrator (Nhạc trưởng điều phối)

Một Use Case không trực tiếp thao tác với cơ sở dữ liệu hay mạng. Bản chất của nó là một **bộ điều phối (Orchestrator)** gom dữ liệu từ nhiều Repositories độc lập, áp dụng các thuật toán nghiệp vụ, và trả về kết quả đã được tinh chế cho ViewModel.

```
                  SƠ ĐỒ PHỐI HỢP CỦA USE CASE (ORCHESTRATOR)
                  
                            CheckoutViewModel
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │  ApplyDiscountCouponUseCase  │
                    └──────────────┬───────────────┘
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         ▼                         ▼                         ▼
  UserRepository           CouponRepository            CartRepository
(Hạng thành viên VIP)   (Kiểm tra mã hợp lệ)       (Lấy tổng tiền giỏ hàng)
         │                         │                         │
         └─────────────────────────┼─────────────────────────┘
                                   ▼
               Áp dụng thuật toán giảm giá (Business Logic)
                                   │
                                   ▼
                   Trả về DiscountResult cho ViewModel
```

---

### 2.2 Kotlin Convention: `operator fun invoke()`

Để làm cho Use Case trở nên trực quan và có cú pháp thanh lịch như một lời gọi hàm thông thường, Kotlin cung cấp từ khóa `operator fun invoke()`:

```kotlin
class GetUserSubscriptionUseCase(
    private val repository: UserRepository
) {
    // Nạp chồng toán tử invoke()
    suspend operator fun invoke(userId: String): SubscriptionStatus {
        return repository.getSubscription(userId)
    }
}

// Khi sử dụng trong ViewModel:
// Bạn có thể gọi trực tiếp instance như một function mà không cần .execute() hay .run()!
val status = getUserSubscriptionUseCase("USER_007") 
```

---

### 2.3 Quản lý Threading độc lập với `@DefaultDispatcher`

Theo khuyến nghị của Google Architecture:
- Nếu một Use Case thực hiện các thuật toán tính toán phức tạp, xử lý mảng lớn hoặc mã hóa dữ liệu (**CPU-intensive work**), chính Use Case đó phải chịu trách nhiệm đảm bảo **Main-Safety** bằng cách tự động chuyển sang `Dispatchers.Default`:

```kotlin
class CalculateComplexPortfolioUseCase(
    private val stockRepository: StockRepository,
    private val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default
) {
    suspend operator fun invoke(): PortfolioSummary = withContext(defaultDispatcher) {
        val stocks = stockRepository.getAllStocks()
        // Thuật toán tính toán độ lệch chuẩn và định giá danh mục nặng CPU...
        computeFinancialMetrics(stocks)
    }
}
```

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Bệnh nan y: "Massive / Fat ViewModel"

Khi không có Domain Layer, ViewModel thường xuyên bị phình to ra hàng nghìn dòng vì phải đảm nhận quá nhiều trách nhiệm:

```kotlin
// ANTI-PATTERN: ViewModel ôm đồm toàn bộ thế giới
class CartViewModel(
    private val cartRepo: CartRepository,
    private val userRepo: UserRepository,
    private val couponRepo: CouponRepository,
    private val taxRepo: TaxRepository,
    private val shippingRepo: ShippingRepository
) : ViewModel() {

    fun checkout() {
        // 1. Validate giỏ hàng
        // 2. Tính tiền thuế theo từng bang
        // 3. Kiểm tra mã giảm giá
        // 4. Áp dụng ưu đãi thành viên
        // 5. Tính phí vận chuyển theo khoảng cách
        // => 500 DÒNG CODE BUSINESS LOGIC NẰM TRONG VIEWMODEL!
    }
}
```

#### Hậu quả khôn lường:
1. **Trùng lặp mã nguồn (Code Duplication):** Khi màn hình `QuickBuyBottomSheet` hoặc `OrderHistoryScreen` cần kiểm tra lại cùng một công thức tính thuế hoặc giảm giá, lập trình viên buộc phải **copy-paste toàn bộ 500 dòng code** đó sang ViewModel mới!
2. **Khó kiểm thử (Testing Nightmare):** Để viết Unit Test kiểm tra công thức tính thuế, bạn phải khởi tạo cả một ViewModel khổng lồ với 5 mocks repositories và môi trường `TestDispatcher`.
3. **Phá vỡ ranh giới kiến trúc:** Thay đổi nhỏ trong quy định khuyến mãi của phòng kinh doanh có thể vô tình làm vỡ logic hiển thị của giao diện UI.

#### Domain Layer giải quyết triệt để:
Tách từng phần việc thành các Use Cases độc lập: `CalculateTaxUseCase`, `ApplyCouponUseCase`, `CalculateShippingUseCase`. Mỗi Use Case chỉ có độ dài 30-50 dòng, thuần túy Kotlin, test trong 5 mili-giây!

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Khi nào NÊN và KHÔNG NÊN dùng Domain Layer?

```
BẠN CÓ NÊN TẠO USE CASE HAY KHÔNG?
│
├── Tính năng chỉ là CRUD 1-1 đơn giản? ─────────────────────────► KHÔNG CẦN!
│   (Ví dụ: UserRepository.getUser() -> ViewModel -> UI)          (Gọi thẳng Repository từ ViewModel)
│
├── Có thuật toán nghiệp vụ phức tạp? ──────────────────────────► BẮT BUỘC NÊN DÙNG!
│   (Tính điểm tín dụng, lọc danh sách theo 5 điều kiện nghiệp vụ)
│
└── Logic được DÙNG CHUNG bởi 2 ViewModel trở lên hoặc Worker? ──► BẮT BUỘC NÊN DÙNG!
    (Đồng bộ dữ liệu, validate coupon, format hồ sơ người dùng)
```

> **Lời khuyên từ Google Architect:** Không biến ứng dụng thành "Kiến trúc quan liêu" (Over-engineering). Nếu bạn chỉ tạo một Use Case chỉ để chuyển tiếp một lời gọi hàm duy nhất `fun invoke() = repository.getData()`, bạn đang làm phức tạp hóa dự án mà không mang lại bất kỳ giá trị thực tế nào.

---

### 4.2 Các nguyên tắc vàng khi thiết kế Use Case

#### 1. Use Case phải hoàn toàn VÔ TRẠNG THÁI (Stateless)
Một Use Case không bao giờ được phép chứa biến `var` hay lưu trữ trạng thái có thể biến đổi. Mọi dữ liệu phải đi vào qua tham số và đi ra qua giá trị trả về:
```kotlin
// SAI (ANTI-PATTERN):
class BadUseCase {
    private var cachedDiscount: Double = 0.0 // NGUY CƠ RACE CONDITION!
}

// ĐÚNG:
class GoodUseCase {
    operator fun invoke(params: Params): Result = ... // Hoàn toàn Stateless
}
```

#### 2. Không lạm dụng Base Use Case Class
Trước đây, nhiều dự án thường tạo ra các base class như:
```kotlin
// KHÔNG NÊN DÙNG (ANTI-PATTERN THEO GOOGLE):
abstract class BaseUseCase<in P, out R> {
    abstract suspend fun execute(params: P): R
}
```
**Tại sao Google khuyên không nên dùng Base Use Case?**
- Nó ép buộc tất cả Use Case vào cùng một khuôn mẫu cứng nhắc.
- Có Use Case cần nhận 3 tham số, có cái không cần tham số nào.
- Có Use Case trả về `Flow<T>`, có cái trả về `suspend fun`, có cái trả về giá trị đồng bộ `T`.
- Sử dụng `operator fun invoke()` với chữ ký hàm linh hoạt tự nhiên là giải pháp sạch sẽ và chuẩn mực nhất.

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Xây dựng Use Case nghiệp vụ thực tế: **Áp dụng mã giảm giá và tính toán ưu đãi thành viên VIP (`ApplyDiscountCouponUseCase`)**.

### Bước 1: Domain Entities & Repository Interfaces

```kotlin
// 1. Pure Domain Entities (Không phụ thuộc DB hay API DTO)
enum class UserTier { STANDARD, VIP }

data class User(val id: String, val name: String, val tier: UserTier)

data class Coupon(val code: String, val discountPercent: Int, val maxDiscountAmount: Double)

data class DiscountResult(
    val originalAmount: Double,
    val discountAmount: Double,
    val finalAmount: Double,
    val vipBonusApplied: Boolean
)

// 2. Repository Interfaces định nghĩa trong Domain Layer
interface UserRepository {
    suspend fun getCurrentUser(userId: String): User
}

interface CouponRepository {
    suspend fun getCouponByCode(code: String): Coupon?
}
```

---

### Bước 2: Triển khai `ApplyDiscountCouponUseCase` (Business Logic)

```kotlin
class ApplyDiscountCouponUseCase(
    private val userRepository: UserRepository,
    private val couponRepository: CouponRepository,
    private val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default
) {

    suspend operator fun invoke(
        userId: String,
        couponCode: String,
        cartTotal: Double
    ): Result<DiscountResult> = withContext(defaultDispatcher) {
        // 1. Kiểm tra tính hợp lệ cơ bản
        if (cartTotal <= 0) {
            return@withContext Result.failure(IllegalArgumentException("Tổng giỏ hàng phải lớn hơn 0"))
        }

        // 2. Lấy thông tin mã giảm giá từ Repository
        val coupon = couponRepository.getCouponByCode(couponCode)
            ?: return@withContext Result.failure(NoSuchElementException("Mã giảm giá không tồn tại hoặc đã hết hạn"))

        // 3. Lấy thông tin người dùng để kiểm tra hạng thành viên
        val user = userRepository.getCurrentUser(userId)

        // 4. QUY TẮC NGHIỆP VỤ (BUSINESS RULES):
        // - Mã giảm giá cơ bản: discountPercent từ coupon
        // - Nếu là thành viên VIP: Được cộng thêm 5% ưu đãi độc quyền
        val isVip = (user.tier == UserTier.VIP)
        val totalDiscountPercent = if (isVip) {
            coupon.discountPercent + 5
        } else {
            coupon.discountPercent
        }

        // 5. Tính số tiền giảm và áp dụng mức giảm tối đa (Cap threshold)
        var calculatedDiscount = cartTotal * (totalDiscountPercent / 100.0)
        if (calculatedDiscount > coupon.maxDiscountAmount) {
            calculatedDiscount = coupon.maxDiscountAmount
        }

        val finalPrice = (cartTotal - calculatedDiscount).coerceAtLeast(0.0)

        Result.success(
            DiscountResult(
                originalAmount = cartTotal,
                discountAmount = calculatedDiscount,
                finalAmount = finalPrice,
                vipBonusApplied = isVip
            )
        )
    }
}
```

---

### Bước 3: Tích hợp Use Case vào ViewModel

```kotlin
class CheckoutViewModel(
    private val applyDiscountCouponUseCase: ApplyDiscountCouponUseCase
) : ViewModel() {

    private val _uiState = MutableStateFlow(CheckoutUiState())
    val uiState: StateFlow<CheckoutUiState> = _uiState.asStateFlow()

    fun onApplyCouponClicked(code: String) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, errorMessage = null) }

            val result = applyDiscountCouponUseCase(
                userId = "USER_123",
                couponCode = code,
                cartTotal = _uiState.value.cartTotal
            )

            result.fold(
                onSuccess = { discountResult ->
                    _uiState.update {
                        it.copy(
                            isLoading = false,
                            discountResult = discountResult,
                            finalTotal = discountResult.finalAmount
                        )
                    }
                },
                onFailure = { error ->
                    _uiState.update {
                        it.copy(isLoading = false, errorMessage = error.message)
                    }
                }
            )
        }
    }
}
```

---

### Bước 4: Viết Unit Test thuần JVM cho Use Case (Không cần Android SDK)

Vì Use Case là **Pure Kotlin**, bài kiểm thử này chạy trực tiếp trên JVM trong chưa đầy 10ms!

```kotlin
class ApplyDiscountCouponUseCaseTest {

    private val testDispatcher = StandardTestDispatcher()
    private lateinit var fakeUserRepo: UserRepository
    private lateinit var fakeCouponRepo: CouponRepository
    private lateinit var useCase: ApplyDiscountCouponUseCase

    @Before
    fun setUp() {
        fakeUserRepo = object : UserRepository {
            override suspend fun getCurrentUser(userId: String) =
                User(userId, "Harry", if (userId == "VIP_USER") UserTier.VIP else UserTier.STANDARD)
        }

        fakeCouponRepo = object : CouponRepository {
            override suspend fun getCouponByCode(code: String): Coupon? {
                return if (code == "SUMMER20") {
                    Coupon(code = "SUMMER20", discountPercent = 20, maxDiscountAmount = 50.0)
                } else null
            }
        }

        useCase = ApplyDiscountCouponUseCase(fakeUserRepo, fakeCouponRepo, testDispatcher)
    }

    @Test
    fun standardUser_appliesStandardDiscount() = runTest(testDispatcher) {
        // Giỏ hàng 100$, giảm 20% = 20$
        val result = useCase(userId = "STANDARD_USER", couponCode = "SUMMER20", cartTotal = 100.0)

        assertTrue(result.isSuccess)
        val data = result.getOrNull()!!
        assertEquals(20.0, data.discountAmount, 0.0)
        assertEquals(80.0, data.finalAmount, 0.0)
        assertFalse(data.vipBonusApplied)
    }

    @Test
    fun vipUser_receivesAdditional5PercentBonus() = runTest(testDispatcher) {
        // Giỏ hàng 100$, giảm 20% + 5% VIP = 25% = 25$
        val result = useCase(userId = "VIP_USER", couponCode = "SUMMER20", cartTotal = 100.0)

        assertTrue(result.isSuccess)
        val data = result.getOrNull()!!
        assertEquals(25.0, data.discountAmount, 0.0)
        assertEquals(75.0, data.finalAmount, 0.0)
        assertTrue(data.vipBonusApplied)
    }

    @Test
    fun discount_cappedAtMaxDiscountAmount() = runTest(testDispatcher) {
        // Giỏ hàng 1000$, giảm 20% = 200$, nhưng maxDiscountAmount là 50$ -> Bị giới hạn ở 50$
        val result = useCase(userId = "STANDARD_USER", couponCode = "SUMMER20", cartTotal = 1000.0)

        assertTrue(result.isSuccess)
        val data = result.getOrNull()!!
        assertEquals(50.0, data.discountAmount, 0.0)
        assertEquals(950.0, data.finalAmount, 0.0)
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior/Staff Android Architect

#### Q1: Use Case có nên gọi một Use Case khác không?
**Trả lời chuẩn bản chất:**
**Có thể, nhưng nên hạn chế.** Việc một Use Case cấp cao (Composite Use Case) phối hợp các Use Case cấp thấp hơn là hoàn toàn hợp lệ trong Clean Architecture. Ví dụ: `CheckoutUseCase` có thể gọi `ValidateCartUseCase` và `ProcessPaymentUseCase`. Tuy nhiên, lập trình viên phải cực kỳ thận trọng để tránh **Circular Dependency (Vòng lặp phụ thuộc A gọi B, B gọi lại A)**.

#### Q2: Khi nào Use Case nên trả về `Flow<T>` và khi nào nên trả về `suspend fun`?
**Trả lời chuẩn bản chất:**
- **Trả về `Flow<T>`:** Khi Use Case lắng nghe một nguồn dữ liệu có thể biến động liên tục theo thời gian (Observable Data Stream). Ví dụ: `GetMessagesStreamUseCase`, `ObserveUserLocationUseCase`.
- **Dùng `suspend fun`:** Khi Use Case đại diện cho một hành động có điểm kết thúc xác định (One-shot Execution / Request-Response). Ví dụ: `LoginUseCase`, `SubmitOrderUseCase`, `DeleteArticleUseCase`.

#### Q3: Có nên tạo interface cho tất cả các Use Case không (ví dụ `interface LoginUseCase` và `class LoginUseCaseImpl`)?
**Trả lời chuẩn bản chất:**
**Không nên.** Trong hầu hết các dự án Android thực tế, một Use Case chỉ có đúng một bản triển khai duy nhất. Việc tạo thêm interface cho mọi Use Case gây ra sự lãng phí tệp tin và boilerplate không cần thiết. Vì bản thân Use Case chỉ nhận các dependency qua constructor (thường là Repository interfaces), bạn hoàn toàn có thể truyền Fake Repositories vào để kiểm thử mà không cần phải mock chính bản thân Use Case!

---

### 6.2 Lỗi Kiến trúc thường gặp & Hướng xử lý

| Tình huống lỗi | Nguyên nhân gốc rễ | Cách xử lý triệt để |
|---|---|---|
| **Rò rỉ Android Framework vào Domain Layer** | Import `android.text.TextUtils`, `Context` hoặc Compose dependencies vào Use Case. | Xóa toàn bộ import Android. Thay thế bằng các hàm chuẩn của Kotlin (`CharSequence.isNullOrBlank()`). |
| **Unit Test bị chậm hoặc treo (Timeout)** | Hardcode `Dispatchers.Default` bên trong Use Case thay vì inject `CoroutineDispatcher`. | Inject `CoroutineDispatcher` qua constructor với giá trị mặc định là `Dispatchers.Default`. Trong Unit Test, truyền `StandardTestDispatcher`. |
| **Kiến trúc quá tải (Over-engineering)** | Tạo hàng chục Use Case rỗng chỉ để gọi `repo.getAll()` mà không có bất kỳ logic nào. | Xóa bỏ các Use Case rỗng, cho phép ViewModel gọi trực tiếp Repository nếu chỉ là thao tác dữ liệu thuần túy 1-1. |

---

*Bài trước: [06 — ViewModel + Flow + Compose UDF](06-viewmodel-flow-compose-udf.md)*  
*Bài tiếp theo: [08 — Repository Pattern với Flow: Offline-First](08-repository-pattern-flow.md)*
