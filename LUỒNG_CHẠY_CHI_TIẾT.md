# LUỒNG CHẠY CHI TIẾT CỦA DỰ ÁN UNDANGAN

## TỔNG QUAN

Dự án là một website mời đám cưới (Wedding Invitation) được xây dựng bằng vanilla JavaScript ES6 modules, HTML5, và CSS3. Ứng dụng hỗ trợ PWA với offline capability, caching, và nhiều tính năng tương tác.

---

## BƯỚC 1: TRÌNH DUYỆT TẢI TRANG (Page Load)

### 1.1. HTML Parsing

- Trình duyệt bắt đầu parse file `index.html`
- Đọc các thẻ meta (SEO, PWA, theme-color)
- Tải các CSS từ CDN:
  - Bootstrap 5.3.8
  - Font Awesome 7.1.0
  - Google Fonts (Josefin Sans)
  - `./css/guest.css` (local)

### 1.2. DOM Structure

- Tạo cấu trúc DOM với:
  - `#loading` - Màn hình loading ban đầu (opacity: 1)
  - `#welcome` - Màn hình chào mừng (opacity: 0)
  - `#root` - Nội dung chính (opacity: 0)
  - Các section: home, bride, wedding-date, gallery, comment
  - Navbar bottom navigation
  - Modal cho hình ảnh

### 1.3. JavaScript Loading

- Script `./dist/guest.js` được tải (defer)
- Bootstrap JS được tải từ CDN

---

## BƯỚC 2: KHỞI TẠO JAVASCRIPT (Module Initialization)

### 2.1. Entry Point: `js/guest.js`

```
window.undangan = guest.init()
```

### 2.2. `guest.init()` trong `js/app/guest/guest.js`

Thực hiện các bước khởi tạo:

#### 2.2.1. Theme Initialization (`theme.init()`)

- Kiểm tra `data-bs-theme` trên `<html>`:
  - `"auto"` → Chế độ tự động theo system preference
  - `"dark"` → Chế độ tối
  - `"light"` → Chế độ sáng
- Đọc localStorage từ `storage('theme')`
- Nếu chưa có, kiểm tra system preference:
  ```javascript
  window.matchMedia("(prefers-color-scheme: dark)").matches;
  ```
- Áp dụng theme (dark/light) và cập nhật meta theme-color

#### 2.2.2. Session Initialization (`session.init()`)

- Khởi tạo `storage('session')` trong localStorage
- Kiểm tra nếu là admin (JWT token có 3 phần):
  ```javascript
  String(getToken() ?? ".").split(".").length === 3;
  ```
- Nếu là admin, xóa các storage không cần thiết:
  - `storage('user').clear()`
  - `storage('owns').clear()`
  - `storage('likes').clear()`
  - `storage('session').clear()`
  - `storage('comment').clear()`

#### 2.2.3. Đăng ký Event Listener cho Window Load

```javascript
window.addEventListener("load", () => {
  pool.init(pageLoaded, ["image", "video", "audio", "libs", "gif"]);
});
```

---

## BƯỚC 3: KHỞI TẠO CACHE POOL (Cache Initialization)

### 3.1. `pool.init()` trong `js/connection/request.js`

- Kiểm tra secure context (HTTPS hoặc localhost)
- Mở các Cache API:
  - `"request"` - Cache cho HTTP requests
  - `"image"` - Cache cho hình ảnh
  - `"video"` - Cache cho video
  - `"audio"` - Cache cho audio
  - `"libs"` - Cache cho thư viện bên ngoài
  - `"gif"` - Cache cho GIF (từ comment)
- Sau khi tất cả cache được mở, gọi callback `pageLoaded()`

---

## BƯỚC 4: KHỞI TẠO CÁC MODULE CHÍNH (`pageLoaded()`)

### 4.1. Language Module (`lang.init()`)

- Khởi tạo hệ thống đa ngôn ngữ

### 4.2. Offline Module (`offline.init()`)

- Đăng ký event listeners:
  - `window.addEventListener('online', onOnline)`
  - `window.addEventListener('offline', onOffline)`
- Tạo alert element hiển thị trạng thái kết nối
- Disable các input/button có `data-offline-disabled="false"` khi offline

### 4.3. Comment Module (`comment.init()`)

- Khởi tạo các sub-modules:
  - `gif.init()` - Xử lý GIF trong comment
  - `like.init()` - Xử lý like button
  - `card.init()` - Render comment cards
  - `pagination.init()` - Phân trang comment
