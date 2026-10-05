# Chế độ hỏi user

Bảng trade-off (dùng khi hỏi user chọn chế độ):

| Chế độ | Trả lời về agent | Trực quan | Phụ thuộc | Token |
|---|---|---|---|---|
| HTML template | User tải `answers.json` (hoặc copy-paste) | Cao — mockup, bảng, sơ đồ trong preview | Không | Thấp — agent chỉ điền JSON |
| Lavish | Tự động (long polling) | Cao nhất — annotate từng phần tử, sửa Mermaid | Node/npx + tool bên thứ 3 | Thấp–vừa |
| Claude Artifact | Tự động qua database của artifact | Cao, mở được trên điện thoại | claude.ai, dữ liệu nằm trên cloud | Vừa |
| Terminal | Tự động | Thấp — text, ≤ 4 câu/lượt, ≤ 4 lựa chọn/câu | Không | Thấp nhất |

Dòng mô tả ngắn cho AskUserQuestion:
- **HTML template** — form trong trình duyệt, có preview trực quan; tải answers.json về là xong. Không phụ thuộc gì.
- **Lavish** — annotate trực tiếp lên HTML, phản hồi tự về agent. Cần npx, tool bên thứ 3.
- **Claude Artifact** — form trên claude.ai, trả lời được từ điện thoại. Dữ liệu nằm trên cloud.
- **Terminal** — menu ngay trong terminal. Nhanh nhất, kém trực quan nhất.

---

## HTML template

1. Copy `questions.html` (cùng thư mục với file này) tới `.artifact/<feature-slug>/questions.html`.
2. Thay đúng một chỗ: dòng `const DATA = {...};` → điền `feature` (slug), `title` (tên feature) và `questions` (mảng câu hỏi theo schema trong SKILL.md). Không sửa phần còn lại của template.
3. Mở file cho user: `xdg-open <path>` (Linux) / `open <path>` (macOS). Lỗi thì in đường dẫn để user tự mở.
4. Nói với user: trả lời xong bấm **Tải answers.json** (file vào thư mục Downloads với tên `answers-<feature-slug>.json`) hoặc **Copy** rồi dán vào terminal.
5. Chờ user báo xong. Đọc `~/Downloads/answers-<feature-slug>.json` (hoặc nội dung user dán). Chuyển file vào `.artifact/<feature-slug>/answers.json` để lưu vết. Lượt hỏi bổ sung: tạo `questions-2.html` từ template, đặt `feature` = `<feature-slug>-2` (file tải về sẽ là `answers-<feature-slug>-2.json`), lưu thành `answers-2.json`.

## Lavish

1. Chạy `npx -y lavish-axi --help` và làm theo hướng dẫn hiện tại của CLI (lệnh thay đổi theo phiên bản — không đoán cú pháp).
2. Viết bộ câu hỏi thành một trang HTML (mỗi câu: câu hỏi, context, các phương án + preview, đánh dấu recommended) tại `.artifact/<feature-slug>/questions.html`, mở bằng Lavish.
3. Chờ phản hồi của user qua Lavish, tổng hợp thành `.artifact/<feature-slug>/answers.json` theo dạng ở mục "Định dạng câu trả lời".
4. npx/Lavish lỗi → báo user và chuyển sang chế độ HTML template.

## Claude Artifact

Chỉ dùng được khi session có tool `Artifact`. Không có → báo user, đề nghị chế độ khác.

1. Load skill `artifact-capabilities` để khai báo capability lưu trữ dùng chung (db) cho trang.
2. Dựng form (có thể dựa trên `questions.html`) lưu mỗi câu trả lời thành một document trong collection `answers`.
3. Publish, đưa link cho user. User báo xong → đọc collection `answers` bằng `ArtifactData`, ghi lại thành `.artifact/<feature-slug>/answers.json`.

## Terminal

Dùng AskUserQuestion, chia bộ câu hỏi thành các lượt ≤ 4 câu (đó vẫn tính là "một lượt hỏi" theo nghĩa của workflow). Phương án recommended đặt đầu, nhãn kết thúc bằng "(Recommended)". `preview` dùng ASCII/markdown thay cho HTML.

---

## Định dạng câu trả lời

```json
{
  "feature": "<feature-slug>",
  "answers": [
    { "id": "q1", "question": "...", "selected": ["Phương án A"], "other": "", "note": "" }
  ]
}
```

`selected` rỗng và `other` rỗng = user để agent tự quyết → dùng phương án recommended.
