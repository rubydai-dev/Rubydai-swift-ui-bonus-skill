# Hướng dẫn cache cho API backend

Tài liệu dùng chung cho mọi dự án có app gọi API (mobile, web) và backend chạy trên edge (Cloudflare Workers hoặc tương tự). Ví dụ code viết cho Swift và Cloudflare Workers, nhưng nguyên tắc áp dụng được cho nền tảng khác.

Mục tiêu của cache:

- **Nhanh hơn cho user**, nhất là lần đầu và khi mạng yếu.
- **Giảm tải và giảm chi phí** cho phần đắt nhất: database, API bên thứ ba, model AI, dịch vụ TTS…
- **Giảm rủi ro bị chặn** khi phụ thuộc vào dịch vụ bên ngoài có giới hạn tần suất hoặc không chính thức.
- **Dùng được khi offline** (cache phía client).

---

## 1. Các lớp cache, từ gần user đến xa

| Lớp | Ở đâu | Dùng chung giữa các user? | Tồn tại bao lâu | Hợp với |
|---|---|---|---|---|
| RAM của app | Bộ nhớ, theo màn hình hoặc theo phiên | Không | Đến khi đóng màn hoặc app | Dữ liệu vừa dùng, tránh gọi lại khi quay lại màn |
| Đĩa của app | `Caches/` (iOS), IndexedDB (web) | Không | Đến khi bị dọn | Dữ liệu lặp lại nhiều, file media (audio, ảnh, SVG), dùng offline |
| HTTP cache của client | `URLCache`, cache của trình duyệt | Không | Theo `Cache-Control` | Response GET có header cache đúng |
| CDN / cache edge | Data center gần user | Có, **theo từng data center** | Theo TTL, có thể bị xoá sớm | Response GET công khai, hoặc POST được đổi thành key GET (mục 4) |
| Kho bền vững | KV, R2/S3, Redis | Có, toàn cầu hoặc một vùng | Đến khi bạn xoá | Kết quả đắt cần giữ lâu, dùng chung toàn cầu |
| Nguồn gốc | Database, API ngoài | — | — | — |

Quy tắc chọn:

- **Bắt đầu từ client** (RAM + đĩa). Không tốn chi phí server và có hiệu quả ngay cho user hay dùng lại.
- **Thêm cache edge** khi nhiều user hỏi cùng một thứ (nội dung từ điển, bài viết, audio của câu mẫu). Lớp này thường miễn phí.
- **Chỉ thêm kho bền vững** khi đo thấy cache edge trượt nhiều mà nguồn gốc lại đắt hoặc rủi ro. Đây là lớp tốn công vận hành nhất (dung lượng, dọn dẹp, lạm dụng).
- Luôn **đo trước và sau** (mục 8). Đừng thêm lớp cache vì "nghe có vẻ tốt".

---

## 2. Thiết kế cache key

Key sai là nguồn lỗi cache phổ biến nhất: trả nhầm dữ liệu cho request khác, hoặc không bao giờ trúng cache.

- **Đưa vào key mọi tham số làm thay đổi kết quả**: ngôn ngữ, giọng đọc, tốc độ, định dạng output, phiên bản dữ liệu… Thiếu một tham số là trả nhầm.
- **Chuẩn hoá trước khi tạo key**: trim khoảng trắng, gộp khoảng trắng liên tiếp, sắp xếp query param theo tên, chuẩn hoá Unicode nếu cần. Chuẩn hoá *đúng như cách backend xử lý input*, để hai request cho cùng kết quả thì có cùng key.
- **Hash nội dung dài** (SHA-256 rồi lấy hex). Key gọn, không lộ nội dung, không vướng giới hạn độ dài URL.
- **Thêm tiền tố phiên bản** (`v1/…`). Khi đổi định dạng output hoặc sửa lỗi dữ liệu, chỉ cần tăng phiên bản để bỏ toàn bộ cache cũ mà không phải xoá từng mục.
- **Ở client, key theo nội dung chứ không theo màn hình.** Key kiểu `voice|text` cho phép cùng một câu ở hai màn dùng chung một file. Key kiểu `screenId|itemIndex` thì không.

