# Checklist lỗi ngầm (silent bugs) cho dự án SwiftUI

Tài liệu này dùng được cho mọi dự án SwiftUI. Có thể copy sang dự án khác, rồi chỉ sửa phần ví dụ.

"Lỗi ngầm" là lỗi **build vẫn qua, nhưng app crash hoặc chạy sai lúc runtime**. Có hai loại:

- **Crash:** app bị kill ngay khi người dùng đi vào đúng luồng đó.
- **Sai lặng lẽ:** app không crash, nhưng nút bấm không có tác dụng, lỗi bị nuốt, dữ liệu gửi lên server sai, hoặc âm thanh bị tắt.

Compiler không bắt được hai loại này. Vì vậy, bước kiểm tra "build OK" sau mỗi thay đổi là **chưa đủ**.

---

## Khi nào chạy checklist

- Sau khi port hoặc copy view từ dự án khác. Đây là lúc dễ gặp lỗi nhất, vì view mang theo "hợp đồng ngầm" với view cha cũ.
- Khi dùng lại một component ở chỗ mới, ví dụ một renderer vốn chỉ nằm trong sheet A, nay được đặt vào màn B.
- Trước khi phát hành.

Thứ tự nên làm:

1. Lấy danh sách warning (mục 0).
2. Quét theo từng nhóm từ mục 1 đến mục 14.
3. Thử tay các kịch bản ở mục 15.

---

## 0. Đọc warning của một lần build sạch

Rất nhiều lỗi runtime vẫn có dấu hiệu dưới dạng **warning**, đặc biệt khi dự án ở Swift 5 mode nhưng bật `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`. Khi đó, vi phạm isolation chỉ là warning.

```sh
# touch để buộc compile lại toàn bộ, nếu không Xcode chỉ in warning của file vừa đổi
find "<App>" -name '*.swift' -exec touch {} +
xcodebuild -workspace "<App>.xcworkspace" -scheme "<App>" \
  -destination 'generic/platform=iOS Simulator' build 2>&1 \
  | grep -E "\.swift:[0-9]+:[0-9]+: warning" | grep -v "/Pods/" | sort -u
```

Cần đọc kỹ các warning sau:

| Warning | Ý nghĩa |
|---|---|
| `main actor-isolated ... cannot be called from outside of the actor` | Code `@MainActor` đang chạy trên thread nền (thường trong `Task.detached`). |
| `expression is 'async' but is not marked with 'await'` | Compiler tự chèn bước chuyển về main actor, nên `Task.detached` không còn tác dụng và main thread vẫn bị chặn. |
| `capture of ... with non-Sendable type` | Có khả năng data race. |
| `result of 'try?' is unused` | Lỗi bị nuốt. Cần xác định có cố ý hay không. |

---

## 1. Dependency lấy từ environment

**Crash:** `No ObservableObject of type X found` hoặc `No Observable object of type X found`.

```sh
grep -rn "@EnvironmentObject" --include='*.swift' .
grep -rhoE "@Environment\([A-Z][A-Za-z]+\.self\)" --include='*.swift' . | sort | uniq -c
grep -rnE "\.environmentObject\(|\.environment\([a-zA-Z]" --include='*.swift' .
```

- Với mỗi type, xác định nơi inject. Nếu inject ở root `App`, mọi view kể cả sheet và fullScreenCover đều nhận được. Nếu inject ở một view con, mọi nơi dùng lại component đó đều phải inject theo.
- **Environment không truyền qua** `UIHostingController` do mình tự tạo, `ImageRenderer` và `UIViewControllerRepresentable` host SwiftUI. Nếu view được render ở những chỗ này, hãy truyền dữ liệu qua tham số.
- **Cách sửa gốc:** với dependency không bắt buộc, không dùng `@EnvironmentObject`. Dùng environment value kiểu optional để khi quên inject thì chỉ mất tính năng, app không crash:

  ```swift
  extension EnvironmentValues {
      @Entry var audioPlayer: AudioPlayerController?
  }
  // view con
  @Environment(\.audioPlayer) private var controller
  var body: some View { if let controller { SpeakerButton(controller: controller) } }
  // view cha
  .environment(\.audioPlayer, player)
  ```

## 2. Khoá quyền trong Info.plist

