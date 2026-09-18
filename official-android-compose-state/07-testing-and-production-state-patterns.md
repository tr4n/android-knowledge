# Bài 07 — Thực Hành Kiến Trúc & Kiểm Thử State: Case Study Phức Tạp & StateRestorationTester

> **Tài liệu tham chiếu chính thức:** [Testing Compose layouts & State — Android Developers](https://developer.android.com/develop/ui/compose/testing)  
> **Phiên bản áp dụng:** Kotlin 2.0+, Compose BOM 2024.06+, Turbine 1.2+, JUnit4  
> **Ngôn ngữ:** Tiếng Việt chuyên ngành (bảo lưu thuật ngữ quốc tế chuẩn trong ngoặc đơn)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

Kiểm thử trạng thái (State Testing) là mắt xích quan trọng nhất để đảm bảo giao diện người dùng hoạt động chính xác, ổn định và không bao giờ đánh mất dữ liệu của người dùng. Trong Jetpack Compose, chiến lược kiểm thử State tuân theo mô hình **Kim tự tháp kiểm thử 3 tầng (3-Tier State Testing Pyramid)**:

```
                               ▲
                              / \
                             /   \
                            / UI  \        3. COMPOSE UI & STATE RESTORATION TEST
                           / TESTS \          - createComposeRule()
                          /─────────\         - StateRestorationTester (Giả lập Process Death)
                         / INTEGRATION\
                        /   TESTS      \   2. VIEWMODEL STATE FLOW TEST
                       /────────────────\     - StandardTestDispatcher & Turbine
                      /    UNIT TESTS    \
                     /────────────────────\ 1. PLAIN STATE HOLDER UNIT TEST
                                               - Pure JVM Test (Chạy siêu tốc < 50ms)
```

- **Plain State Holder Unit Test:** Kiểm tra các hàm logic giao diện (UI Logic) độc lập trên máy ảo JVM mà không cần khởi động Android Emulator hay Compose Runtime.
- **ViewModel State Test:** Kiểm tra luồng phát sinh trạng thái (`StateFlow<ScreenUiState>`) dựa trên các tương tác nghiệp vụ bằng CashApp `Turbine`.
- **`StateRestorationTester`:** Công cụ kiểm thử đặc biệt do Google cung cấp, có khả năng mô phỏng chính xác chu trình hệ thống lưu trạng thái vào `Bundle`, tiêu hủy giao diện và khôi phục lại khi tiến trình được tái sinh (Process Death / Config Change).

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Cách thức vận hành của `StateRestorationTester`

`StateRestorationTester` hoạt động như một cỗ máy thời gian mô phỏng lại hành vi của Linux OS và Android Framework trong môi trường kiểm thử:

```
1. Khởi tạo nội dung Composable trong môi trường Test:
   restorationTester.setContent { CheckoutScreen(...) }
          │
          ▼
2. Người dùng thao tác nhập liệu trên UI (nhập tên, email):
   composeTestRule.onNodeWithText("Họ và tên").performTextInput("Nguyễn Văn A")
          │
          ▼
3. KÍCH HOẠT GIẢ LẬP PROCESS DEATH:
   restorationTester.emulateSavedInstanceStateRestore()
          │
          ├─► Bước 3.1: Gọi SavedStateRegistry lưu toàn bộ State vào mock Bundle
          ├─► Bước 3.2: Hủy hoàn toàn Composition hiện tại (disposeContent)
          ├─► Bước 3.3: Tạo lại Composition mới hoàn toàn từ đầu
          └─► Bước 3.4: Bơm lại mock Bundle vào Composition mới
          │
          ▼
4. KIỂM CHỨNG XÁC NHẬN:
   composeTestRule.onNodeWithText("Nguyễn Văn A").assertIsDisplayed()
   (Dữ liệu vẫn còn nguyên vẹn sau khi "chết" và sống lại!)
```

---

## 3. Bài toán Thực tế (Case Study): Quy trình Thanh toán (Checkout Flow)

Để minh họa toàn bộ các kiến thức từ Bài 01 đến Bài 06, ta sẽ xây dựng một Case Study thực tế chuẩn Clean Architecture: **Màn hình Đặt hàng & Thanh toán (Checkout Flow)** đáp ứng các tiêu chuẩn khắt khe:

1. **Screen UI State:** Quản lý bởi `CheckoutViewModel` kết hợp `SavedStateHandle` để không mất bước đang thanh toán khi bị kill ngầm.
2. **UI Element State:** Đóng gói trong Plain State Holder `CheckoutUiStateHolder` để điều khiển hiển thị BottomSheet mã giảm giá và thông báo lỗi.
3. **Stateless UI:** Giao diện phân tách thành các component thuần túy, dễ dàng xem trước với `@Preview` và kiểm thử.
4. **Bảo vệ dữ liệu tuyệt đối:** Sử dụng `@Parcelize` và `rememberSaveable`.

---

## 4. Thực hành tốt nhất & Cảnh báo (Best Practices & Anti-patterns)

> [!TIP]
> 1. **Tách biệt kiểm thử:** Đừng cố gắng kiểm thử toàn bộ nghiệp vụ phức tạp thông qua Compose UI Test. Hãy viết Unit Test thuần túy cho ViewModel và Plain State Holder trước (tốc độ thực thi chỉ vài mili-giây), sau đó chỉ dùng Compose Test để xác minh hành vi render và State Restoration.
> 2. **Luôn gọi `composeTestRule.waitForIdle()`:** Đảm bảo tất cả các hoạt ảnh (Animations) và Recompositions đã ổn định trước khi thực hiện các câu lệnh kiểm chứng (`assert`).

---

## 5. Mã nguồn Thực tế (Implementation)

### 5.1 Model Trạng thái & Plain State Holder

```kotlin
package com.example.compose.state.testing

import android.os.Parcelable
import androidx.compose.material3.SnackbarHostState
import androidx.compose.runtime.*
import androidx.compose.runtime.saveable.rememberSaveable
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.launch
import kotlinx.parcelize.Parcelize

// Model dữ liệu giỏ hàng được Parcelize để sống qua Process Death
@Parcelize
data class CheckoutFormState(
    val recipientName: String = "",
    val deliveryAddress: String = "",
    val discountCode: String = ""
) : Parcelable

/**
 * PLAIN STATE HOLDER: Quản lý logic giao diện thuần túy
 */
class CheckoutUiStateHolder(
    val snackbarHostState: SnackbarHostState,
    val coroutineScope: CoroutineScope
) {
    var isDiscountSheetVisible by mutableStateOf(false)
        private set

    fun openDiscountSheet() {
        isDiscountSheetVisible = true
    }

    fun closeDiscountSheet() {
        isDiscountSheetVisible = false
    }

    fun showErrorMessage(message: String) {
        coroutineScope.launch {
            snackbarHostState.showSnackbar(message)
        }
    }
}
```

### 5.2 ViewModel tích hợp `SavedStateHandle`

```kotlin
package com.example.compose.state.testing

import androidx.lifecycle.SavedStateHandle
import androidx.lifecycle.ViewModel
import kotlinx.coroutines.flow.StateFlow

class CheckoutViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    companion object {
        private const val KEY_STEP = "checkout_current_step"
    }

    // Lưu trữ bước thanh toán hiện tại (1: Địa chỉ, 2: Thanh toán, 3: Thành công)
    val currentStep: StateFlow<Int> = savedStateHandle.getStateFlow(KEY_STEP, 1)

    fun nextStep() {
        val next = currentStep.value + 1
        if (next <= 3) {
            savedStateHandle[KEY_STEP] = next
        }
    }

    fun previousStep() {
        val prev = currentStep.value - 1
        if (prev >= 1) {
            savedStateHandle[KEY_STEP] = prev
        }
    }
}
```

### 5.3 Màn hình Stateless & Stateful Checkout

```kotlin
package com.example.compose.state.testing

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.runtime.saveable.rememberSaveable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun CheckoutScreen(
    currentStep: Int,
    formState: CheckoutFormState,
    onFormChange: (CheckoutFormState) -> Unit,
    onNextClicked: () -> Unit,
    modifier: Modifier = Modifier
) {
    Column(
        modifier = modifier
            .fillMaxSize()
            .padding(16.dp)
    ) {
        Text(
            text = "Tiến trình thanh toán: Bước $currentStep / 3",
            style = MaterialTheme.typography.titleLarge
        )

        Spacer(modifier = Modifier.height(16.dp))

        OutlinedTextField(
            value = formState.recipientName,
            onValueChange = { onFormChange(formState.copy(recipientName = it)) },
            label = { Text("Họ và tên người nhận") },
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(modifier = Modifier.height(12.dp))

        OutlinedTextField(
            value = formState.deliveryAddress,
            onValueChange = { onFormChange(formState.copy(deliveryAddress = it)) },
            label = { Text("Địa chỉ giao hàng") },
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(modifier = Modifier.height(24.dp))

        Button(
            onClick = onNextClicked,
            enabled = formState.recipientName.isNotBlank() && formState.deliveryAddress.isNotBlank(),
            modifier = Modifier.fillMaxWidth()
        ) {
            Text(if (currentStep == 3) "Hoàn tất đặt hàng" else "Tiếp tục")
        }
    }
}
```

---

## 5.4 Bộ Kiểm Thử Tự Động Toàn Diện

### Test 1: Unit Test cho Plain State Holder (Chạy thuần trên JVM)

```kotlin
package com.example.compose.state.testing

import androidx.compose.material3.SnackbarHostState
import com.google.common.truth.Truth.assertThat
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.UnconfinedTestDispatcher
import kotlinx.coroutines.test.runTest
import org.junit.Test

@OptIn(ExperimentalCoroutinesApi::class)
class CheckoutUiStateHolderTest {

    @Test
    fun openAndCloseDiscountSheet_updatesStateCorrectly() = runTest {
        val testDispatcher = UnconfinedTestDispatcher(testScheduler)
        val stateHolder = CheckoutUiStateHolder(
            snackbarHostState = SnackbarHostState(),
            coroutineScope = this
        )

        assertThat(stateHolder.isDiscountSheetVisible).isFalse()

        stateHolder.openDiscountSheet()
        assertThat(stateHolder.isDiscountSheetVisible).isTrue()

        stateHolder.closeDiscountSheet()
        assertThat(stateHolder.isDiscountSheetVisible).isFalse()
    }
}
```

### Test 2: UI Test kiểm tra khả năng phục hồi State qua Process Death

```kotlin
package com.example.compose.state.testing

import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.saveable.rememberSaveable
import androidx.compose.runtime.setValue
import androidx.compose.ui.test.*
import androidx.compose.ui.test.junit4.StateRestorationTester
import androidx.compose.ui.test.junit4.createComposeRule
import org.junit.Rule
import org.junit.Test

class CheckoutStateRestorationTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun recipientInfo_survivesProcessDeathAndRestoration() {
        // 1. Khởi tạo StateRestorationTester
        val restorationTester = StateRestorationTester(composeTestRule)

        restorationTester.setContent {
            var formState by rememberSaveable { mutableStateOf(CheckoutFormState()) }

            CheckoutScreen(
                currentStep = 1,
                formState = formState,
                onFormChange = { formState = it },
                onNextClicked = {}
            )
        }

        // 2. Thao tác người dùng nhập dữ liệu
        val testName = "Trần Hoàng Long"
        val testAddress = "123 Nguyễn Huệ, Quận 1, TP.HCM"

        composeTestRule.onNodeWithText("Họ và tên người nhận")
            .performTextInput(testName)

        composeTestRule.onNodeWithText("Địa chỉ giao hàng")
            .performTextInput(testAddress)

        // Xác nhận dữ liệu đã hiển thị đúng trên giao diện
        composeTestRule.onNodeWithText(testName).assertIsDisplayed()
        composeTestRule.onNodeWithText(testAddress).assertIsDisplayed()

        // 3. GIẢ LẬP TIẾN TRÌNH BỊ HỆ THỐNG KILL NGẦM & KHÔI PHỤC LẠI
        restorationTester.emulateSavedInstanceStateRestore()

        // 4. KIỂM CHỨNG: Dữ liệu vẫn còn nguyên vẹn 100%!
        composeTestRule.onNodeWithText(testName).assertIsDisplayed()
        composeTestRule.onNodeWithText(testAddress).assertIsDisplayed()
    }
}
```

---

## 6. Các câu hỏi thực tế thường gặp & Xử lý sự cố (FAQ & Troubleshooting)

### Q1: Tại sao nên sử dụng `StateRestorationTester` thay vì gọi lệnh ADB thủ công khi chạy CI/CD?
- **Trả lời:** Lệnh ADB (`am kill`) yêu cầu phải cài app lên thiết bị thật hoặc Emulator chạy ngầm, mất nhiều phút thiết lập và khó tích hợp vào luồng Pull Request tự động.
- `StateRestorationTester` chạy trực tiếp trong bộ kiểm thử UI Test (kể cả với Robolectric trên máy chủ CI). Nó kiểm chứng chính xác hợp đồng lưu/đọc của `SavedStateRegistry` và `Bundle` chỉ trong vài trăm mili-giây, phát hiện tức thì các lỗi quên gắn `@Parcelize` hoặc thiếu `Saver`.

### Q2: Khi nào nên chọn Robolectric thay vì Android Instrumented Test cho Compose State?
- **Robolectric:** Chạy trên máy tính phát triển (JVM), không cần mở Emulator. Cực kỳ thích hợp cho các bài kiểm thử xác minh logic hiển thị, chuyển đổi State, và kiểm tra `StateRestorationTester` với tốc độ rất nhanh.
- **Instrumented Test (Thiết bị thật):** Bắt buộc khi cần đo đạc hiệu năng thực tế (FPS, đếm số lần Recomposition thực tế trên GPU phần cứng, kiểm tra độ mượt của cử chỉ vuốt chạm màn hình).