```js
const params = { text: normalize(text), voice, rate, pitch, format };
const hash = await sha256Hex(JSON.stringify(params));   // thứ tự khoá cố định
const key = `v1/${hash}`;
```

---

## 3. Không được cache gì

- **Lỗi**: 4xx, 5xx, timeout, response rỗng hoặc không hợp lệ. Cache lỗi nghĩa là user bị lỗi suốt TTL. Chỉ ghi cache sau khi đã kiểm tra kết quả hợp lệ.
- **Dữ liệu riêng của user** (cần đăng nhập, có `Authorization` hoặc cookie) ở cache dùng chung. Nếu bắt buộc phải cache thì đưa user id vào key, và chỉ dùng cache riêng.
- **Nội dung nhạy cảm** do user tự nhập (tài liệu cá nhân, ảnh OCR…) trong kho bền vững dùng chung. Cache trên máy của chính user thì được. Nếu cần, cho client gửi kèm cờ kiểu `cache: false`.
- **Response có `Set-Cookie`.** Phần lớn CDN sẽ không cache, hoặc tệ hơn là cache cookie của người này rồi trả cho người khác.

---

## 4. Cache edge trên Cloudflare Workers (Cache API)

`caches.default` miễn phí, không cần binding, phản hồi rất nhanh (chỉ vài ms khi trúng). Những điều cần biết:

- **Theo từng data center.** User ở hai nước khác nhau mỗi bên phải tạo lần đầu riêng. Dữ liệu có thể bị xoá trước TTL nếu ít dùng.
- **Chỉ dùng key là request GET.** Với API dạng POST, hãy tạo một URL giả từ hash của body làm key.
- **Không chạy trên domain `*.workers.dev`.** Phải gắn custom domain thì `cache.put` mới có tác dụng.
- Response đưa vào cache cần có `Cache-Control: public, max-age=…`.
- **Ghi cache bằng `ctx.waitUntil`**, để user không phải chờ bước ghi. Nhớ truyền `ctx` vào handler.
- Thêm header **`X-Cache: HIT/MISS`** vào response để kiểm tra và đo tỉ lệ trúng.

```js
const CACHE_VERSION = "v1";
const CACHE_TTL = 30 * 24 * 60 * 60;

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    const body = await request.json();
    const params = normalizeParams(body);

    const cache = caches.default;
    const key = new Request(`${url.origin}/__cache/${CACHE_VERSION}/${await sha256Hex(JSON.stringify(params))}`);

    const hit = await cache.match(key);
    if (hit) {
      return new Response(hit.body, { headers: { "Content-Type": "application/json", "X-Cache": "HIT" } });
    }

    const json = JSON.stringify(await expensiveWork(params));   // throw on failure → never cached

    ctx.waitUntil(cache.put(key, new Response(json, {
      headers: { "Content-Type": "application/json", "Cache-Control": `public, max-age=${CACHE_TTL}` },
    })));
    return new Response(json, { headers: { "Content-Type": "application/json", "X-Cache": "MISS" } });
  },
};

async function sha256Hex(str) {
  const digest = await crypto.subtle.digest("SHA-256", new TextEncoder().encode(str));
  return [...new Uint8Array(digest)].map((b) => b.toString(16).padStart(2, "0")).join("");
}
```

**Giữ nguyên định dạng response** khi thêm cache. Nhờ vậy các bản app đã phát hành hưởng lợi ngay mà không cần cập nhật.

Với API dạng **GET công khai**, có thể để CDN tự cache bằng header `Cache-Control` (cộng với Cache Rules trên dashboard nếu cần), và **sắp xếp query param cố định** ở client để các request giống nhau có cùng URL.

