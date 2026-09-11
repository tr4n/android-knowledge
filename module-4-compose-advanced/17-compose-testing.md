# Bài 17 — Compose Testing: ComposeTestRule, Semantics Tree & Screenshot Testing

> **Module:** 4 — Jetpack Compose Advanced  
> **Prerequisite:** [Bài 16 — Performance & Lazy Layouts: LazyColumn, Keys](16-performance-lazy-layouts.md)  
> **Official Docs:**
> - [Testing Compose layouts — Android Developers](https://developer.android.com/develop/ui/compose/testing)
> - [Semantics in Jetpack Compose](https://developer.android.com/develop/ui/compose/semantics)
> - [Roborazzi — JVM Screenshot Testing for Android](https://github.com/takahirom/roborazzi)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

Trong Jetpack Compose, giao diện người dùng được vẽ trực tiếp lên một Canvas phần cứng duy nhất thông qua các hàm toán học, không còn tồn tại hệ thống phân cấp các `android.view.View` riêng lẻ. Do đó, các công cụ kiểm thử truyền thống như Espresso không thể tìm kiếm hay tương tác với các thành phần UI bằng ID (`R.id.button`).

Để phục vụ cho cả tính năng trợ năng (**Accessibility**) và kiểm thử tự động (**UI Testing**), Compose xây dựng một cây thông tin song song gọi là **Semantics Tree (Cây ngữ nghĩa)**.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        COMPOSE TESTING FRAMEWORK                       │
│                                                                        │
│   1. Semantics Tree (Cây ngữ nghĩa):                                   │
│      - Tồn tại song song với Composition Tree                          │
│      - Chứa ý nghĩa nội dung: text, role, onClick, isSelected...       │
│                                                                        │
│   2. Bốn trụ cột của Compose Test API:                                 │
│      ├── Finders:     onNode(), onAllNodes()                           │
│      ├── Matchers:    hasText(), hasClickAction(), hasTestTag()        │
│      ├── Actions:     performClick(), performTextInput(), performScroll│
│      └── Assertions:  assertIsDisplayed(), assertIsEnabled(), assert...│
│                                                                        │
│   3. Test Rules:                                                       │
│      - ComposeTestRule: Kiểm thử độc lập Composable (cực nhanh)        │
│      - AndroidComposeTestRule: Kiểm thử tích hợp gắn với Activity      │
│                                                                        │
│   4. Visual Regression:                                                │
│      - Screenshot Testing (Roborazzi / Paparazzi): So sánh pixel ảnh   │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Cây ngữ nghĩa (Semantics Tree) và Cơ chế Gộp Node (Merging)

Làm thế nào Compose Test Engine "nhìn thấy" một nút bấm bao gồm cả icon và dòng chữ bên trong?

```
CÂY COMPOSITION THỰC TẾ               CÂY NGỮ NGHĨA GỘP (MERGED SEMANTICS TREE)
┌──────────────────────┐              ┌──────────────────────────────────────┐
│ Button(onClick = {}) │              │ Node: [Button]                       │
│   │                  │              │ Properties:                          │
│   ├── Row            │  ──GỘP LẠI─► │  - Role: Button                      │
│   │     ├── Icon     │              │  - OnClickAction: Defined            │
│   │     └── Text("OK│              │  - Text: ["OK"]                      │
└──────────────────────┘              └──────────────────────────────────────┘
```

#### Phân biệt Merged Tree vs Unmerged Tree:
1. **Merged Semantics Tree (Mặc định):**
   - Tự động gộp tất cả các node con của một thành phần tương tác (như Button, ListItem, Checkbox) thành **MỘT node duy nhất**.
   - Phản ánh chính xác cách các công cụ trợ năng (TalkBack / Screen Reader) đọc thông tin cho người khiếm thị. Khi tìm kiếm: `composeTestRule.onNodeWithText("OK")` sẽ tìm thấy chính nút Button đó!
2. **Unmerged Semantics Tree (`useUnmergedTree = true`):**
   - Tách rời từng node con riêng biệt. Chỉ sử dụng khi bạn cần kiểm tra chi tiết cấu trúc nội bộ của component con (ví dụ kiểm tra riêng icon bên trong button).

---

### 2.2 Cơ chế Tự động đồng bộ hóa (Auto-Synchronization)

Một trong những ưu thế vượt bậc của Compose Testing so với Espresso là **loại bỏ hoàn toàn các bài test chập chờn (Flaky Tests)** nhờ cơ chế **Auto-Synchronization**:

```
                  CƠ CHẾ AUTO-SYNCHRONIZATION CỦA COMPOSE
                  
Câu lệnh Test: composeTestRule.onNodeWithText("Submit").performClick()
                               │
                               ▼
   ComposeTestRule KIỂM TRA ĐIỀU KIỆN IDLE TRƯỚC KHI THỰC THI:
   ├── 1. Có Composable nào đang Recompose không?
   ├── 2. Có Coroutine nào đang chạy trong CoroutineContext của UI không?
   └── 3. Có Animation nào đang chạy frame VSYNC không?
                               │
                ┌──────────────┴──────────────┐
                ▼                             ▼
         [ ĐANG BẬN / CHẠY ]               [ IDLE / RẢNH ]
                │                             │
         TỰ ĐỘNG CHỜ ĐỢI                     ▼
         (Không cần Thread.sleep!)     THỰC THI LỆNH VÀ ASSERTION
                                       CHÍNH XÁC 100%!
```

---

### 2.3 Điều khiển Thời gian Ảo với Compose Virtual Clock

Khi kiểm thử các tính năng có animation kéo dài (như mở rộng thẻ sau 500ms) hoặc bộ đếm ngược (đếm 60s), bạn **không bao giờ phải chờ 60 giây ngoài đời thực**. Compose cung cấp Virtual Clock:

```kotlin
// Tắt chế độ tự động chạy giờ
composeTestRule.mainClock.autoAdvance = false

// Tua nhanh thời gian ảo đúng 1000ms trong 1 phần triệu giây!
composeTestRule.mainClock.advanceTimeBy(1000L)
```

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Nỗi thất vọng với Espresso trong hệ thống cũ

| Tiêu chí | Espresso cũ (View System) | Compose Test Framework |
|---|---|---|
| **Cơ chế tìm kiếm** | Phụ thuộc vào Resource ID (`R.id.submit_button`). Dễ vỡ khi đổi ID. | Tìm kiếm dựa trên **Ngữ nghĩa hiển thị** (`onNodeWithText("Xác nhận")`). Bám sát trải nghiệm người dùng thực tế. |
| **Xử lý bất đồng bộ** | Phải cấu hình `IdlingResource` thủ công phức tạp. Quên là test bị fail ngẫu nhiên (Flaky Test). | **Tự động đồng bộ 100%**: Tự động đợi Recomposition và Coroutines kết thúc. |
| **Tốc độ thực thi** | Bắt buộc phải chạy trên Android Emulator hoặc thiết bị thật (chậm chạp, tốn RAM). | Có thể chạy trực tiếp trên **JVM Desktop** bằng Robolectric / Roborazzi với tốc độ siêu tốc (vài giây cho cả suite test). |

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Quy tắc lựa chọn Matcher chuẩn mực từ Google

```
BẠN NÊN TÌM KIẾM NODE BẰNG CÁCH NÀO?
│
├── 1. ƯU TIÊN SỐ 1: onNodeWithText("Nội dung hiển thị")
│      (Bám sát nhất với những gì người dùng thực tế nhìn thấy)
│
├── 2. ƯU TIÊN SỐ 2: onNodeWithContentDescription("Mô tả trợ năng")
│      (Dành cho Icon, ImageButton có gắn nhãn mô tả cho TalkBack)
│
└── 3. LỰA CHỌN CUỐI CÙNG: onNodeWithTag("unique_test_id")
       (Chỉ dùng cho các phần tử đồ họa thuần túy không có chữ hay mô tả)
```

> **Lời khuyên của Senior Architect:** Đừng biến toàn bộ code Compose của bạn thành một mớ rác rưởi đầy rẫy các thẻ `Modifier.testTag("id")`. Hãy thiết kế giao diện có đầy đủ `contentDescription` và text rõ ràng; điều này vừa giúp code test dễ dàng vừa nâng cao điểm số chuẩn Trợ năng (Accessibility) của ứng dụng lên mức tối đa!

---

### 4.2 Phân biệt `assertIsDisplayed()` vs `assertExists()`

- **`assertExists()`**: Node đó **có tồn tại trong cây Semantics Tree** (nhưng có thể đang nằm ngoài màn hình chưa cuộn tới, hoặc bị ẩn với `alpha = 0f`).
- **`assertIsDisplayed()`**: Node đó **thực sự đang hiển thị trên màn hình** và người dùng có thể nhìn thấy bằng mắt thường (chiếm ít nhất 1% diện tích viewport).
> **Quy tắc:** Luôn dùng `assertIsDisplayed()` khi kiểm tra các phần tử tương tác của người dùng.

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Xây dựng bộ kiểm thử hoàn chỉnh cho tính năng **Giỏ Hàng & Đặt Hàng (Cart & Checkout UI Flow)**:
- Kiểm thử Component đơn lẻ (Stateless Composable Test).
- Kiểm thử tích hợp toàn màn hình (Screen Integration Test với Fake ViewModel).
- Hướng dẫn thiết lập **Visual Regression / Screenshot Testing** với thư viện **Roborazzi**.

### Bước 1: Khai báo Dependencies

```kotlin
// build.gradle.kts (Module: app)
dependencies {
    // Compose Testing (JUnit 4)
    androidTestImplementation(platform("androidx.compose:compose-bom:2024.06.00"))
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    debugImplementation("androidx.compose.ui:ui-test-manifest")

    // Roborazzi Screenshot Testing trên JVM (Không cần Emulator!)
    testImplementation("io.github.takahirom.roborazzi:roborazzi:1.26.0")
    testImplementation("io.github.takahirom.roborazzi:roborazzi-compose:1.26.0")
    testImplementation("org.robolectric:robolectric:4.13")
}
```

---

### Bước 2: Triển khai Component cần test (`CartSummaryCard`)

```kotlin
@Composable
fun CartSummaryCard(
    itemCount: Int,
    totalPrice: Double,
    isCheckoutEnabled: Boolean,
    onCheckoutClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Card(
        modifier = modifier.fillMaxWidth().padding(16.dp),
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(
                text = "Tổng số lượng: $itemCount sản phẩm",
                style = MaterialTheme.typography.titleMedium
            )
            Spacer(modifier = Modifier.height(4.dp))
            Text(
                text = "Thành tiền: $$totalPrice",
                style = MaterialTheme.typography.headlineSmall,
                color = MaterialTheme.colorScheme.primary
            )
            Spacer(modifier = Modifier.height(16.dp))
            Button(
                onClick = onCheckoutClick,
                enabled = isCheckoutEnabled,
                modifier = Modifier.fillMaxWidth()
            ) {
                Text("Tiến hành thanh toán")
            }
        }
    }
}
```

---

### Bước 3: Viết Unit Test giao diện độc lập với `createComposeRule()`

```kotlin
@RunWith(AndroidJUnit4::class)
class CartSummaryCardTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun cartSummaryCard_displaysCorrectInfo_andDisablesButtonWhenEmpty() {
        var checkoutClicked = false

        composeTestRule.setContent {
            CartSummaryCard(
                itemCount = 0,
                totalPrice = 0.0,
                isCheckoutEnabled = false,
                onCheckoutClick = { checkoutClicked = true }
            )
        }

        // 1. Kiểm tra văn bản hiển thị
        composeTestRule.onNodeWithText("Tổng số lượng: 0 sản phẩm").assertIsDisplayed()
        composeTestRule.onNodeWithText("Thành tiền: $0.0").assertIsDisplayed()

        // 2. Nút bấm phải bị vô hiệu hóa (Disabled)
        val checkoutButton = composeTestRule.onNodeWithText("Tiến hành thanh toán")
        checkoutButton.assertIsDisplayed()
        checkoutButton.assertIsNotEnabled()

        // 3. Click thử vào nút disabled -> Biến callback không được phép đổi giá trị!
        checkoutButton.performClick()
        assertFalse(checkoutClicked)
    }

    @Test
    fun cartSummaryCard_enablesButton_andTriggersCallback_whenValid() {
        var checkoutClicked = false

        composeTestRule.setContent {
            CartSummaryCard(
                itemCount = 3,
                totalPrice = 150.0,
                isCheckoutEnabled = true,
                onCheckoutClick = { checkoutClicked = true }
            )
        }

        val checkoutButton = composeTestRule.onNodeWithText("Tiến hành thanh toán")
        checkoutButton.assertIsEnabled()

        // Thực hiện tương tác Click
        checkoutButton.performClick()
        assertTrue(checkoutClicked)
    }
}
```

---

### Bước 4: Viết Screenshot Test trực tiếp trên JVM với Roborazzi

Screenshot Testing giúp phát hiện các lỗi vỡ layout hoặc sai lệch màu sắc pixel mà unit test thông thường không thể phát hiện ra.

```kotlin
@RunWith(RobolectricTestRunner::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
class CartSummaryScreenshotTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun captureCartSummaryCard_normalState() {
        composeTestRule.setContent {
            MaterialTheme {
                CartSummaryCard(
                    itemCount = 5,
                    totalPrice = 499.99,
                    isCheckoutEnabled = true,
                    onCheckoutClick = {}
                )
            }
        }

        // Chụp ảnh màn hình và so sánh với ảnh chuẩn (Golden Image) trong thư mục test
        composeTestRule.onRoot().captureRoboImage("screenshots/cart_summary_normal.png")
    }
}
```

Chạy lệnh `./gradlew recordRoborazziDebug` để chụp ảnh mẫu, sau đó chạy `./gradlew verifyRoborazziDebug` trên hệ thống CI/CD để tự động phát hiện bất kỳ pixel nào bị lệch!

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior/Staff Android Architect

#### Q1: Tại sao lệnh `onNodeWithText("Submit")` báo lỗi không tìm thấy node dù nút có chữ "Submit"?
**Trả lời chuẩn bản chất:**
Lỗi này thường xảy ra khi Composable con nằm bên trong một container bị ẩn hoặc bạn đang tìm kiếm trên cây **Unmerged Tree** mà quên truyền cờ `useUnmergedTree = true`. Ví dụ: Một component tùy biến bọc Text bên trong nhiều layout lồng nhau và đặt thuộc tính `clearAndSetSemantics { }`. Thao tác này sẽ xóa sạch toàn bộ thông tin ngữ nghĩa của các node con, khiến Test Engine không thể tìm thấy chữ "Submit" được nữa!

#### Q2: Làm thế nào để kiểm thử một màn hình có chứa Animation vô tận (như CircularProgressIndicator) mà không bị treo Test Timeout?
**Trả lời chuẩn bản chất:**
Một animation vô tận (`rememberInfiniteTransition`) sẽ liên tục phát ra frame mới trên VSYNC clock, khiến `ComposeTestRule` hiểu rằng hệ thống **không bao giờ ở trạng thái Idle**. Hậu quả là bài test sẽ bị treo cho đến khi ném lỗi `ComposeTimeoutException`.
**Cách xử lý chuẩn:**
1. Tạm dừng đồng hồ tự động: `composeTestRule.mainClock.autoAdvance = false`.
2. Hoặc sử dụng cờ kiểm thử để thay thế `CircularProgressIndicator` bằng một Text hiển thị `"Đang tải..."` khi chạy UI test.

#### Q3: Khi nào nên sử dụng `waitUntil { }` thay vì dựa vào cơ chế Auto-Sync của Compose?
**Trả lời chuẩn bản chất:**
Mặc dù Compose tự động đồng bộ hóa với các coroutine trong phạm vi UI, nó **hoàn toàn không thể tự đồng bộ hóa với các tác vụ bất đồng bộ nằm ngoài quyền quản lý của Compose** (ví dụ: gọi API Retrofit thực tế, chuyển hướng qua WebView, hoặc lắng nghe sự kiện từ Firebase Socket). Trong các trường hợp đó, bạn bắt buộc phải dùng:
```kotlin
composeTestRule.waitUntil(timeoutMillis = 5000) {
    composeTestRule.onAllNodesWithText("Dữ liệu đã về").fetchSemanticsNodes().isNotEmpty()
}
```

---

### 6.2 Lỗi Runtime thường gặp & Hướng xử lý

| Tình huống lỗi | Nguyên nhân gốc rễ | Cách xử lý triệt để |
|---|---|---|
| **Crash: `ComposeTimeoutException: Timed out waiting for idle state`** | Có một Coroutine chạy vô hạn (`while(isActive) delay(100)`) hoặc animation lặp vô tận trong UI. | Tắt `mainClock.autoAdvance = false` hoặc hủy coroutine khi hoàn tất công việc. |
| **Lỗi `AssertionError: Expected exactly '1' node but found '2'`** | Bộ lọc tìm kiếm quá chung chung (ví dụ tìm chữ "Home" nhưng cả Title và BottomBar đều có chữ "Home"). | Thêm bộ lọc cha-con hoặc tìm kiếm cụ thể: `hasText("Home") and hasClickAction()`. |
| **Screenshot Test bị sai lệch font chữ khi chạy trên máy Mac và Linux CI** | Khác biệt về engine vẽ font giữa hệ điều hành macOS và môi trường Ubuntu CI. | Cấu hình Roborazzi sử dụng Docker hoặc font chữ nhúng tĩnh (Bundled TTF Font) để đảm bảo 100% đồng nhất pixel. |

---

*Bài trước: [16 — Performance & Lazy Layouts: LazyColumn, Keys, Baseline Profiles](16-performance-lazy-layouts.md)*  
*Module tiếp theo: [Module 5 — Advanced Patterns](../module-5-advanced-patterns/18-mvi-architecture.md)*
