# binh-swift-ui-bonus-skill

Skill cho [Claude Code](https://claude.com/claude-code), dùng cho mọi dự án SwiftUI / iOS. Nó gom lại các bài học "build OK nhưng chạy vẫn hỏng" để Claude tự áp dụng khi làm việc trong dự án SwiftUI, không cần nhắc.

## Nội dung

| File | Nội dung |
|---|---|
| `SKILL.md` | Phần Claude đọc: mô tả khi nào dùng skill, các quy tắc luôn áp dụng, và khi nào mở file tham khảo nào |
| `references/silent-bugs-checklist.md` | Checklist lỗi ngầm: environment, quyền trong Info.plist, phép toán có thể crash, `String(format:)`, `ForEach` id, bấm lặp, lỗi bị nuốt, vòng đời view, âm thanh, concurrency, điều hướng, locale, thời điểm lúc mở app, **lần đầu dùng bị chậm (kèm cách đo)**, kịch bản thử tay trước khi phát hành |
| `references/api-caching-guide.md` | Hướng dẫn cache cho API backend: các lớp cache, thiết kế key, những gì không được cache, Cloudflare Cache API, KV / R2 / Redis, cache trên client (RAM, đĩa, `URLCache`, tải trước), chống lạm dụng, cách đo |

Hai file tham khảo viết bằng tiếng Việt và dùng được độc lập, không cần Claude.

## Cài đặt trên máy mới

Skill cấp user nằm ở `~/.claude/skills/<tên-skill>/`, áp dụng cho mọi dự án.

**Cách 1: clone thẳng vào thư mục skills** (đơn giản nhất)

```sh
mkdir -p ~/.claude/skills
git clone <url-repo> ~/.claude/skills/binh-swift-ui-bonus-skill
```

Cập nhật về sau: `git -C ~/.claude/skills/binh-swift-ui-bonus-skill pull`.

**Cách 2: clone vào chỗ khác rồi tạo symlink** (nếu muốn sửa skill ngay trong thư mục dự án của mình)

```sh
git clone <url-repo> "<thư-mục-của-bạn>/binh-swift-ui-bonus-skill"
mkdir -p ~/.claude/skills
ln -s "<thư-mục-của-bạn>/binh-swift-ui-bonus-skill" ~/.claude/skills/binh-swift-ui-bonus-skill
```

Mở một phiên Claude Code mới, gõ `/` và kiểm tra `binh-swift-ui-bonus-skill` có trong danh sách skill.

### Muốn Claude luôn dùng skill trong dự án SwiftUI

Claude tự dùng skill khi thấy việc đang làm khớp với phần `description` trong `SKILL.md`, nhưng không đảm bảo lần nào cũng dùng. Muốn chắc chắn thì thêm vào `~/.claude/CLAUDE.md`:

```md
- Trong mọi dự án SwiftUI / Xcode, luôn dùng skill `binh-swift-ui-bonus-skill`.
```

## Cập nhật nội dung

- Sửa trực tiếp các file trong repo này, rồi commit và push.
- Nếu một dự án có bản `docs/silent-bugs-checklist.md` hoặc `docs/api-caching-guide.md` mới hơn, copy nó vào `references/` của repo này.
- Giữ nội dung **chung chung**: không đưa tên class, đường dẫn hay tính năng riêng của một dự án vào đây.
- Phần `description` trong `SKILL.md` quyết định khi nào Claude dùng skill. Nên giữ dưới 1024 ký tự và liệt kê rõ các tình huống cần kích hoạt.