---

## 5. Kho bền vững: khi nào cần, chọn gì

Chỉ cần khi: kết quả **đắt** (gọi AI, TTS, API trả phí), được **nhiều user dùng lại**, và cache edge **trượt nhiều** vì user ở nhiều vùng hoặc dữ liệu ít người hỏi.

| Lựa chọn | Hợp với | Lưu ý |
|---|---|---|
| Object storage (R2, S3) | File nhị phân (audio, ảnh), số lượng lớn | Rẻ; R2 không tính phí egress. Lifecycle rule xoá theo *tuổi file*, không theo lần truy cập cuối |
| KV (Workers KV) | Đọc rất nhiều, ghi rất ít | Gói miễn phí giới hạn số lần ghi mỗi ngày rất thấp, **không hợp** với cache ghi liên tục. Ghi xong mất vài chục giây mới thấy ở mọi nơi |
| Redis có REST API (ví dụ Upstash) | Key–value nhỏ, cần TTL theo từng key, đếm, rate limit | Gọi được từ Worker qua HTTP. Gói miễn phí giới hạn theo dung lượng và số lệnh |
| Redis chỉ có TCP (ví dụ Redis Cloud gói miễn phí) | Backend chạy server truyền thống | Gói miễn phí rất nhỏ (cỡ vài chục MB). Từ edge function thì khó kết nối và chậm, vì database nằm ở một vùng |

Giới hạn và giá của các gói miễn phí thay đổi thường xuyên. **Kiểm tra lại trên trang giá** trước khi quyết định.

Khi dùng kho bền vững:

- Đặt **Cache API trước** kho bền vững: Cache API → kho → nguồn gốc.
- Ước lượng dung lượng: kích thước trung bình một mục × số mục dự kiến. Ví dụ, audio MP3 48 kbps tốn khoảng 6 KB mỗi giây.
- Có kế hoạch **dọn dẹp** (TTL, lifecycle) và **chặn lạm dụng** (mục 7).

---

## 6. Cache trên client

### RAM

- Dictionary theo key, sống theo màn hoặc theo phiên. Rất nhanh nhưng mất khi đóng màn.
- **Chống gọi trùng (in-flight dedupe):** lưu `Task` đang chạy theo key. Request thứ hai cho cùng key thì `await` chính `Task` đó.

```swift
private var inFlight: [String: Task<Item, Error>] = [:]

func item(for key: String) async throws -> Item {
    if let cached = cache[key] { return cached }
    if let running = inFlight[key] { return try await running.value }
    // Unstructured: if the caller is cancelled (e.g. a debounced prefetch), the result still lands in the cache.
    let task = Task { try await fetch(key) }
    inFlight[key] = task
    defer { inFlight[key] = nil }
    let value = try await task.value
    cache[key] = value
    return value
}
```

### Đĩa

- iOS: dùng thư mục `Caches/`. Hệ thống có thể tự dọn khi máy thiếu dung lượng, nên **luôn coi dữ liệu ở đây là có thể mất**.
- Tên file là hash của key (kèm tiền tố phiên bản), ghi với `.atomic`.
- **Giới hạn dung lượng** và xoá file ít dùng nhất (LRU): cập nhật ngày sửa file mỗi lần đọc; khi vượt giới hạn thì xoá theo ngày cũ nhất, xuống dưới khoảng 75% giới hạn để không phải dọn liên tục. Chạy việc dọn tối đa một lần mỗi phiên.
- **Đọc và ghi file ngoài main thread.** Với dự án đặt isolation mặc định là `MainActor`, đánh dấu type là `nonisolated` và hàm là `@concurrent`.
- Ghi cache không được làm chậm việc hiển thị: trả dữ liệu cho UI trước, ghi file sau (`Task { await write(...) }`).

### HTTP cache (`URLCache`)