**Crash:** iOS kill app ngay khi gọi API cần quyền mà thiếu khoá `NS...UsageDescription`. Chỉ bị ở đúng lần đầu gọi API, nên rất dễ lọt.

| API / hành động | Khoá cần có |
|---|---|
| `UIImagePickerController` camera, `AVCaptureDevice` | `NSCameraUsageDescription` |
| `AVAudioEngine`, `AVAudioRecorder`, `requestRecordPermission` | `NSMicrophoneUsageDescription` |
| `SFSpeechRecognizer` | `NSSpeechRecognitionUsageDescription` |
| `PHPhotoLibrary` đọc thư viện | `NSPhotoLibraryUsageDescription` (`PhotosPicker` không cần) |
| **`UIActivityViewController` / `ShareLink` chia sẻ ảnh** (người dùng bấm "Save Image"), `UIImageWriteToSavedPhotosAlbum` | **`NSPhotoLibraryAddUsageDescription`** (hay bị quên nhất) |
| `ATTrackingManager` | `NSUserTrackingUsageDescription` |
| `CLLocationManager` | `NSLocationWhenInUseUsageDescription` |

```sh
grep -rnE "UIImagePickerController|AVCaptureDevice|AVAudioEngine|SFSpeechRecognizer|PHPhotoLibrary|UIActivityViewController|ShareLink|UIImageWriteToSavedPhotosAlbum|ATTrackingManager|CLLocationManager|CNContactStore|EKEventStore" --include='*.swift' .
grep -o "NS[A-Za-z]*UsageDescription" *.xcodeproj/project.pbxproj */Info.plist 2>/dev/null | sort -u
```

## 3. Các phép toán có thể trap

```sh
grep -rnE "try!|fatalError|preconditionFailure|as! " --include='*.swift' .
grep -rnE "[A-Za-z0-9_\)\]]![^=]" --include='*.swift' . | grep -v '"'      # force unwrap
grep -rn "uniqueKeysWithValues" --include='*.swift' .                    # key trùng thì crash
grep -rnE "\.removeFirst\(|\.removeLast\(|\.first!|\.last!|randomElement\(\)!" --include='*.swift' .
```

- **Index mảng:** `arr[i]` phải có guard `indices.contains(i)` hoặc `i < count`. Đặc biệt cẩn thận với index lưu trong `@State` khi dữ liệu nguồn thay đổi.
- **Range:** `a..<b` và `a...b` crash khi `a > b`. `Slider(in: 0...x)` crash khi `x` là NaN. Lưu ý `max(NaN, 0.01)` trả về NaN.
- **Int từ Double:** `Int(x)` crash khi `x` là NaN hoặc vô cực. Duration của audio, phép chia 0/0 và progress có thể ra NaN, nên kiểm tra `isFinite` trước.
- **`abs(Int.min)`** tràn số. Không lấy `abs(hashValue) % n`.
- **`Dictionary(uniqueKeysWithValues:)`** crash khi key trùng. Nếu dữ liệu lấy từ server, dùng `Dictionary(_:uniquingKeysWith:)`.
- **Parser cho file người dùng nhập** (zip, epub, docx, binary): mọi phép `subdata` hoặc `data[i]` phải kiểm tra biên với `data.count`. File bị cắt hoặc hỏng là chuyện bình thường.

## 4. `String(format:)` với chuỗi đã dịch

**Crash (`EXC_BAD_ACCESS`)** khi bản dịch có specifier khác với tham số, ví dụ `%@` nhưng truyền `Int`, hoặc có 2 specifier mà chỉ truyền 1 tham số. Lỗi chỉ xuất hiện khi máy dùng đúng ngôn ngữ đó.

Script: với mỗi key được dùng trong `String(format: strings.KEY, ...)`, so bộ specifier của **mọi** bản dịch với bản tiếng Anh.