- Lấy DOM element `#comments`
- Đăng ký event: `comments.addEventListener('undangan.comment.show', show)`
- Khởi tạo storage:
  - `owns = storage('owns')` - Lưu mapping UUID → own ID
  - `showHide = storage('comment')` - Lưu trạng thái show/hide replies

### 4.4. Progress Module (`progress.init()`)

- Lấy DOM elements:
  - `#progress-info` - Hiển thị thông tin loading
  - `#progress-bar` - Progress bar
- Hiển thị progress info (remove `d-none`)
- Tạo promise `cancelProgress` để có thể cancel khi có lỗi

### 4.5. Khởi tạo các Module Loader

#### 4.5.1. Video Module (`video.init()`)

- Thêm 1 vào progress counter: `progress.add()`
- Khởi tạo cache: `cache('video').withForceCache()`
- Trả về object: `{ load }`

#### 4.5.2. Image Module (`image.init()`)

- Lấy tất cả `<img>` elements
- Đếm số lượng images: `images.forEach(progress.add)`
- Khởi tạo cache: `cache('image').withForceCache()`
- Trả về object: `{ load, download, hasDataSrc }`

#### 4.5.3. Audio Module (`audio.init()`)

- Thêm 1 vào progress counter: `progress.add()`
- Trả về object: `{ load }`

#### 4.5.4. Loader Libs Module (`loaderLibs()`)

- Thêm 1 vào progress counter: `progress.add()`
- Trả về object: `{ load }`

### 4.6. Lấy Token và Parameters

```javascript
const token = document.body.getAttribute("data-key");
const params = new URLSearchParams(window.location.search);
```

### 4.7. Đăng ký Event Listeners

- `window.addEventListener("resize", util.debounce(slide))` - Slide desktop images
- `document.addEventListener("undangan.progress.done", () => booting())` - Khi loading xong
- `document.addEventListener("hide.bs.modal", () => document.activeElement?.blur())` - Xử lý modal
- Download button handler cho modal image

### 4.8. Bắt đầu Load Resources

#### 4.8.1. Kiểm tra Images có data-src không

```javascript
if (!img.hasDataSrc()) {
  img.load(); // Load images ngay lập tức
}
```

#### 4.8.2. Thêm Progress Counter

- Thêm 2 progress cho config và comment: `progress.add()` x2

#### 4.8.3. Fetch Config từ API

```javascript
session.guest(params.get("k") ?? token);
```

**Chi tiết `session.guest()`:**

1. Gọi API: `GET /api/v2/config`
   - Headers:
     - `x-access-key: {token}` (nếu token không phải JWT)
     - `Authorization: Bearer {token}` (nếu token là JWT)
   - Cache: 30 phút với force cache
2. Lưu config vào localStorage:
   ```javascript
   const config = storage("config");
   for (const [k, v] of Object.entries(res.data)) {
     config.set(k, v);
   }
   ```
3. Lưu token vào session storage
4. Dispatch event: `document.dispatchEvent(new Event("undangan.session"))`
5. Complete progress: `progress.complete("config")`

#### 4.8.4. Sau khi có Config, Load các Resources

**a) Load Images (nếu có data-src):**

```javascript
if (img.hasDataSrc()) {
  img.load();
}
```

**Chi tiết `image.load()`:**

1. Chia images thành 2 nhóm:
   - Có `fetchpriority` (ưu tiên cao)
   - Không có `fetchpriority`
2. Với mỗi image:
   - Nếu có `data-src`: Thêm vào `urlCache` để fetch từ cache
   - Nếu không: Dùng `onload`/`onerror` event
3. Gọi `cache.run(urlCache, progress.getAbort())`:
   - Kiểm tra cache trước
   - Nếu không có, fetch và cache
   - Tạo object URL từ blob
   - Gọi callback `appendImage()` cho mỗi image
4. Update progress: `progress.complete('image')` cho mỗi image

**b) Load Video:**

```javascript
vid.load();
```

**Chi tiết `video.load()`:**

1. Kiểm tra element `#video-love-stroy` có `data-src` không
2. Nếu không có, remove element và complete progress
3. Nếu có:
   - Tạo `<video>` element với attributes
   - Kiểm tra cache video trước
   - Nếu cache hit:
     - Load từ blob
     - Append vào DOM
     - Complete progress
   - Nếu cache miss:
     - Fetch với Range header để kiểm tra hỗ trợ partial content
     - Nếu hỗ trợ: Load video với progress tracking
     - Fetch full video với retry và progress function
     - Cache video sau khi load xong
   - Tạo IntersectionObserver để auto-play khi vào viewport

**c) Load Audio:**

```javascript
aud.load();
```