- Tôn trọng `Cache-Control` từ server. Đây là cách rẻ nhất nếu backend đặt header đúng.
- Mặc định dung lượng khá nhỏ. Nếu dựa vào nó thì cấu hình `URLCache` lớn hơn cho session đó.
- Response lớn so với dung lượng cache có thể không được lưu. Với file lớn, hãy tự cache trên đĩa.
- Khi đo hiệu năng, nhớ rằng `URLCache` của lần chạy trước làm kết quả đẹp giả. Gỡ app trước khi đo.

### Tải trước (prefetch)

- Tải trước những gì user nhiều khả năng sẽ bấm (ví dụ 3 kết quả tìm kiếm đầu), **có debounce** (300–500ms) vì danh sách đổi theo từng phím gõ.
- Dùng chung cơ chế chống gọi trùng ở trên. Nếu không, màn chi tiết mở ra lúc request tải trước chưa xong sẽ gửi request thứ hai.
- Bỏ qua khi dữ liệu đã có sẵn trên máy (gói offline, cache).

---

## 7. An toàn và chống lạm dụng

Cache dùng chung biến endpoint công khai thành một chỗ có thể bị làm đầy hoặc bị dùng ké.

- **Giới hạn kích thước input ngay ở backend.** Không tin vào việc client đã tự cắt, vì ai cũng gọi thẳng endpoint được.
- **Rate limit** theo IP hoặc theo user (rule trên CDN hoặc trong code).
- Với kho bền vững: giới hạn dung lượng, có lifecycle, theo dõi tốc độ tăng.
- Validate tham số theo danh sách cho phép (ngôn ngữ, giọng, định dạng) để kẻ xấu không tạo ra vô số key khác nhau.

---

## 8. Đo lường

Đo **trước khi** thêm cache và **sau khi** thêm, cùng một kịch bản.

```sh
# Tách chi phí kết nối, xử lý và cache
curl -s -o /dev/null -D - \
  -w "dns=%{time_namelookup} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n" \
  "https://api.example.com/lookup?term=..." | grep -iE "x-cache|cf-cache-status|age|dns="
```

- Gọi 2 lần liên tiếp cùng một input: lần đầu phải là `MISS`, lần sau là `HIT`, và **nội dung phải giống hệt** (so bằng `cmp`).
- Chạy thử local (`wrangler dev`) trước khi deploy. Nếu commit là tự deploy thì đây là bước kiểm tra cuối cùng.
- Theo dõi tỉ lệ trúng cache qua log hoặc header. Tỉ lệ trúng thấp thường do key chưa chuẩn hoá, hoặc do dữ liệu vốn ít lặp lại (trường hợp này thì cache không đáng làm).
- Phân biệt các chi phí: **kết nối** (DNS + TLS, chữa bằng làm nóng kết nối hoặc giữ kết nối), **truy vấn** (chữa bằng cache) và **hiển thị** (chữa phía client, xem mục "Lần đầu dùng bị chậm" trong `silent-bugs-checklist.md`).

---

## 9. Checklist khi thêm cache

- [ ] Key có đủ mọi tham số ảnh hưởng kết quả, đã chuẩn hoá, có tiền tố phiên bản.
- [ ] Chỉ cache kết quả thành công và hợp lệ.
- [ ] Không cache dữ liệu riêng của user ở lớp dùng chung.
- [ ] Ghi cache không chặn response (`waitUntil` ở server, ghi nền ở client).
- [ ] Có giới hạn dung lượng và cách dọn dẹp.
- [ ] Backend giới hạn kích thước input và có rate limit.
- [ ] Định dạng response không đổi (client cũ vẫn chạy).
- [ ] Có header HIT/MISS hoặc log để đo.
- [ ] Đã thử MISS → HIT với nội dung giống hệt trước khi deploy.
- [ ] Đã ghi vào README: TTL, cách xoá cache (tăng phiên bản), những gì cố ý không cache.
