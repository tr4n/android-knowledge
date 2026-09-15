# Bài 03 — Kotlin Flows trên Android (Kotlin Flows on Android)

> **Tài liệu tham chiếu gốc:** [Kotlin flows on Android — Android Developers](https://developer.android.com/kotlin/flow?hl=vi)  
> **Áp dụng:** Kotlin 2.0+, `kotlinx.coroutines:1.11.0`, Android Jetpack Architecture  
> **Mục tiêu:** Nắm vững bản chất luồng dữ liệu bất đồng bộ (Asynchronous Data Stream), 3 thực thể Producer - Intermediary - Consumer, cơ chế Cold Flow, các toán tử biến đổi và gom luồng, nguyên lý Exception Transparency với toán tử `catch`, và quy tắc bảo toàn ngữ cảnh (Context Preservation) với toán tử `flowOn`.

---

## 1. Khái niệm: Asynchronous Data Stream trong Android

Trong khi một hàm tạm dừng (`suspend function`) chỉ trả về một giá trị duy nhất (one-shot value) một cách bất đồng bộ, thì trong thực tế phát triển ứng dụng Android, bạn thường xuyên phải xử lý các luồng dữ liệu liên tục thay đổi theo thời gian:
- Cập nhật tọa độ vị trí GPS theo thời gian thực từ phần cứng.
- Dữ liệu tin nhắn mới hoặc thông báo đẩy từ WebSocket / Firebase.
- Dữ liệu bảng Database được cập nhật tự động qua thư viện Jetpack Room.
- Luồng sự kiện người dùng gõ phím trên thanh tìm kiếm (Search-as-you-type).

Một **Flow (Luồng)** trong Kotlin đại diện cho một luồng dữ liệu có thể được tính toán và phát ra một cách bất đồng bộ. Flow được xây dựng ngay trên nền tảng của Coroutines, do đó nó thừa hưởng toàn bộ ưu điểm về quản lý tài nguyên, hủy bỏ tự động và tính an toàn luồng (Main-safety).

---

## 2. Ba Thực thể Cốt lõi của một Flow Pipeline

Một chuỗi xử lý Flow chuẩn luôn bao gồm 3 thực thể tham gia:

```
┌───────────────────────────┐      ┌───────────────────────────┐      ┌───────────────────────────┐
│     PRODUCER (Sản xuất)   │ ───► │  INTERMEDIARIES (Biến đổi)│ ───► │    CONSUMER (Tiêu thụ)    │
│  - flow { emit(...) }     │      │  - map, filter, debounce  │      │  - collect { ... }        │
│  - Room Database Query    │      │  - catch, flowOn          │      │  - Jetpack Compose UI     │
└───────────────────────────┘      └───────────────────────────┘      └───────────────────────────┘
```

1. **Nhà sản xuất (Producer):** Tạo và phát dữ liệu vào luồng. Nhờ coroutines, việc sản xuất dữ liệu có thể diễn ra bất đồng bộ mà không cần block thread.
2. **(Tùy chọn) Các toán tử trung gian (Intermediaries):** Biến đổi từng giá trị được phát ra hoặc điều chỉnh cách luồng vận hành (lọc, chuyển đổi kiểu, hoãn thời gian, đổi thread).
3. **Bên tiêu thụ (Consumer):** Thu thập (`collect`) các giá trị từ luồng để hiển thị lên UI hoặc thực thi logic nghiệp vụ.

---

## 3. Tạo một Flow (Creating a Flow)

Để tạo một Flow cơ bản, bạn sử dụng coroutine builder `flow { ... }`. Bên trong khối mã này, bạn sử dụng hàm `emit()` để phát dữ liệu:

```kotlin
class NewsRemoteDataSource(
    private val newsApi: NewsApi,
    private val refreshIntervalMs: Long = 5000L
) {
    // Tạo một Flow định kỳ lấy tin tức mới mỗi 5 giây
    val latestNews: Flow<List<ArticleHeadline>> = flow {
        while (true) {
            val latestNews = newsApi.fetchLatestNews()
            emit(latestNews) // Phát dữ liệu vào luồng
            delay(refreshIntervalMs) // Tạm dừng bất đồng bộ 5 giây
        }
    }
}
```

### Đặc tính Cold Flow (Luồng lười / Lazy):
> [!NOTE]
> Flow được tạo bởi builder `flow { ... }` là một **Cold Flow (Luồng lạnh)**.  
> Mã bên trong khối `flow { ... }` **sẽ không hề chạy** cho đến khi có một bên tiêu thụ bắt đầu gọi hàm thu thập (`collect`).  
> Nếu có 2 collectors cùng thu thập độc lập, khối code producer sẽ chạy 2 lần riêng biệt cho 2 collectors đó.

---

## 4. Biến đổi Luồng Dữ liệu (Intermediate Operators)

Các toán tử trung gian nhận đầu vào là một `Flow`, áp dụng các phép biến đổi và trả về một `Flow` mới. Các toán tử này cũng hoàn toàn "lười" (lazy) và không tự kích hoạt việc thu thập.

### 4.1. `map` — Biến đổi dữ liệu
Chuyển đổi từng phần tử sang một kiểu dữ liệu khác:

```kotlin
class NewsRepository(private val remoteDataSource: NewsRemoteDataSource) {
    val favoriteNews: Flow<List<Article>> = remoteDataSource.latestNews
        .map { headlines -> 
            // Chuyển đổi từ DTO sang Model nội bộ và đánh dấu tin yêu thích
            headlines.map { it.toArticle(isFavorite = checkFavorite(it.id)) }
        }
}
```

### 4.2. `filter` — Lọc dữ liệu
Chỉ cho phép các giá trị thỏa mãn điều kiện đi tiếp xuống downstream:

```kotlin
val urgentNews = repository.favoriteNews
    .map { list -> list.filter { it.isBreakingNews } }
```

### 4.3. `transform` — Tùy biến phát dữ liệu linh hoạt
Cho phép bạn tự do gọi `emit()` nhiều lần hoặc không phát gì cho mỗi phần tử:

```kotlin
val processedStream = flowOf(1, 2, 3).transform { value ->
    emit("Bắt đầu xử lý: $value")
    emit("Kết quả nhân đôi: ${value * 2}")
}
```

---

## 5. Thu thập Dữ liệu từ Flow (Terminal Operators)

Để kích hoạt luồng và bắt đầu nhận các giá trị, bạn phải sử dụng một **toán tử kết thúc (Terminal operator)**. Tất cả các terminal operator đều là `suspend functions`.

### 5.1. Toán tử `collect`
Toán tử cơ bản nhất để lắng nghe từng phần tử được phát ra:

```kotlin
class NewsViewModel(
    private val newsRepository: NewsRepository
) : ViewModel() {

    fun startListeningNews() {
        viewModelScope.launch {
            newsRepository.favoriteNews.collect { articles ->
                // Được gọi mỗi khi có dữ liệu mới phát ra
                _uiState.value = NewsUiState.Success(articles)
            }
        }
    }
}
```

### 5.2. Các Terminal Operators phổ biến khác:
- **`first()` / `firstOrNull()`:** Thu thập đúng giá trị đầu tiên rồi lập tức hủy bỏ luồng (rất hữu ích khi chỉ muốn lấy một snapshot dữ liệu).
- **`toList()` / `toSet()`:** Thu thập toàn bộ các giá trị phát ra cho đến khi luồng kết thúc và gom vào một `List` hoặc `Set`.
- **`reduce()` / `fold()`:** Tích lũy các phần tử (ví dụ: tính tổng các giá trị phát ra).

---

## 6. Xử lý Ngoại lệ và Tính Minh bạch Ngoại lệ (Exception Transparency)

Khi làm việc với Flow, ngoại lệ có thể xảy ra ở tầng Producer (ví dụ: mất mạng khi gọi API), ở các toán tử trung gian, hoặc ở bên Consumer (`collect`).

Kotlin Flow cung cấp toán tử `catch` để bắt ngoại lệ theo nguyên tắc **Exception Transparency (Tính minh bạch ngoại lệ)**:

> [!IMPORTANT]
> **Quy tắc:**  
> Toán tử `catch` **chỉ bắt các ngoại lệ xảy ra ở Upstream** (tức là các toán tử hoặc producer nằm phía trên nó trong chuỗi pipeline).  
> Nó **không bao giờ** bắt các ngoại lệ xảy ra ở Downstream (bên dưới nó, bao gồm cả khối mã bên trong `collect { ... }`).

```kotlin
class NewsRepository(private val remoteDataSource: NewsRemoteDataSource) {
    val newsFlow: Flow<List<Article>> = remoteDataSource.latestNews
        .map { transformArticles(it) }
        .catch { exception ->
            // Bắt lỗi từ latestNews hoặc transformArticles
            Log.e("NewsRepo", "Lỗi tải tin tức", exception)
            // Phát ra giá trị mặc định thay thế hoặc phát danh sách rỗng
            emit(emptyList())
        }
}
```

### Cách xử lý lỗi bên trong Consumer (`collect`):
Nếu có ngoại lệ xảy ra bên trong khối `collect`, hãy dùng khối `try-catch` truyền thống bọc bên ngoài:

```kotlin
viewModelScope.launch {
    try {
        newsRepository.newsFlow.collect { articles ->
            renderToUI(articles) // Nếu hàm này ném lỗi, try-catch này sẽ bắt
        }
    } catch (e: Exception) {
        showErrorDialog(e.message)
    }
}
```

---

## 7. Chuyển đổi Ngữ cảnh Luồng với `flowOn`

### Vấn đề: Vi phạm bảo toàn ngữ cảnh (Context Preservation)
Mặc định, code trong `flow { ... }` sẽ chạy trên chính `CoroutineContext` của coroutine đang gọi hàm `collect`. 

Nếu producer cần thực hiện I/O nặng nhưng bạn lại dùng `withContext(Dispatchers.IO)` bên trong `flow { ... }` để phát dữ liệu, Kotlin sẽ ném ngay lỗi ngoại lệ lúc runtime:
`IllegalStateException: Flow invariant is violated`!

```kotlin
// ❌ SAI LẦM: Không được dùng withContext để phát dữ liệu trong flow builder
val badFlow = flow {
    withContext(Dispatchers.IO) {
        val data = fetchFromNetwork()
        emit(data) // LỖI RUNTIME: Vi phạm Context Preservation!
    }
}
```

### Giải pháp chuẩn của Google: Sử dụng toán tử `flowOn`
Toán tử `flowOn` được thiết kế riêng để thay đổi ngữ cảnh cho toàn bộ các toán tử và producer **nằm phía trên nó (Upstream)**, trong khi Consumer bên dưới vẫn thu thập an toàn trên Main Thread:

```kotlin
class NewsRepository(private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO) {
    val newsStream: Flow<List<Article>> = flow {
        // Khối mã này sẽ chạy trên Dispatchers.IO nhờ có flowOn bên dưới
        while (true) {
            val data = apiService.getNews()
            emit(data)
            delay(10000)
        }
    }
    .flowOn(ioDispatcher) // ◄ Đổi ngữ cảnh cho Upstream thành Dispatchers.IO
}

// Bên Consumer:
viewModelScope.launch {
    // Thu thập an toàn trên Dispatchers.Main
    repository.newsStream.collect { news ->
        updateUi(news)
    }
}
```

```
Sơ đồ phân chia luồng của flowOn:
[Producer: flow { ... }] ────────► [Toán tử: map { ... }] ──┐
  (Chạy trên Dispatchers.IO)                                │  flowOn(Dispatchers.IO)
────────────────────────────────────────────────────────────┼────────────────────────
[Consumer: collect { ... }] ◄───────────────────────────────┘
  (Chạy trên Dispatchers.Main)
```

---

## 8. Tích hợp Flow với Jetpack Room Database

Jetpack Room hỗ trợ Flow như một công dân hạng nhất (first-class citizen) cho các câu lệnh truy vấn dữ liệu quan sát (`Observable Queries`).

```kotlin
@Dao
interface UserDao {
    // Trả về Flow: Room tự động chạy ngầm trên Background Thread
    // Mỗi khi bảng 'users' có thay đổi, Room tự động emit List mới nhất!
    @Query("SELECT * FROM users ORDER BY name ASC")
    fun getAllUsers(): Flow<List<UserEntity>>
}
```

Khi sử dụng Room với Flow:
1. Room tự động xử lý chuyển luồng I/O an toàn.
2. Bạn không cần phải gọi thủ công lại hàm truy vấn mỗi khi có dữ liệu được thêm mới hay xóa bỏ.
3. Flow tự động ngừng quan sát bảng database khi coroutine của bên thu thập (`collect`) bị hủy.