**Chi tiết `audio.load()`:**

1. Lấy URL từ `document.body.getAttribute('data-audio')`
2. Nếu không có URL, complete progress và return
3. Nếu có:
   - Fetch audio từ cache (force cache)
   - Tạo `Audio` object với:
     - `loop = true`
     - `muted = false`
     - `autoplay = false`
   - Complete progress
   - Đăng ký event listeners:
     - `undangan.open` → Hiển thị button và play audio
     - `offline` → Pause audio
     - `click` trên button → Toggle play/pause

**d) Load Libraries:**

```javascript
lib.load({ confetti: data.is_confetti_animation });
```

**Chi tiết `loader()`:**

1. Khởi tạo cache: `cache('libs').withForceCache()`
2. Load các thư viện:
   - **AOS (Animate On Scroll):**
     - Load CSS từ CDN
     - Load JS từ CDN
     - Khởi tạo: `window.AOS.init()`
   - **Confetti (nếu enabled):**
     - Load JS từ CDN
     - Kiểm tra `window.confetti` có tồn tại
   - **Additional Fonts:**
     - Sacramento font
     - Noto Naskh Arabic font
     - Load và cache fonts
3. Complete progress: `progress.complete("libs")`

**e) Load Comments:**

```javascript
comment
  .show()
  .then(() => progress.complete("comment"))
  .catch(() => progress.invalid("comment"));
```

**Chi tiết `comment.show()`:**

1. Remove các event listeners cũ từ `lastRender`
2. Set `data-loading="true"` và hiển thị loading cards
3. Gọi API: `GET /api/v2/comment?per={per}&next={next}&lang={lang}`
   - Token từ session
   - Cache 30 giây với force cache
4. Parse response với DTO: `dto.getCommentsResponseV2`
5. Render comments:
   - Xóa GIF cũ từ `lastRender`
   - Nếu không có comments, hiển thị message "share invitation"
   - Flatten UUIDs từ nested comments
   - Xử lý show/hide replies dựa trên storage
   - Render HTML với `card.renderContentMany()`
   - Thêm event listeners cho like buttons
6. Nếu là admin, fetch location info cho mỗi comment (từ IP)
7. Update pagination total
8. Dispatch events:
   - `undangan.comment.result`
   - `undangan.comment.done`

---

## BƯỚC 5: KHI TẤT CẢ RESOURCES LOAD XONG (`booting()`)

### 5.1. Event Trigger

Khi tất cả progress complete, event `undangan.progress.done` được dispatch:

```javascript
document.dispatchEvent(new Event("undangan.progress.done"));
```

### 5.2. Hàm `booting()` thực hiện:

#### 5.2.1. Khởi tạo Animations

- `animateSvg()` - Thêm animation class cho SVG elements theo `data-time`
- `normalizeArabicFont()` - Normalize Arabic text với `String.normalize("NFC")`

#### 5.2.2. Countdown Timer

- `countDownDate()` - Bắt đầu countdown timer:
  1. Lấy thời gian từ `data-time` attribute
  2. Tính khoảng cách tuyệt đối đến thời gian đó
  3. Update DOM mỗi giây: `#day`, `#hour`, `#minute`, `#second`
  4. Sử dụng `util.timeOut()` để sync với giây thực

#### 5.2.3. Show Guest Name

- `showGuestName()`:
  1. Parse query string `?to=name`
  2. Hiển thị tên guest trên welcome screen
  3. Pre-fill form name nếu có

#### 5.2.4. Modal Image Click Handler

- `modalImageClick()` - Toggle visibility của button controls khi click image

#### 5.2.5. Build Google Calendar Link

- `buildGoogleCalendar()` - Tạo URL Google Calendar với event details

#### 5.2.6. Restore Form State

- Kiểm tra localStorage `information`:
  - Nếu có `presence`, set giá trị cho `#form-presence`
  - Nếu có `info = true`, remove information alert

#### 5.2.7. Show Welcome Screen

```javascript
await util.changeOpacity(document.getElementById("welcome"), true);
```

#### 5.2.8. Hide Loading Screen

```javascript
await util
  .changeOpacity(document.getElementById("loading"), false)
  .then((el) => el.remove());
```

---

## BƯỚC 6: NGƯỜI DÙNG TƯƠNG TÁC

### 6.1. Click "Open Invitation" Button

- Gọi `undangan.guest.open(button)`

**Chi tiết `open()`:**