```python
import re, glob, collections, subprocess
keys = set(re.findall(r"\.(\w+)$", subprocess.run(["bash","-c",
  "grep -rhoE 'String\\(format: *[a-zA-Z.]+\\.[a-zA-Z0-9_]+' --include='*.swift' ."],
  capture_output=True, text=True).stdout, re.M)) - {"current"}
spec = re.compile(r"%(?:(\d+)\$)?[-+ #0]*\d*(?:\.\d+)?(lld|ld|d|i|u|@|f|s)")
norm = lambda t: {"lld":"d","ld":"d","i":"d","u":"d"}.get(t, t)
def specs(s):
    return tuple(sorted((int(m[1]) if m[1] else i+1, norm(m[2]))
                        for i, m in enumerate(spec.finditer(s.replace("%%", "")))))
res = collections.defaultdict(set)
for f in glob.glob("**/*.swift", recursive=True):
    src = open(f, encoding="utf-8").read()
    for k in keys:
        for m in re.finditer(r'\b' + k + r'\s*[:=]\s*"((?:[^"\\]|\\.)*)"', src):
            res[k].add(specs(m.group(1)))
for k, v in res.items():
    if len(v) > 1: print("MISMATCH", k, v)
```

Sau đó đối chiếu specifier của bản tiếng Anh với kiểu tham số ở chỗ gọi.

## 5. ID trong `ForEach`

`ForEach(xs, id: \.self)` với dữ liệu có phần tử trùng sẽ vẽ sai, thiếu dòng hoặc giữ nhầm state. Trong `List` hoặc lazy stack, nó có thể gây crash.

```sh
grep -rnE "ForEach\([^)]*id: \\\\\.self" --include='*.swift' . | grep -vE "\.\.<|allCases|indices"
```

- Với dữ liệu từ server hoặc người dùng (các đáp án, tag, danh sách con), dùng `ForEach(Array(xs.enumerated()), id: \.offset)`, hoặc lọc trùng nếu phần tử trùng vốn không có ý nghĩa (ví dụ các ký tự lặp lại trong một từ).
- Với model `Identifiable` lấy từ truy vấn JOIN hoặc FTS, kiểm tra DB xem có trả về cùng một id nhiều lần không.

## 6. Hàm xử lý bị gọi lặp (double tap, bấm trong lúc đang await)

**Sai dữ liệu:** bấm nhanh 2 lần vào Continue/Submit thì nhảy qua một mục, hoặc bước "finalize" chạy 2 lần và gửi kết quả lên server 2 lần.

- Mọi hàm async được gọi từ nút bấm và có thay đổi state hoặc gọi mạng phải có cờ chặn (`guard !isSubmitting`). Cờ phải được bật **trước lần `await` đầu tiên**.
- Hàm chấm hoặc chuyển câu nên nhận **id của mục được trả lời** và bỏ qua nếu id đó không còn là mục hiện tại:

  ```swift
  func grade(itemId: Item.ID, correct: Bool) async {
      guard !isFinalizing, let item = currentItem, item.id == itemId else { return }
      ...
  }
  ```

## 7. Lỗi bị nuốt

- Controller có `@Published var errorMessage` nhưng view sở hữu nó không gắn `.toast(...)` hoặc `.alert(...)`. Khi đó người dùng bấm mà không thấy gì xảy ra.

  ```sh
  # các file tạo controller (đổi "Controller()" theo quy ước đặt tên của dự án) nhưng không đọc errorMessage
  for f in $(grep -rl "Controller()" --include='*.swift' .); do
    grep -q errorMessage "$f" || echo "$f"; done
  ```

- `catch` chỉ ghi `Logger` trong một luồng mà người dùng tin là đã lưu, ví dụ màn tổng kết hiện "xong" trong khi request thất bại. Cần báo lỗi lên UI.
- `try?` trên các thao tác ghi dữ liệu.
- `try rows.compactMap { try decode(...) }`: chỉ một dòng hỏng cũng làm hỏng cả danh sách. Cân nhắc bỏ qua riêng dòng hỏng và ghi log.

- **Giá trị mặc định của property KHÔNG cứu được `Decodable`.** `var mastered: Int = 0` vẫn ném `keyNotFound`
  khi JSON thiếu khoá đó (lỗi hiện ra là "Không thể đọc dữ liệu vì dữ liệu bị thiếu"). Chuyện này hay xảy ra
  khi cùng một model được dùng cho nhiều endpoint mà mỗi endpoint trả về một tập khoá khác nhau — request
  vẫn thành công trên server nhưng UI lại báo lỗi. Cách sửa: viết `init(from:)` và dùng `decodeIfPresent`
  hoặc khai báo optional:

  ```swift
  init(from decoder: Decoder) throws {
      let c = try decoder.container(keyedBy: CodingKeys.self)
      id = try c.decode(String.self, forKey: .id)
      mastered = try c.decodeIfPresent(Int.self, forKey: .mastered) ?? 0
  }
  ```

  ```sh
  # model Decodable có property không optional kèm giá trị mặc định
  grep -rn -B30 "= 0$\|= \[\]$\|= false$" Models/ | grep -E "struct .*Decodable|struct .*Codable"
  ```

