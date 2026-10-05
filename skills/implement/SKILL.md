---
name: implement
description: Implement một feature theo artifact đã duyệt trong .artifact/<feature>/, tuân theo cấu trúc code và rule của project; gặp gap thì dừng lại hỏi. Chỉ chạy khi user gọi /phong-dn:implement.
argument-hint: "[feature-slug]"
disable-model-invocation: true
---

# /phong-dn:implement

Feature: $ARGUMENTS

## Bước 1 — Tìm artifact

- Có tham số → dùng `.artifact/<tham số>/`. Không tồn tại → liệt kê các thư mục gần giống, hỏi user.
- Không có tham số → đọc dòng `Status:` của mọi `.artifact/*/README.md`, liệt kê các feature `approved` / `in-progress`, hỏi user chọn (AskUserQuestion). Chỉ một feature → xác nhận nhanh với user.
- `Status: draft` → dừng, đề nghị user duyệt qua `/phong-dn:plan` trước. `done` → hỏi user có chắc muốn làm tiếp không.

## Bước 2 — Hiểu nhanh artifact

Đọc theo thứ tự, dừng khi đủ hiểu:
1. `README.md` — mục tiêu, phạm vi, AC, quyết định, Open questions còn mở, Deviations.
2. `02-design/technical-approach.md` — các bước, bước nào đã tick (đang làm dở từ session trước → tiếp tục từ bước chưa tick đầu tiên).
3. Chỉ phần liên quan tới bước đang làm: `03-contracts/` khi đụng API, `04-data/` khi đụng DB, `02-design/adr/` khi có quyết định kiến trúc.

Còn Open question chưa trả lời và nó chặn bước sắp làm → hỏi user trước.

## Bước 3 — Hiểu rule và cấu trúc code

- Rule: `CLAUDE.md`, `AGENTS.md` (gốc repo và thư mục con liên quan), `docs/agents/`, ADR liên quan. Tóm tắt nhanh cho bản thân các rule áp dụng cho feature này (kiến trúc layer, naming, docs bắt buộc, null safety, checkstyle, quy tắc DB/query...).
- Cấu trúc: với mỗi file trong "Ảnh hưởng", xem 1–2 file **cùng loại đã có** (controller/service/adapter/migration/test tương tự) và làm theo đúng pattern đó. CodeGraph nếu repo có `.codegraph/`; project có Serena MCP thì ưu tiên theo CLAUDE.md.
- Lệnh kiểm chứng: lấy từ `technical-approach.md` → `CLAUDE.md` → Taskfile/Makefile/build file.

## Bước 4 — Branch

- Đang ở nhánh chính/phát triển (`main`, `master`, `develop`) → tạo branch theo quy ước trong CLAUDE.md/AGENTS.md; mặc định `feature/<feature-slug>` tách từ `develop` nếu có, không thì từ nhánh mặc định.
- Đang ở branch khác → hỏi user có làm trên branch hiện tại không.
- **Không tự commit.** User tự commit sau `/phong-dn:final-check`.
- Đổi `Status: in-progress` trong README.

## Bước 5 — Làm theo từng bước

Với mỗi bước chưa tick trong technical approach:
1. Implement theo pattern có sẵn và rule ở Bước 3. Không thêm thứ artifact không yêu cầu.
2. Viết test theo AC tương ứng, cùng style test đang có.
3. Tick `[x]` bước đó ngay khi xong (để session sau làm tiếp được).
4. Làm khác technical approach (đổi thứ tự, gộp/tách bước, đổi hướng nhỏ) → ghi vào `## Deviations` trong README: `- <bước>: <khác gì> — <vì sao>`.

### Gặp gap → dừng lại hỏi ngay

Gap là khi: artifact không nói tới mà quyết định ảnh hưởng hành vi/contract/DB; code chưa có pattern cho loại việc này; rule mâu thuẫn với artifact; hoặc phải làm khác lớn so với technical approach.

1. Ghi vào `## Open questions` trong README: `- [ ] Qn: <câu hỏi> — đề xuất: <phương án> vì <lý do>`.
2. Hỏi user ngay (AskUserQuestion, phương án đề xuất đặt đầu). **Không tự đoán rồi làm tiếp.**
3. Có câu trả lời → tick câu hỏi, ghi câu trả lời cạnh đó, cập nhật artifact nếu nó đổi thiết kế (AC, contracts, data...), rồi làm tiếp.

## Bước 6 — Kiểm chứng một lần

- Chỉ chạy lệnh check/test **sau khi làm xong toàn bộ các bước** — không chạy sau từng chỉnh sửa nhỏ.
- Có lỗi → gom toàn bộ, sửa một lượt, chạy lại. Lặp đến khi pass.
- Không chạy được (thiếu Docker, DB...) → nói rõ lệnh nào không chạy và vì sao, không báo là đã pass.

## Bước 7 — Báo cáo

Ngắn gọn: các bước đã xong, Deviations (nếu có), kết quả check/test (pass/fail thật), file đã đổi. Gợi ý chạy `/phong-dn:final-check <feature-slug>`. Giữ `Status: in-progress` (final-check mới chuyển sang done).