1. Disable button
2. Scroll to top: `document.body.scrollIntoView({ behavior: "instant" })`
3. Hiển thị root: `#root.classList.remove("opacity-0")`
4. Nếu theme auto mode, hiển thị theme button
5. Bắt đầu slide desktop images: `slide()`
6. Bắt đầu theme spy: `theme.spyTop()` (đổi theme-color theo section)
7. Chạy confetti animations:
   - `confetti.basicAnimation()` - Ngay lập tức
   - `confetti.openAnimation()` - Sau 1.5 giây
8. Dispatch event: `undangan.open`
9. Ẩn welcome screen: `changeOpacity(#welcome, false).then(remove)`

### 6.2. Audio Auto-Play

- Khi event `undangan.open` được dispatch:
  - Audio button hiển thị: `#button-music.classList.remove("d-none")`
  - Nếu `playOnOpen = true`, gọi `audio.play()`

### 6.3. Scroll Interactions

#### 6.3.1. Bootstrap Scroll Spy

- Theo dõi scroll position
- Highlight nav item tương ứng với section hiện tại

#### 6.3.2. AOS Animations

- Khi section vào viewport, trigger animation từ `data-aos` attributes

#### 6.3.3. Video Auto-Play

- IntersectionObserver tự động play/pause video khi vào/ra viewport

### 6.4. Image Interactions

#### 6.4.1. Click Image để xem Modal

- Gọi `undangan.guest.modal(img)`
- Set image source, width, height
- Hiển thị Bootstrap modal: `bs.modal("modal-image").show()`

#### 6.4.2. Download Image

- Click download button trong modal
- Gọi `image.download(blobUrl)`
- Sử dụng `cache.download()` với `request().withDownload()`

### 6.5. Comment Interactions

#### 6.5.1. Send Comment

- Gọi `undangan.comment.send(button)`

**Chi tiết:**

1. Validate:
   - Name không rỗng
   - Presence được chọn (nếu là comment chính)
   - Comment hoặc GIF không rỗng
2. Disable form và buttons
3. Gọi API: `POST /api/comment?lang={lang}`
   - Body: `{ name, presence, comment, gif_id, parent_uuid }`
4. Nếu thành công:
   - Lưu own ID vào storage
   - Clear form
   - Insert comment vào DOM
   - Nếu là parent comment, reload list
   - Nếu là reply, insert vào reply section
5. Restore form và buttons

#### 6.5.2. Like Comment

- Click like button
- Gọi `like.toggle(uuid)`
- API: `POST /api/comment/{uuid}/like`
- Update UI: Icon và count

#### 6.5.3. Reply Comment

- Click reply button
- Gọi `undangan.comment.reply(uuid)`
- Insert reply form vào DOM
- Khi submit, gọi `send()` với `parent_uuid`

#### 6.5.4. Edit Comment

- Click edit button
- Gọi `undangan.comment.edit(button, is_parent)`
- Insert edit form với content hiện tại
- Khi submit, gọi `update()`

#### 6.5.5. Delete Comment

- Click delete button
- Gọi `undangan.comment.remove(button)`
- Confirm với user
- API: `DELETE /api/comment/{own_id}`
- Remove element từ DOM

#### 6.5.6. Pagination

- Click pagination button
- Gọi `comment.show()` với `next` parameter mới
- Reload comments list

### 6.6. Theme Toggle

- Click theme button
- Gọi `undangan.theme.change()`
- Toggle giữa dark/light mode
- Update localStorage và meta theme-color

### 6.7. Audio Toggle

- Click audio button
- Toggle play/pause
- Update icon (pause/play)

### 6.8. Copy to Clipboard

- Click copy button (ví dụ: số tài khoản)
- Gọi `undangan.util.copy(button)`
- Copy `data-copy` attribute
- Show success feedback

### 6.9. Show/Hide Story

- Click "Lihat Story" button
- Gọi `undangan.guest.showStory(div)`
- Vibrate device (nếu hỗ trợ)
- Chạy confetti animation
- Ẩn overlay button

---

## BƯỚC 7: OFFLINE HANDLING

### 7.1. Khi Mất Kết Nối

- Event `offline` được trigger
- `offline.onOffline()`:
  1. Set `online = false`
  2. Hiển thị alert: "Koneksi tidak tersedia" (màu đỏ)
  3. Disable tất cả elements có `data-offline-disabled="false"`
  4. Pause audio nếu đang play

### 7.2. Khi Có Lại Kết Nối

- Event `online` được trigger
- `offline.onOnline()`:
  1. Set `online = true`
  2. Hiển thị alert: "Koneksi tersedia kembali" (màu xanh)
  3. Sau 3 giây, ẩn alert
  4. Enable lại các elements

---