## 8. Vòng đời view

- **Audio, timer, speech:** mọi view sở hữu player hoặc timer cần `.onDisappear { stop() }`. Nút X gọi `stop()` là chưa đủ, vì người dùng có thể vuốt back hoặc vuốt đóng sheet.
- **`@State` hoặc `@StateObject` khởi tạo từ tham số** (`_x = State(initialValue: param)`) chỉ lấy giá trị ở lần đầu. Nếu view được tái sử dụng với tham số mới, nó sẽ hiện dữ liệu cũ. Dùng `.id(param)` hoặc `.onChange(of: param)` để reset.
- Khi view con reset state trong `.onChange(of: id)`, body vẫn được render **một lần với dữ liệu mới và state cũ** trước khi onChange chạy. State dạng index trỏ vào dữ liệu mới có thể vượt biên.
- **State không bao giờ được dùng:** script tìm `@State` được gán nhưng không có binding `$x` và không được đọc ở đâu, tức nút bấm không có tác dụng:

  ```python
  import re, glob
  for f in glob.glob("**/*.swift", recursive=True):
      s = open(f, encoding="utf-8").read()
      for m in re.finditer(r"@State (?:private )?var (\w+)", s):
          n = m.group(1)
          uses = len(re.findall(r"(?<![\w.])\$?" + n + r"\b", s)) - 1   # minus the declaration
          binds = len(re.findall(r"\$" + n + r"\b", s))
          assigns = len(re.findall(r"(?<![\w$])(?:self\.)?" + n +
              r"\s*(?:=[^=]|\.toggle\(\)|\+=|-=|\.append|\.insert|\.remove)", s)) - 1
          if binds == 0 and uses - assigns - binds <= 0: print(f, n)
  ```

## 9. Âm thanh (`AVAudioSession` dùng chung toàn app)

- Session **chỉ có một cho cả app**. Module nào đổi category (ví dụ `.record` để nhận giọng nói) sẽ ảnh hưởng mọi module khác. Mỗi player phải đặt lại category **ngay trước mỗi lần `play()`**, không chỉ một lần trong `init` của singleton.
- `setActive(false)` **dừng mọi player đang chạy** trên session, kể cả nhạc nền của module khác. Chỉ tắt session khi chắc chắn không còn ai đang phát.
- `AVAudioEngine.inputNode.installTap`: nếu `outputFormat(forBus: 0)` có `sampleRate == 0` hoặc `channelCount == 0` (không có mic, mic đang bận vì cuộc gọi), hàm sẽ ném **NSException** và app crash. Guard trước khi gọi, và `removeTap` nếu `engine.start()` thất bại.

```sh
grep -rn "setCategory\|setActive\|installTap" --include='*.swift' .
```

## 10. Concurrency

- Khi bật `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`, mọi type mặc định là `@MainActor`. Code chạy trong `Task.detached` hoặc `DispatchQueue.global` mà gọi vào các type đó là vi phạm isolation. Ở Swift 5 mode, compiler chỉ báo warning (xem mục 0).
- Type thuần xử lý dữ liệu (parser, tokenizer, zip reader) mà cố ý chạy nền thì đánh dấu `nonisolated` ở mức type hoặc hàm.
- Nhóm `LocalizedError` có `errorDescription` đọc chuỗi localized thuộc main actor: để type lỗi ở isolation mặc định, chỉ đánh dấu hàm ném lỗi là `nonisolated`.
- Có thể thử runtime bằng biến môi trường `SWIFT_UNEXPECTED_EXECUTOR_LOG_LEVEL=2`: sai executor thì crash, `1` thì chỉ log. Bật thêm **Thread Sanitizer** và **Main Thread Checker** trong Scheme → Diagnostics.

## 11. Điều hướng và trình bày bị bỏ qua không báo

