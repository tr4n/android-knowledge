# Bài 03 — Kotlin Flows trên Android (Bản Dịch & Hệ Thống Hóa Chuẩn 1:1)

> **Tài liệu tham chiếu gốc:** [Kotlin flows on Android — Android Developers](https://developer.android.com/kotlin/flow?hl=vi)  
> **Áp dụng:** Kotlin 2.0+, `kotlinx.coroutines:1.11.0`, Android Jetpack Room & Firebase  
> **Mục tiêu:** Nắm vững toàn bộ tài liệu chính thức của Google: khái niệm luồng dữ liệu bất đồng bộ (Asynchronous Stream), 3 thực thể tham gia (Producer, Intermediary, Consumer), tạo flow định kỳ với `flow { emit(...) }`, sửa đổi luồng với toán tử trung gian, thu thập với `collect`, xử lý ngoại lệ theo nguyên lý Exception Transparency với `catch`, đổi ngữ cảnh với `flowOn`, tích hợp Jetpack Room, và đặc biệt là kỹ thuật chuyển đổi API callback thành flow với `callbackFlow` và `awaitClose`.

---

## 1. Giới thiệu: Luồng Dữ Liệu Bất Đồng Bộ (Asynchronous Stream of Data)

Trong coroutine, một **hàm tạm ngưng (`suspend function`)** chỉ có khả năng trả về một giá trị duy nhất một cách bất đồng bộ. Tuy nhiên, trong phát triển ứng dụng Android, bạn thường xuyên phải tiếp nhận và xử lý các bản cập nhật dữ liệu diễn ra liên tục theo thời gian (ví dụ: cập nhật trực tiếp từ cơ sở dữ liệu khi có bản ghi mới, luồng tin nhắn chat thời gian thực, hoặc tín hiệu định vị GPS).

Một **Flow (Luồng)** là một kiểu dữ liệu có thể phát ra **nhiều giá trị tuần tự một cách bất đồng bộ**. Flow được xây dựng trực tiếp trên nền tảng của Coroutines, do đó nó sở hữu toàn bộ các đặc tính ưu việt của coroutine: xử lý bất đồng bộ không gây nghẽn luồng, quản lý vòng đời chặt chẽ và tích hợp sẵn cơ chế hủy bỏ.

---

## 2. Ba Thực Thể Cốt Lõi của một Dòng Dữ Liệu

Một dòng dữ liệu Flow hoàn chỉnh bao gồm 3 thực thể tham gia với trách nhiệm rõ ràng:

```
┌────────────────────────────────┐      ┌────────────────────────────────┐      ┌────────────────────────────────┐
│      PRODUCER (Tạo ra)         │ ───► │    INTERMEDIARY (Trung gian)   │ ───► │      CONSUMER (Tiêu thụ)       │
│  - Tạo và phát dữ liệu vào luồng│      │  - Sửa đổi dữ liệu hoặc luồng  │      │  - Nhận và tiêu thụ các giá trị│
│  - flow { emit(...) }          │      │  - map, filter, catch, flowOn  │      │  - collect { ... }             │
│  - Room Database Query         │      │  - Thiết lập chuỗi thao tác     │      │  - Cập nhật giao diện UI       │
└────────────────────────────────┘      └────────────────────────────────┘      └────────────────────────────────┘
```

1. **Thực thể tạo (Producer):** Tạo ra dữ liệu được thêm vào dòng dữ liệu. Nhờ coroutine, flow có thể tạo dữ liệu theo cách không đồng bộ mà không cần chặn luồng.
2. **(Tùy chọn) Thực thể trung gian (Intermediary):** Có thể sửa đổi từng giá trị được phát vào dòng dữ liệu hoặc điều chỉnh cách dòng dữ liệu vận hành (lọc, chuyển đổi kiểu, đổi luồng thực thi).
3. **Thực thể tiêu thụ (Consumer):** Thu thập và tiêu thụ các giá trị trong dòng dữ liệu để cập nhật trạng thái hoặc hiển thị lên màn hình.

---

## 3. Tạo Flow (Creating a Flow)

Để tạo flow, hãy sử dụng các **API tạo flow**. Hàm tạo `flow { ... }` sẽ tạo một dòng dữ liệu mới, cho phép bạn phát các giá trị vào dòng dữ liệu theo cách thủ công thông qua hàm `emit`.

Trong ví dụ chính thức dưới đây từ Google, một nguồn dữ liệu (`NewsRemoteDataSource`) sẽ tự động tìm nạp tin tức mới nhất từ máy chủ theo một khoảng thời gian cố định:

```kotlin
class NewsRemoteDataSource(
    private val newsApi: NewsApi,
    private val refreshIntervalMs: Long = 5000
) {
    // Thuộc tính Flow phát ra danh sách bài báo mới nhất mỗi 5 giây
    val latestNews: Flow<List<ArticleHeadline>> = flow {
        while (true) {
            val latestNews = newsApi.fetchLatestNews()
            emit(latestNews) // Phát kết quả vào dòng dữ liệu
            delay(refreshIntervalMs) // Tạm ngưng coroutine trong khoảng thời gian quy định
        }
    }
}

// Giao diện gọi API mạng bằng hàm suspend
interface NewsApi {
    suspend fun fetchLatestNews(): List<ArticleHeadline>
}
```

### Các Đặc Tính Quan Trọng Của Hàm Tạo `flow`:
- **Thực thi tuần tự:** Mã bên trong khối `flow { ... }` được thực thi tuần tự từng dòng.
- **Tính chất luồng lạnh (Cold Flow):** 
  > [!NOTE]
  > Flow mang tính chất **nguội (cold)**. Nghĩa là đoạn mã bên trong khối `flow { ... }` **sẽ không hề chạy** cho đến khi có một bên tiêu thụ bắt đầu gọi hàm thu thập (`collect`). Mỗi khi có một bên thu thập mới, hàm tạo `flow` sẽ chạy lại độc lập từ đầu cho bên thu thập đó.
- **Hỗ trợ hàm suspend:** Bên trong khối `flow`, bạn có thể tự do gọi các hàm `suspend` khác (chẳng hạn như `newsApi.fetchLatestNews()` và `delay()`).

---

## 4. Sửa Đổi Dòng Dữ Liệu (Modifying the Stream)

Thực thể trung gian có thể sử dụng các **toán tử trung gian (intermediate operators)** để sửa đổi dòng dữ liệu mà không cần phải tiêu thụ ngay các giá trị trong đó.

Khi được áp dụng cho một dòng dữ liệu, các toán tử này không thực thi ngay lập tức; chúng chỉ thiết lập một **chuỗi thao tác (cold pipeline)** và sẽ chỉ thực sự chạy khi dòng dữ liệu được thu thập ở hạ nguồn.

Trong ví dụ dưới đây, tầng `NewsRepository` sử dụng toán tử trung gian `map` để chuyển đổi dữ liệu và lọc chỉ giữ lại những tin tức thuộc chủ đề yêu thích của người dùng:

```kotlin
class NewsRepository(
    private val newsRemoteDataSource: NewsRemoteDataSource,
    private val userDataUserDataSource: UserDataUserDataSource
) {
    // Sửa đổi dòng dữ liệu bằng toán tử trung gian
    val favoriteLatestNews: Flow<List<ArticleHeadline>> =
        newsRemoteDataSource.latestNews
            // Toán tử map lọc danh sách tin tức theo chủ đề yêu thích
            .map { news -> 
                news.filter { userDataUserDataSource.isFavoriteTopic(it.topic) } 
            }
            // Toán tử onEach thực hiện hành vi phụ (side-effect): lưu vào bộ nhớ cache
            .onEach { news -> 
                saveInCache(news) 
            }
}
```

Bạn có thể xâu chuỗi nhiều toán tử trung gian liên tiếp để tạo nên một quy trình biến đổi dữ liệu phức tạp trước khi chuyển tiếp cho tầng giao diện.

---

## 5. Thu Thập Dữ Liệu từ Flow (Collecting from a Flow)

Sử dụng một **toán tử đầu cuối (terminal operator)** để kích hoạt flow và bắt đầu lắng nghe các giá trị được phát ra. 

Toán tử đầu cuối cơ bản và phổ biến nhất là **`collect`**. Vì `collect` là một hàm tạm ngưng (`suspend function`), nó bắt buộc phải được thực thi bên trong một coroutine.

Trong ví dụ dưới đây, `LatestNewsViewModel` kích hoạt việc thu thập dữ liệu bằng cách khởi chạy một coroutine bên trong `viewModelScope`:

```kotlin
class LatestNewsViewModel(
    private val newsRepository: NewsRepository
) : ViewModel() {

    init {
        // Khởi tạo coroutine gắn liền với vòng đời của ViewModel
        viewModelScope.launch {
            // Kích hoạt dòng dữ liệu và bắt đầu lắng nghe các giá trị
            newsRepository.favoriteLatestNews.collect { favoriteNews ->
                // Cập nhật giao diện người dùng với danh sách tin tức mới nhất nhận được
                displayNews(favoriteNews)
            }
        }
    }

    private fun displayNews(news: List<ArticleHeadline>) {
        // Cập nhật UI
    }
}
```

### Cơ chế Tự động Hủy của `viewModelScope`:
Việc thu thập flow sẽ dừng lại khi coroutine chứa nó bị hủy bỏ. Khi `ViewModel` bị xóa khỏi bộ nhớ (sự kiện `onCleared()`), `viewModelScope` sẽ tự động bị hủy, kéo theo việc coroutine thu thập dừng lại và nhà sản xuất `latestNews` ở tầng `NewsRemoteDataSource` cũng sẽ tự động dừng vòng lặp `while(true)`, giải phóng hoàn toàn tài nguyên mạng.

---

## 6. Phát Hiện Ngoại Lệ Không Mong Muốn (Catching Unexpected Exceptions)

Mã triển khai thực thể tạo có thể đến từ một thư viện của bên thứ ba hoặc phát sinh các lỗi mạng ngoài dự kiến. Để xử lý các ngoại lệ này một cách an toàn trong đường ống xử lý, hãy sử dụng toán tử trung gian **`catch`**.

Ví dụ dưới đây cho thấy cách `LatestNewsViewModel` sử dụng toán tử `catch`:

```kotlin
class LatestNewsViewModel(
    private val newsRepository: NewsRepository
) : ViewModel() {

    init {
        viewModelScope.launch {
            newsRepository.favoriteLatestNews
                // Bắt bất kỳ ngoại lệ nào xảy ra ở các toán tử thượng nguồn
                .catch { exception -> 
                    notifyError(exception) 
                }
                .collect { favoriteNews ->
                    displayNews(favoriteNews)
                }
        }
    }

    private fun notifyError(exception: Throwable) {
        // Hiển thị thông báo lỗi lên màn hình cho người dùng
    }
}
```

### Nguyên Lý Minh Bạch Ngoại Lệ (Exception Transparency Invariant):

> [!IMPORTANT]
> **Quy tắc vàng của Google:**  
> Toán tử `catch` **chỉ bắt các ngoại lệ xảy ra ở phía trên nó (Upstream)** trong chuỗi pipeline.  
> Nó **không bao giờ bắt** các ngoại lệ phát sinh từ bên dưới nó (Downstream), bao gồm cả khối mã bên trong toán tử `collect { ... }`.

Nếu bạn muốn xử lý các ngoại lệ xảy ra bên trong chính khối `collect`, hãy sử dụng khối `try-catch` truyền thống bọc xung quanh lệnh gọi `collect`:

```kotlin
viewModelScope.launch {
    try {
        newsRepository.favoriteLatestNews.collect { favoriteNews ->
            // Nếu hàm displayNews(favoriteNews) ném ra ngoại lệ,
            // khối try-catch này sẽ bắt và xử lý an toàn
            displayNews(favoriteNews)
        }
    } catch (e: Throwable) {
        showErrorMessage(e)
    }
}
```

Ngoài ra, toán tử `catch` cũng có thể phát các giá trị thay thế xuống hạ nguồn thông qua hàm `emit()` (ví dụ: phát danh sách rỗng hoặc dữ liệu dự phòng từ cache).

---

## 7. Thực Thi trong một CoroutineContext Khác (`flowOn`)

### Quy Tắc Bất Biến Bảo Toàn Ngữ Cảnh (Context Preservation)
Theo mặc định, thực thể tạo của hàm tạo `flow` sẽ thực thi trong `CoroutineContext` của chính coroutine đảm nhiệm việc thu thập dữ liệu (`collect`) từ flow đó.

Nếu bên thu thập đang chạy trên Main thread, thì mã bên trong `flow { ... }` cũng sẽ chạy trên Main thread. Nếu thực thể tạo cần thực hiện tác vụ nặng, bạn **không thể dùng `withContext` bên trong khối `flow` để gọi `emit`**:

```kotlin
// ❌ SAI LẦM NGHIÊM TRỌNG: Vi phạm quy tắc bảo toàn ngữ cảnh!
val badFlow = flow {
    withContext(Dispatchers.IO) {
        val data = performHeavyComputation()
        emit(data) // NÉM LỖI RUNTIME: IllegalStateException: Flow invariant is violated!
    }
}
```

---

### Giải Pháp Chuẩn Mực: Toán Tử `flowOn`

Để thay đổi `CoroutineContext` cho việc sản xuất dữ liệu mà không vi phạm quy tắc bảo toàn ngữ cảnh, hãy sử dụng toán tử trung gian **`flowOn`**.

Toán tử `flowOn` thay đổi ngữ cảnh cho **toàn bộ các toán tử nằm phía trước nó (Upstream)**, trong khi phía sau nó (Downstream) vẫn giữ nguyên ngữ cảnh của bên thu thập:

```kotlin
class NewsRepository(
    private val newsRemoteDataSource: NewsRemoteDataSource,
    private val userDataUserDataSource: UserDataUserDataSource,
    private val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default
) {
    val favoriteLatestNews: Flow<List<ArticleHeadline>> =
        newsRemoteDataSource.latestNews
            .map { news -> news.filter { userDataUserDataSource.isFavoriteTopic(it.topic) } }
            .onEach { news -> saveInCache(news) }
            // flowOn ảnh hưởng đến tất cả các toán tử phía trên:
            // newsRemoteDataSource.latestNews, map, và onEach sẽ chạy trên defaultDispatcher
            .flowOn(defaultDispatcher)
            // flowOn KHÔNG ảnh hưởng đến các toán tử phía sau nó:
            // Bên thu thập (collect) trong ViewModel vẫn chạy an toàn trên Dispatchers.Main
}
```

```
Sơ đồ phân chia luồng của flowOn:
[Producer: fetchLatestNews] ────► [map: filter] ────► [onEach: saveCache] ──┐
  (Chạy trên Dispatchers.Default nhờ flowOn)                                │  .flowOn(defaultDispatcher)
────────────────────────────────────────────────────────────────────────────┼────────────────────────────
[Consumer: collect { displayNews() }] ◄─────────────────────────────────────┘
  (Chạy trên Dispatchers.Main trong viewModelScope)
```

---

## 8. Flow trong Thư Viện Jetpack

Flow được tích hợp sâu rộng vào hệ sinh thái các thư viện Android Jetpack:

### 8.1. Jetpack Room Database
Room cung cấp hỗ trợ trực tiếp cho Flow trong các câu lệnh truy vấn dữ liệu quan sát được (Observable Queries):

```kotlin
@Dao
interface ExampleDao {
    // Room tự động thực thi trên background thread và phát ra danh sách mới
    // mỗi khi có bản ghi trong bảng 'Example' được thêm, sửa, hoặc xóa
    @Query("SELECT * FROM Example ORDER BY id DESC")
    fun getExamples(): Flow<List<Example>>
}
```

Khi sử dụng Room với Flow, Room sẽ tự động gửi thông báo cập nhật bất cứ khi nào bảng cơ sở dữ liệu có sự thay đổi mà bạn không cần phải thực hiện truy vấn thủ công lại.

### 8.2. Jetpack DataStore
Thư viện DataStore thay thế cho SharedPreferences sử dụng Flow như một cơ chế cốt lõi để đọc dữ liệu cấu hình bất đồng bộ và an toàn luồng.

---

## 9. Chuyển Đổi API Callback Thành Flow (`callbackFlow`)

> [!IMPORTANT]
> **Kỹ thuật then chốt của Google:**  
> Rất nhiều thư viện Android truyền thống hoặc SDK đám mây (như Firebase Firestore, Android LocationListener, CameraX) sử dụng mô hình **hàm gọi lại (callback listener)** để thông báo dữ liệu mới. Hàm tạo **`callbackFlow`** cho phép bạn bọc các API dựa trên callback này thành một `Flow` hiện đại.

### 9.1. Triển Khai Mẫu Chuẩn với Firebase Firestore

Trong ví dụ chính thức từ Google, chúng ta chuyển đổi `SnapshotListener` của Firebase Firestore thành một `Flow<UserEvents>`:

```kotlin
class FirestoreUserEventsDataSource(
    private val firestore: FirebaseFirestore
) {
    // Chuyển đổi Callback Listener của Firestore thành một Flow bất đồng bộ
    fun getUserEvents(userId: String): Flow<UserEvents> = callbackFlow {
        // 1. Đăng ký listener nhận sự kiện từ Firestore
        val subscription = firestore.collection("users")
            .document(userId)
            .addSnapshotListener { snapshot, _ ->
                if (snapshot == null) { return@addSnapshotListener }
                try {
                    // Phát giá trị mới vào dòng dữ liệu thông qua trySend()
                    trySend(snapshot.toObject(UserEvents::class.java))
                } catch (e: Throwable) {
                    // Xử lý lỗi nếu việc chuyển đổi dữ liệu thất bại
                }
            }

        // 2. BẮT BUỘC: awaitClose giữ coroutine tiếp tục sống và dọn dẹp khi kết thúc
        awaitClose { 
            // Hủy đăng ký listener để chống rò rỉ bộ nhớ khi Flow dừng hoặc bị hủy
            subscription.remove() 
        }
    }
}
```

---

### 9.2. Phân Tích Hai Khái Niệm Sống Còn trong `callbackFlow`:

#### 1. Tại sao dùng `trySend()` thay vì `send()`?
- Khác với hàm tạo `flow { emit(...) }` thông thường, bên trong khối mã của `callbackFlow` bạn sử dụng **`trySend()`** hoặc **`send()`**.
- `send()` là một hàm tạm ngưng (`suspend function`), nó không thể được gọi trực tiếp từ bên trong các callback thông thường nếu callback đó là non-suspending.
- **`trySend()`** là một hàm đồng bộ (non-suspending), nó lập tức đẩy phần tử vào bộ đệm của Channel và trả về một đối tượng `ChannelResult` (thành công hoặc thất bại), rất an toàn khi gọi từ bên trong các listener của hệ thống.

#### 2. Vai trò Bắt Buộc của `awaitClose { ... }`:
> [!CAUTION]
> **Cảnh báo sống còn:**  
> `awaitClose` là thành phần **bắt buộc phải có** ở cuối mỗi hàm `callbackFlow`:  
> 1. **Giữ coroutine sống:** Nếu không có `awaitClose`, coroutine của `callbackFlow` sẽ kết thúc ngay sau khi đăng ký listener, khiến flow lập tức đóng lại và không bao giờ nhận được dữ liệu!  
> 2. **Giải phóng tài nguyên (Clean up):** Khi bên thu thập hạ nguồn dừng lắng nghe hoặc coroutine bị hủy, khối mã bên trong `awaitClose` sẽ tự động được kích hoạt để gỡ bỏ listener (`subscription.remove()`), ngăn chặn hoàn toàn việc rò rỉ bộ nhớ (memory leak).

---

## 10. Các Tài Nguyên Tham Khảo Mở Rộng về Flow

- **StateFlow và SharedFlow:** Hướng dẫn chi tiết cách quản lý trạng thái giao diện UI và phát sự kiện một lần (xem chi tiết tại [Bài 04](04-stateflow-and-sharedflow.md)).
- **Thu thập Flow theo Vòng đời Android:** Kỹ thuật thu thập an toàn với `repeatOnLifecycle` và `collectAsStateWithLifecycle` (xem chi tiết tại [Bài 05](05-lifecycle-aware-flow-collection.md)).
- **Kiểm thử Flow của Kotlin trên Android:** Hướng dẫn viết Unit Test toàn diện cho Flow bằng thư viện `Turbine` (xem chi tiết tại [Bài 06](06-testing-coroutines-and-flow.md)).