## BƯỚC 8: CACHE MANAGEMENT

### 8.1. Request Cache

- Tất cả GET requests được cache trong `cacheRequest`
- TTL mặc định: 6 giờ (có thể override)
- Kiểm tra cache trước khi fetch
- Update cache sau khi fetch thành công

### 8.2. Resource Cache

- **Images:** Cache trong `image` cache với force cache
- **Videos:** Cache trong `video` cache với force cache
- **Audio:** Cache trong `audio` cache với force cache
- **Libraries:** Cache trong `libs` cache với force cache
- **GIFs:** Cache trong `gif` cache

### 8.3. Cache Strategy

- **Cache First:** Kiểm tra cache trước, nếu không có mới fetch
- **Force Cache:** Luôn cache với TTL được set
- **Cache Invalidation:** Xóa cache cũ khi có version mới

---

## BƯỚC 9: ERROR HANDLING

### 9.1. Progress Errors

- Nếu có lỗi khi load resource:
  - `progress.invalid(type)` được gọi
  - Progress bar chuyển sang màu đỏ
  - Dispatch event: `undangan.progress.invalid`
  - Cancel các request đang pending

### 9.2. API Errors

- Nếu API trả về lỗi:
  - Hiển thị alert với message từ server
  - Nếu là server error (500+), hiển thị error ID
  - Nếu là client error, hiển thị warning message

### 9.3. Network Errors

- Nếu mất kết nối:
  - Offline module xử lý
  - Retry với exponential backoff (nếu có `withRetry()`)

---

## BƯỚC 10: PERFORMANCE OPTIMIZATIONS

### 10.1. Lazy Loading

- Images với `data-src` được load lazy
- Images có `fetchpriority` được load trước
- Video chỉ load khi vào viewport

### 10.2. Debouncing

- Window resize handler được debounce
- Scroll handlers được optimize

### 10.3. Request Pooling

- Sử dụng `inFlightRequests` Map để tránh duplicate requests
- Cache object URLs để tái sử dụng

### 10.4. Progressive Loading

- Loading screen hiển thị progress bar
- Welcome screen hiển thị sau khi load xong
- Root content hiển thị sau khi user click "Open Invitation"

---

## TÓM TẮT LUỒNG CHẠY

```
1. Browser Load HTML
   ↓
2. Parse DOM Structure
   ↓
3. Load JavaScript (guest.js)
   ↓
4. guest.init() → theme.init(), session.init()
   ↓
5. window.addEventListener('load')
   ↓
6. pool.init() → Mở các Cache API
   ↓
7. pageLoaded() → Khởi tạo modules
   ↓
8. Load Resources:
   - Images (lazy load)
   - Video (cache first)
   - Audio (cache first)
   - Libraries (AOS, Confetti, Fonts)
   - Comments (API call)
   ↓
9. booting() → Setup UI, animations, countdown
   ↓
10. Show Welcome Screen
    ↓
11. User Click "Open Invitation"
    ↓
12. Show Root Content + Confetti + Audio
    ↓
13. User Interactions (scroll, click, comment, etc.)
```

---

## CÁC THÀNH PHẦN CHÍNH

### Modules

- **guest.js** - Main entry point và orchestration
- **theme.js** - Dark/light theme management
- **session.js** - Authentication và token management
- **storage.js** - localStorage wrapper
- **request.js** - HTTP client với caching và retry
- **cache.js** - Cache API wrapper
- **progress.js** - Loading progress tracking
- **comment.js** - Comment system với like, reply, edit, delete
- **image.js** - Image loading và caching
- **video.js** - Video loading với progress
- **audio.js** - Background music
- **offline.js** - Offline detection và handling
- **util.js** - Utility functions

### Components

- **card.js** - Comment card rendering
- **like.js** - Like button functionality
- **gif.js** - GIF picker và integration
- **pagination.js** - Comment pagination

### Libraries

- **bootstrap.js** - Bootstrap utilities
- **confetti.js** - Confetti animations
- **loader.js** - External library loader

---

## LƯU Ý QUAN TRỌNG

1. **Secure Context Required:** Ứng dụng yêu cầu HTTPS hoặc localhost để sử dụng Cache API
2. **Token Management:** Token được lưu trong localStorage với key `session`
3. **Progress Tracking:** Mỗi resource được đếm trong progress counter
4. **Event-Driven:** Nhiều tương tác dựa trên custom events
5. **Offline Support:** Ứng dụng có thể hoạt động offline với cached resources
6. **PWA Ready:** Có thể được cài đặt như PWA với manifest.json và service worker (nếu có)