- `NavigationStack` lồng trong một view đã được push, hoặc `navigationDestination` đặt ngoài stack hay bên trong `List`/lazy container: SwiftUI chỉ log rồi bỏ qua.
- Mở sheet hoặc fullScreenCover trong khi một cái khác đang trình bày hoặc đang đóng: cái sau bị bỏ qua. Cần chờ khoảng 350 ms hoặc dùng `onDismiss`.
- Đổi qua lại giữa hai `NavigationStack` (ví dụ nhánh đăng nhập và nhánh khách) có thể crash navbar của UIKit. Chỉ nên có một nhánh sở hữu stack.

```sh
grep -rn "NavigationStack\|navigationDestination" --include='*.swift' .
```

## 12. Locale và ngôn ngữ

- `Locale.preferredLanguages` trả mã có script: `zh-Hant-TW`, `zh-Hans-CN`, `pt-BR`, `fil-PH`. So khớp theo tiền tố `"zh"` sẽ đưa người dùng tiếng Trung phồn thể sang giản thể. Cần xử lý `zh-Hant*` trước.
- Khi thêm ngôn ngữ mới, kiểm tra cả số nhiều, tên ngôn ngữ (`Locale.localizedString(forIdentifier:)`) và specifier của format (mục 4).
- Nếu app tự đổi ngôn ngữ trong app (không theo ngôn ngữ máy), mọi lần đổi nên đi qua **một hàm duy nhất** trong store giữ ngôn ngữ, và hàm đó gán giá trị **không có animation**:

  ```swift
  func apply(_ language: AppLanguage?) {
      var transaction = Transaction(animation: nil)
      transaction.disablesAnimations = true
      withTransaction(transaction) { selectedLanguage = language }
  }
  ```

  Đừng gán ngôn ngữ trong `withAnimation`, và đừng dựa vào `.animation(nil, value: languageCode)` ở view cha. View đọc store trực tiếp qua Observation (nhiều view nằm trong `NavigationStack` riêng) nên modifier ở cha không có tác dụng. Nếu lần đổi rơi vào một transaction có animation (khôi phục phiên lúc đang cuộn, đóng menu khi đăng xuất), chữ sẽ cross-fade giữa hai ngôn ngữ.
- Một bản sao dạng `static var currentCode` của ngôn ngữ (cho code không truy cập được environment) **không được observe**. Chỉ dùng nó cho lỗi, service và accessibility label; trong `body` hãy đọc ngôn ngữ từ store qua environment để view cập nhật khi đổi ngôn ngữ.

## 13. Thời điểm lúc mở app

Các bước chạy lúc khởi động thường **ngầm phụ thuộc vào thời gian** của nhau. Bỏ hoặc đổi thứ tự một bước có thể làm bước sau hỏng lặng lẽ.

- **Hộp xin quyền của hệ thống bị bỏ qua khi app chưa active.** Ví dụ, `ATTrackingManager.requestTrackingAuthorization` gọi lúc app chưa `.active` sẽ trả về ngay mà không hiện gì. Trước đó code có thể chạy đúng chỉ vì một bước khác (một request mạng, một form đồng ý) vô tình làm trễ đủ lâu. Khi bỏ bước đó thì hộp xin quyền biến mất. Hãy chờ app active một cách tường minh:

  ```swift
  if UIApplication.shared.applicationState != .active {
      for await _ in NotificationCenter.default.notifications(named: UIApplication.didBecomeActiveNotification) { break }
  }
  ```

- **Modal của app tranh chỗ với hộp thoại hệ thống.** Đừng tự mở sheet hay modal khi các bước "hỏi user" lúc khởi động (form đồng ý, ATT, thông báo) chưa xong. Hãy chờ một cờ "luồng khởi động đã xong", và cả trạng thái phiên đăng nhập nếu nội dung modal phụ thuộc vào nó (ví dụ ngôn ngữ của tài khoản).
- **Đừng hỏi user ở lần mở app đầu tiên** nếu lúc đó đã có hộp thoại khác. Nếu modal có nút "Để sau" thì cũng phải tính trường hợp user vuốt đóng (ví dụ bắt đầu thời gian chờ ngay khi modal hiện ra).
- Khi thử trên simulator, nếu kill app trong lúc hộp thoại hệ thống đang mở thì hộp đó có thể **còn sót lại** ở lần mở sau. Hãy đọc log để biết lần mở này có thật sự gọi API xin quyền hay không, đừng chỉ nhìn ảnh chụp.

