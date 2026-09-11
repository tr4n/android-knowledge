# Bài 14 — Compose Navigation: Type-Safe, Deep Links & Back Stack

> **Module:** 4 — Jetpack Compose Advanced  
> **Prerequisite:** Module 3 — Jetpack Compose Foundation hoàn chỉnh  
> **Official Docs:**
> - [Navigation Compose Overview — Android Developers](https://developer.android.com/develop/ui/compose/navigation)
> - [Type safety in Navigation Compose (Navigation 2.8+)](https://developer.android.com/guide/navigation/design/type-safety)
> - [Multi-stack navigation in Compose](https://developer.android.com/guide/navigation/backstack/multi-back-stacks)

---

## 1. Định nghĩa & Thuật ngữ chuẩn (Definition)

**Navigation Compose** là giải pháp điều hướng màn hình chính thức của Google cho các ứng dụng Jetpack Compose. Bắt đầu từ **Navigation 2.8+**, Android đã chính thức giới thiệu cơ chế **Type-Safe Navigation (Điều hướng an toàn kiểu dữ liệu)** dựa trên thư viện **Kotlinx Serialization**, loại bỏ hoàn toàn các chuỗi String URL lỏng lẻo dễ gây lỗi runtime.

```
┌────────────────────────────────────────────────────────────────────────┐
│                     TYPE-SAFE NAVIGATION ARCHITECTURE                  │
│                                                                        │
│   1. Routes Definition (@Serializable):                                │
│      @Serializable object HomeRoute                                    │
│      @Serializable data class DetailRoute(val id: String, val page: Int)│
│                                                                        │
│   2. NavHost (Container quản lý hiển thị):                             │
│      NavHost(navController, startDestination = HomeRoute) {            │
│          composable<HomeRoute> { HomeScreen(...) }                     │
│          composable<DetailRoute> { backStackEntry ->                   │
│              val route: DetailRoute = backStackEntry.toRoute()         │
│              DetailScreen(id = route.id)                               │
│          }                                                             │
│      }                                                                 │
│                                                                        │
│   3. NavController (Bộ điều phối lệnh chuyển trang):                   │
│      navController.navigate(DetailRoute(id = "P100", page = 1))        │
└────────────────────────────────────────────────────────────────────────┘
```

### 1.1 Các thuật ngữ then chốt

- **`NavController`**: Đối tượng trung tâm theo dõi ngăn xếp cuộc gọi màn hình (**Back Stack**) và thực thi các lệnh điều hướng (`navigate`, `popBackStack`).
- **`NavHost`**: Thành phần Composable đóng vai trò như một viewport container, hiển thị Destination hiện tại tương ứng với đỉnh của Back Stack.
- **`NavGraph`**: Sơ đồ định tuyến tổng thể gom các Destinations lại thành một hệ thống hoặc các đồ thị con (Nested Graphs).
- **`NavBackStackEntry`**: Một phần tử cụ thể nằm trong Back Stack, lưu giữ đối tượng Route, `SavedStateHandle`, và vòng đời riêng biệt (`LifecycleOwner`) của màn hình đó.

---

## 2. Bản chất & Cơ chế hoạt động (Under the Hood)

### 2.1 Cơ chế Type-Safe Route Serialization (Bên dưới tấm màn nhung)

Làm thế nào Navigation 2.8+ chuyển đổi một `@Serializable data class` thành điểm đến trên màn hình mà không cần định nghĩa chuỗi URL thủ công?

```
                    CƠ CHẾ PARSING ROUTE TYPE-SAFE
                    
@Serializable data class DetailRoute(val id: String, val count: Int = 1)
                                   │
                                   ▼ Compose Compiler & Kotlinx Serialization
          Sinh ra Route URI Template ngầm định:
          "com.example.DetailRoute/{id}?count={count}"
                                   │
┌──────────────────────────────────┴──────────────────────────────────┐
│ LỆNH GỌI: navController.navigate(DetailRoute(id = "ITEM_9", count = 3))│
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
                                   ▼
    1. Đóng gói tham số vào Android Bundle:
       Bundle: ["id" -> "ITEM_9", "count" -> 3]
    2. Đẩy NavBackStackEntry mới vào Back Stack
    3. Ở Destination nhận:
       val detail = backStackEntry.toRoute<DetailRoute>()
       (Giải mã Bundle ngược lại thành đối tượng Kotlin an toàn 100%!)
```

- Toàn bộ quá trình kiểm tra kiểu dữ liệu (`String`, `Int`, `Boolean`, `Float`) diễn ra tại **thời điểm biên dịch (Compile-time)**.
- Bạn không thể truyền thiếu một tham số bắt buộc hoặc truyền nhầm kiểu dữ liệu. Nếu sai, code sẽ báo lỗi đỏ ngay trong IDE!

---

### 2.2 Vòng đời của từng Destination trong `NavBackStackEntry`

Mỗi màn hình trong Back Stack không chỉ là một Composable đơn thuần, nó sở hữu một **vòng đời độc lập**:

```
                       VÒNG ĐỜI DESTINATION TRONG BACK STACK
                       
Màn hình A (Home) ──navigate──► Màn hình B (Detail)
        │                               │
        ▼                               ▼
Trở thành Destination bên dưới    Nằm ở đỉnh Back Stack (Top)
Lifecycle tụt xuống:              Lifecycle đạt trạng thái cao nhất:
Lifecycle.State.STARTED           Lifecycle.State.RESUMED
(Không bị DESTROYED!)             (Đang hiển thị và nhận tương tác)
        │                               │
        │                       Người dùng bấm nút Back
        │                               │
        ▼                               ▼
Được đưa lại lên đỉnh Stack        Bị POP khỏi Back Stack
Lifecycle quay lại:               Lifecycle rơi thẳng xuống:
Lifecycle.State.RESUMED           Lifecycle.State.DESTROYED
                                  (Giải phóng ViewModel và Memory!)
```

---

### 2.3 Cơ chế Multiple Back Stacks cho Bottom Navigation

Trong các ứng dụng có thanh điều hướng đáy (Bottom Navigation) với nhiều Tabs (Home, Search, Profile):
- Người dùng mong muốn khi đang ở sâu trong Tab Search (Search $\rightarrow$ Results $\rightarrow$ Detail), nếu họ chuyển sang Tab Home rồi quay lại Tab Search, **vị trí và ngăn xếp cũ của Tab Search vẫn phải được bảo toàn**!

```
               CƠ CHẾ MULTIPLE BACK STACKS (SAVE & RESTORE)

Tab Home Stack:       Tab Search Stack:       Tab Profile Stack:
┌──────────────┐     ┌──────────────┐        ┌──────────────┐
│ HomeMain     │     │ SearchDetail │ ◄ Top  │ ProfileMain  │
└──────────────┘     ├──────────────┤        └──────────────┘
                     │ SearchResults│
                     ├──────────────┤
                     │ SearchMain   │
                     └──────────────┘
Chuyển sang Tab Home:
navController.navigate(HomeRoute) {
    popUpTo(navController.graph.findStartDestination().id) {
        saveState = true //  LƯU TOÀN BỘ NGĂN XẾP CỦA SEARCH VÀO BỘ NHỚ ĐỆM!
    }
    restoreState = true  //  PHỤC HỒI LẠI ĐÚNG VỊ TRÍ CŨ NẾU TAB ĐÓ ĐÃ CÓ STATE!
    launchSingleTop = true
}
```

---

## 3. Giải quyết bài toán gì? (Problem Statement & Value)

### 3.1 Nỗi kinh hoàng của Fragment Transactions cũ

| Tiêu chí | Fragment Navigation cũ | Compose Navigation 2.8+ Type-Safe |
|---|---|---|
| **Cú pháp điều hướng** | Phức tạp, dễ crash: `fragmentManager.beginTransaction().replace(...).addToBackStack().commit()` | Thanh lịch, trực quan: `navController.navigate(DetailRoute(id))` |
| **An toàn kiểu (Type Safety)**| Rất kém. Truyền dữ liệu qua `Bundle` lỏng lẻo bằng String Keys (`bundle.putString("KEY_ID", id)`), gõ sai là nhận `null`. | **Tuyệt đối an toàn 100% tại compile-time** nhờ `@Serializable` data classes. |
| **Crash khi lưu State** | Hay gặp lỗi `IllegalStateException: Can not perform this action after onSaveInstanceState` khi gọi async. | Hoàn toàn biến mất: Compose Navigation tự đồng bộ với snapshot state. |
| **Tái cấu trúc (Refactoring)** | Đổi tên tham số phải tìm kiếm String trong toàn bộ project để sửa thủ công. | Dùng tính năng Rename của IDE an toàn trên toàn bộ codebase. |

---

## 4. Ứng dụng thực tế & Best Practices (Real-world Use Cases)

### 4.1 Quy tắc kiến trúc vàng của Google

#### 1. Tuyệt đối không truyền `NavController` vào Composable con sâu bên trong
`NavController` là một đối tượng cấp cao của tầng điều hướng. Nếu truyền nó vào các Composable UI sâu bên trong:
- Phá vỡ tính độc lập của UI (Không thể viết `@Preview` vì thiếu NavController).
- Không thể viết Unit Test cho Composable đó một cách cô lập.
- **Giải pháp:** Luôn sử dụng **Event Hoisting** (truyền lambda callbacks ra ngoài):

```kotlin
// SAI LẦM:
@Composable
fun ProductItem(product: Product, navController: NavController) {
    Button(onClick = { navController.navigate(DetailRoute(product.id)) }) { ... }
}

// CHUẨN MỰC:
@Composable
fun ProductItem(product: Product, onProductClick: (String) -> Unit) {
    Button(onClick = { onProductClick(product.id) }) { ... }
}
```

#### 2. Trích xuất tham số trực tiếp trong ViewModel với `SavedStateHandle`
Trong Navigation 2.8+, bạn không cần trích xuất tham số ở UI rồi truyền vào ViewModel. ViewModel có thể **tự động đọc Route Type-Safe** thông qua extension `toRoute<T>()`:

```kotlin
@HiltViewModel
class ProductDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle
) : ViewModel() {

    // TRÍCH XUẤT TYPE-SAFE 100% TRỰC TIẾP TỪ SAVEDSTATEHANDLE!
    val route = savedStateHandle.toRoute<ProductDetailRoute>()
    val productId: String = route.productId
}
```

#### 3. Cấm truyền Object dữ liệu lớn qua Navigation Route
Nhiều lập trình viên cố gắng serialize một đối tượng `User` có 50 thuộc tính hoặc cả một `Cart` phức tạp qua Route.
> **Lời khuyên của Google Architect:** **Chỉ truyền các định danh tối thiểu (Unique IDs)** qua Navigation Route. Hãy để màn hình đích sử dụng ID đó và yêu cầu Repository truy xuất dữ liệu từ Local DB hoặc Cache. Điều này đảm bảo dữ liệu luôn là phiên bản mới nhất và không vi phạm giới hạn kích thước của Android Bundle!

---

### 4.2 Xử lý luồng Đăng nhập (Auth Flow) với `popUpTo` và `inclusive`

Khi người dùng đăng nhập thành công và chuyển vào màn hình chính (`HomeRoute`), nếu họ bấm nút Back vật lý, họ **không bao giờ được phép quay lại màn hình Login**:

```kotlin
navController.navigate(HomeRoute) {
    // Xóa LoginRoute ra khỏi Back Stack hoàn toàn!
    popUpTo<LoginRoute> {
        inclusive = true // Xóa luôn cả LoginRoute
    }
}
```

---

## 5. Hướng dẫn triển khai từng bước (Step-by-Step Implementation)

Xây dựng hệ thống điều hướng hoàn chỉnh cho ứng dụng **E-Commerce Store**:
- Định nghĩa Type-Safe Routes bằng `@Serializable`.
- Cấu hình `NavHost` đa màn hình: `HomeRoute`, `ProductDetailRoute`, `CartRoute`.
- Thanh điều hướng Bottom Navigation Bar hỗ trợ Multiple Back Stacks.
- Hỗ trợ **Deep Link** mở thẳng vào sản phẩm từ Web URL.
- Viết Unit Test điều hướng với `TestNavHostController`.

### Bước 1: Khai báo Dependencies

```kotlin
// build.gradle.kts (Module: app)
plugins {
    alias(libs.plugins.kotlin.serialization)
}

dependencies {
    // Navigation Compose 2.8+ Type-Safe
    implementation("androidx.navigation:navigation-compose:2.8.0")
    // Kotlinx Serialization JSON
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.1")
}
```

---

### Bước 2: Định nghĩa tập hợp Type-Safe Routes

```kotlin
// Các màn hình chính trong ứng dụng
sealed interface AppRoute {
    @Serializable
    data object Home : AppRoute

    @Serializable
    data class ProductDetail(val productId: String, val category: String = "general") : AppRoute

    @Serializable
    data object Cart : AppRoute

    @Serializable
    data object Profile : AppRoute
}
```

---

### Bước 3: Triển khai `AppNavHost` với Deep Links

```kotlin
@Composable
fun AppNavHost(
    navController: NavHostController,
    modifier: Modifier = Modifier
) {
    NavHost(
        navController = navController,
        startDestination = AppRoute.Home,
        modifier = modifier
    ) {
        // 1. Màn hình Home
        composable<AppRoute.Home> {
            HomeScreen(
                onProductClick = { id ->
                    navController.navigate(AppRoute.ProductDetail(productId = id))
                }
            )
        }

        // 2. Màn hình Chi tiết sản phẩm với DEEP LINK
        composable<AppRoute.ProductDetail>(
            deepLinks = listOf(
                navDeepLink<AppRoute.ProductDetail>(
                    basePath = "https://mycoolstore.com/products"
                )
            )
        ) { backStackEntry ->
            // Tự động giải mã Route Type-Safe
            val route = backStackEntry.toRoute<AppRoute.ProductDetail>()
            ProductDetailScreen(
                productId = route.productId,
                category = route.category,
                onBackClick = { navController.popBackStack() },
                onAddToCart = { navController.navigate(AppRoute.Cart) }
            )
        }

        // 3. Màn hình Giỏ hàng
        composable<AppRoute.Cart> {
            CartScreen(
                onCheckoutSuccess = {
                    navController.navigate(AppRoute.Home) {
                        popUpTo<AppRoute.Home> { inclusive = false }
                    }
                }
            )
        }

        // 4. Màn hình Hồ sơ cá nhân
        composable<AppRoute.Profile> {
            ProfileScreen()
        }
    }
}
```

---

### Bước 4: Bottom Navigation Bar với Multiple Back Stacks

```kotlin
data class BottomNavItem(val label: String, val icon: ImageVector, val route: Any)

@Composable
fun MainScaffold() {
    val navController = rememberNavController()
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    val currentDestination = navBackStackEntry?.destination

    val bottomItems = listOf(
        BottomNavItem("Trang chủ", Icons.Default.Home, AppRoute.Home),
        BottomNavItem("Giỏ hàng", Icons.Default.ShoppingCart, AppRoute.Cart),
        BottomNavItem("Cá nhân", Icons.Default.Person, AppRoute.Profile)
    )

    Scaffold(
        bottomBar = {
            NavigationBar {
                bottomItems.forEach { item ->
                    // Kiểm tra xem Route hiện tại có thuộc về Tab này không
                    val isSelected = currentDestination?.hasRoute(item.route::class) == true

                    NavigationBarItem(
                        selected = isSelected,
                        onClick = {
                            navController.navigate(item.route) {
                                // Pop về root destination của graph để tránh tích tụ stack
                                popUpTo(navController.graph.findStartDestination().id) {
                                    saveState = true //  LƯU STATE CỦA TAB CŨ
                                }
                                launchSingleTop = true
                                restoreState = true  //  PHỤC HỒI STATE CỦA TAB MỚI
                            }
                        },
                        icon = { Icon(item.icon, contentDescription = item.label) },
                        label = { Text(item.label) }
                    )
                }
            }
        }
    ) { padding ->
        AppNavHost(
            navController = navController,
            modifier = Modifier.padding(padding)
        )
    }
}
```

---

### Bước 5: Viết Unit Test điều hướng với `TestNavHostController`

```kotlin
@RunWith(AndroidJUnit4::class)
class NavigationTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    private lateinit var navController: TestNavHostController

    @Before
    fun setupAppNavHost() {
        composeTestRule.setContent {
            navController = TestNavHostController(LocalContext.current)
            navController.navigatorProvider.addNavigator(ComposeNavigator())
            AppNavHost(navController = navController)
        }
    }

    @Test
    fun appNavHost_verifyStartDestination_isHome() {
        // Kiểm tra destination khởi đầu
        assertEquals(
            AppRoute.Home::class.qualifiedName,
            navController.currentBackStackEntry?.destination?.route
        )
    }

    @Test
    fun appNavHost_navigateToDetail_updatesRouteWithArguments() {
        composeTestRule.runOnUiThread {
            navController.navigate(AppRoute.ProductDetail(productId = "PROD_99", category = "phones"))
        }

        // Xác nhận màn hình hiện tại là ProductDetail với đúng arguments
        val currentEntry = navController.currentBackStackEntry!!
        val route = currentEntry.toRoute<AppRoute.ProductDetail>()
        
        assertEquals("PROD_99", route.productId)
        assertEquals("phones", route.category)
    }
}
```

---

## 6. Câu hỏi thường gặp & Gỡ rối thực tế (FAQ & Troubleshooting)

### 6.1 Câu hỏi phỏng vấn Senior/Staff Android Architect

#### Q1: Làm thế nào để chia sẻ chung một ViewModel (Shared ViewModel) giữa 2 màn hình liền kề trong Navigation Compose?
**Trả lời chuẩn bản chất:**
Mặc định hàm `hiltViewModel()` sẽ gắn ViewModel với `NavBackStackEntry` của màn hình hiện tại. Nếu muốn chia sẻ ViewModel giữa 2 màn hình (ví dụ: Flow Đặt hàng gồm `AddressScreen` và `PaymentScreen`):
1. Nhóm 2 màn hình này vào một Nested Navigation Graph (`navigation<OrderFlowGraph> { ... }`).
2. Trong mỗi màn hình, lấy `NavBackStackEntry` của graph cha:
```kotlin
val parentEntry = remember(backStackEntry) {
    navController.getBackStackEntry<OrderFlowGraph>()
}
val sharedViewModel: OrderSharedViewModel = hiltViewModel(parentEntry)
```
Khi đó, cả 2 màn hình sẽ cùng truy cập vào một instance ViewModel duy nhất và ViewModel này chỉ bị tiêu hủy khi toàn bộ Order Flow kết thúc!

#### Q2: Sự khác nhau giữa `hasRoute(Route::class)` và việc so sánh `destination.route` truyền thống?
**Trả lời chuẩn bản chất:**
Trong Navigation 2.8+, `destination.route` trả về chuỗi URI Pattern bên dưới (ví dụ `"com.example.ProductDetail/{productId}"`). Việc so sánh chuỗi rất dễ bị sai lệch khi có tham số tùy chọn hoặc URL query parameters. Hàm extension `hasRoute(Route::class)` kiểm tra chính xác cấu trúc kiểu của Route dựa trên KClass metadata, đảm bảo nhận diện chính xác 100% xem destination có thuộc về Route đó hay không.

#### Q3: Deep Link trong Type-Safe Navigation xử lý như thế nào nếu người dùng truyền sai định dạng dữ liệu (ví dụ Int nhưng truyền String)?
**Trả lời chuẩn bản chất:**
Navigation 2.8+ sẽ tự động phân tích cú pháp Deep Link URI dựa trên các serializer của Kotlinx Serialization. Nếu tham số trong URL không thể parse được thành kiểu dữ liệu khai báo trong `@Serializable data class`, quá trình match Deep Link sẽ bị coi là thất bại (Failed to match). Hệ thống sẽ chuyển tiếp sang Deep Link tiếp theo hoặc mở start destination thay vì làm sập ứng dụng với lỗi `NumberFormatException` như trước đây!

---

### 6.2 Lỗi Runtime thường gặp & Hướng xử lý

| Tình huống lỗi | Nguyên nhân gốc rễ | Cách xử lý triệt để |
|---|---|---|
| **Crash: `SerializationException: Serializer for class ... is not found`** | Quên gắn annotation `@Serializable` lên class / object định nghĩa Route. | Thêm `@Serializable` lên trên định nghĩa Route và bật plugin `kotlinx.serialization`. |
| **Bấm nút Back bị quay lại màn hình Login** | Quên cấu hình `popUpTo` khi navigate từ Login sang Home. | Bổ sung `popUpTo<LoginRoute> { inclusive = true }` trong block điều hướng. |
| **Mất trạng thái cuộn của Tab khi bấm BottomNavigation** | Thiếu `saveState = true` và `restoreState = true` khi gọi `navigate()`. | Cấu hình đầy đủ bộ ba `popUpTo { saveState = true }`, `restoreState = true`, `launchSingleTop = true`. |

---

*Bài trước: [13 — Side Effects trong Compose: LaunchedEffect, DisposableEffect](../module-3-compose-foundation/13-side-effects.md)*  
*Bài tiếp theo: [15 — Compose Animation: animate*AsState, AnimatedVisibility](15-animation.md)*