## 14. Lần đầu dùng bị chậm (first-use latency)

Không crash, không sai dữ liệu, nhưng **lần đầu** mở một màn mất 2–4 giây, còn các lần sau thì nhanh. Lúc dev ít ai thấy, vì máy dev đã có sẵn cache và kết nối. Với app tra cứu, user chỉ cần gặp một lần là đủ để xoá app.

**Nguyên nhân hay gặp, thường cộng dồn với nhau:**

| Nguyên nhân | Dấu hiệu |
|---|---|
| Kết nối đầu tiên tới mỗi host: DNS + TLS (0,3–1s mỗi host trên 4G) | Request đầu tiên chậm, các request sau nhanh |
| Database ở backend nằm ở một vùng cố định (dù Worker hay edge function không có cold start) | Mục chưa có trong cache edge chậm hơn mục đã có |
| `WKWebView` đầu tiên phải khởi động tiến trình WebKit (vài trăm ms trên main thread) | Màn có web view (SVG, HTML, banner quảng cáo) giật ở lần đầu |
| SwiftUI dựng layout lần đầu và Swift runtime tra conformance lần đầu (nặng hơn hẳn ở Debug) | Main thread đứng vài trăm ms khi mở sheet lần đầu |
| **Loading toàn màn hình che dữ liệu đã có sẵn** | User nhìn màn trống, dù app đã có một phiên bản dữ liệu dùng được (ví dụ bản gốc chưa dịch) |

**Cách sửa, theo thứ tự hiệu quả:**

1. **Không che dữ liệu đã có.** Hiện ngay dữ liệu đang có (bản gốc, bản cache cũ), chỉ báo đang tải bằng một chỉ báo nhỏ không chặn thao tác, khi dữ liệu mới về thì thay vào. Đây là thay đổi lớn nhất về cảm giác.
2. **Làm nóng kết nối** ngay sau khi app mở (chờ 1–2 giây để màn đầu tiên ổn định): gửi một request `HEAD` nhẹ tới từng host mà màn quan trọng sẽ gọi.
   - **Connection pool là của từng `URLSession`.** Làm nóng qua `URLSession.shared` không giúp gì cho một service có `URLSession` riêng. Phải làm nóng qua đúng session mà service đó dùng.
3. **Làm nóng WebKit:** tạo sẵn một `WKWebView` ẩn (`loadHTMLString("")`) và **giữ nó suốt phiên**, để tiến trình web content luôn chạy sẵn.
4. **Tải trước** dữ liệu cho những gì user có khả năng bấm tiếp (ví dụ vài kết quả tìm kiếm đầu tiên), có debounce vì danh sách đổi theo từng phím gõ.
   - Thêm **chống gọi trùng** (in-flight dedupe): lưu `Task` đang chạy theo key. Nếu màn chi tiết mở ra khi request tải trước còn đang chạy thì `await` chính `Task` đó, không gửi request thứ hai.
   - Cho `Task` tải trước chạy độc lập (unstructured), để khi debounce huỷ thì kết quả vẫn được ghi vào cache.
   - Nếu một response trả về nhiều mục (ví dụ mọi mục cùng một từ khoá) thì cache hết các mục đó.
5. **Dữ liệu offline** (gói tải về, database đóng gói sẵn) giúp bỏ hẳn bước gọi mạng. Hãy chủ động mời user tải gói thay vì chỉ để một icon nhỏ.

**Cách đo** (đo trước khi sửa, đừng đoán):

- **Đánh dấu thời gian** bằng `os_log` tại các điểm: lúc bấm, `onAppear`, bắt đầu/kết thúc request, bắt đầu/kết thúc tạo web view. Thêm một **heartbeat main thread** (timer 20ms, log khi khoảng cách giữa hai lần chạy > 80ms) để bắt chỗ main thread bị đứng. Đặt một dấu chung (ví dụ `// PERF-TMP`) để gỡ sạch sau khi đo.

  ```swift
  nonisolated enum PerfTmp {
      nonisolated(unsafe) static var t0 = CACurrentMediaTime()
      static let log = Logger(subsystem: "perf.tmp", category: "PerfTmp")
      static func mark(_ s: String) {
          log.notice("PERF \(Int((CACurrentMediaTime() - t0) * 1000), privacy: .public)ms \(s, privacy: .public)")
      }
  }
  ```

- **Tự mở màn cần đo** bằng một launch argument chỉ dùng khi đo (ví dụ `-PerfDetail`): chờ vài giây sau khi mở app, rồi mở màn đó lần 1, đóng lại, mở lần 2 với **dữ liệu khác** để tách chi phí "lần đầu" khỏi chi phí "mỗi lần".
- **Gỡ app trước mỗi lần đo**, vì `URLCache` của lần chạy trước làm kết quả đẹp giả.

  ```sh
  xcrun simctl uninstall $DEV $BUNDLE && xcrun simctl install $DEV "$APP"
  xcrun simctl launch $DEV $BUNDLE -PerfDetail; sleep 20
  xcrun simctl spawn $DEV log show --last 30s --predicate 'subsystem == "perf.tmp"' --style compact
  ```

- **Time Profiler cho simulator:** attach bằng pid phía Mac *và* UDID của simulator, rồi export ra XML để đọc các frame nặng trên main thread.

  ```sh
  PID=$(pgrep -f "<App>.app/<App>" | head -1)
  xcrun xctrace record --template 'Time Profiler' --device $DEV --attach $PID --time-limit 6s --output tp.trace
  xcrun xctrace export --input tp.trace \
    --xpath '/trace-toc/run[@number="1"]/data/table[@schema="time-profile"]' --output tp.xml
  ```

- **Đo phía server riêng** bằng `curl -w "dns=%{time_namelookup} tls=%{time_appconnect} ttfb=%{time_starttransfer}"` để tách chi phí kết nối, thời gian truy vấn và cache edge (xem header `cf-cache-status`, `age`).
- Số đo trên simulator là **chặn dưới**. Máy thật dùng 4G và CPU yếu hơn sẽ chậm hơn nhiều, nên luôn thử lại trên máy thật với bản Release.

## 15. Thử tay trước khi phát hành

Các kịch bản mà phần lớn lỗi ngầm chỉ lộ ra khi chạy thật:

| Kịch bản | Bắt được |
|---|---|
| Đi mọi lối vào của một tính năng, không chỉ lối chính (ví dụ một màn luyện tập mở từ Home, từ danh sách yêu thích và từ một màn khác) | Thiếu inject environment |
| Tắt mạng, hoặc bị giới hạn tần suất (429), rồi bấm mọi nút gọi mạng | Lỗi bị nuốt |
| Từ chối từng quyền, rồi dùng tính năng đó | Luồng xử lý khi bị từ chối quyền |
| Share ảnh → Save Image | Thiếu `NSPhotoLibraryAddUsageDescription` |
| Bấm nhanh 2 lần mọi nút Submit, Continue, Buy | Hàm bị gọi lặp |
| Vuốt đóng sheet hoặc vuốt back trong lúc audio đang phát | Thiếu stop trong `onDisappear` |
| Chuyển giữa các module có âm thanh (ghi âm → phát) | Category của session |
| Đổi ngôn ngữ máy sang từng ngôn ngữ hỗ trợ, nhất là zh-Hant | Format, locale |
| Máy để English, đăng nhập tài khoản có ngôn ngữ khác, tắt hẳn app rồi mở lại và cuộn ngay | Chữ cross-fade giữa hai ngôn ngữ khi khôi phục phiên |
| Nhập file hỏng hoặc rỗng (pdf, epub, docx, zip đổi đuôi) | Parser |
| Gọi điện hoặc mở app khác đang giữ mic rồi dùng tính năng thu âm | `installTap` |
| Chạy bản **Release** trên máy thật | Nhánh `#if DEBUG`, test ad unit, URL localhost |
| **Cài mới**, dùng 4G, mở màn chi tiết quan trọng lần đầu tiên | Lần đầu chậm (mục 14) |
| Cài mới, mở app lần 1 và lần 2, không bấm gì lúc khởi động | Hộp xin quyền bị bỏ qua, modal chồng lên hộp thoại hệ thống (mục 13) |
