---
name: plan
description: Biến ý tưởng/yêu cầu thô thành bộ artifact thiết kế trong .artifact/<feature>/ (README + technical approach + các phần cần thiết). Khảo sát code, hỏi user một lượt các câu làm rõ, rồi ghi artifact. Chỉ chạy khi user gọi /phong-dn:plan.
argument-hint: <ý tưởng hoặc yêu cầu thô>
disable-model-invocation: true
---

# /phong-dn:plan

Input của user: $ARGUMENTS

Mục tiêu: một bộ artifact đủ rõ để `/phong-dn:implement` làm hết mà không phải đoán. **Không viết code trong skill này.**

**Ngôn ngữ**: viết mọi câu hỏi và artifact bằng ngôn ngữ user dùng trong input; giữ nguyên thuật ngữ kỹ thuật tiếng Anh.

## Bước 1 — Khảo sát trước khi hỏi

- Đọc rule project (file nào có thì đọc): `CLAUDE.md`, `AGENTS.md`, `CONTEXT.md`, `docs/adr/`, `docs/agents/`.
- Tìm code liên quan (CodeGraph nếu repo có `.codegraph/`, không thì grep/đọc file). Ghi nhận: module/file/symbol sẽ bị động tới, pattern có sẵn để tái dùng, bảng DB/cột liên quan, API hiện có.
- Xem `.artifact/` đã có feature nào liên quan chưa (tránh trùng, hoặc mở rộng cái cũ).
- Điều gì trả lời được từ code/docs thì **không hỏi user**.

## Bước 2 — Soạn bộ câu hỏi

Gom TẤT CẢ điểm chưa rõ thành một danh sách. Chỉ hỏi điều thực sự là quyết định của user: hành vi nghiệp vụ, edge case, phạm vi (làm / không làm), đánh đổi giữa các phương án kỹ thuật.

Mỗi câu theo schema:

```json
{
  "id": "q1",
  "question": "Câu hỏi ngắn, kết thúc bằng dấu ?",
  "context": "Vì sao phải hỏi + agent thấy gì trong code (1–3 câu)",
  "multi": false,
  "options": [
    { "label": "Phương án A", "description": "Hệ quả / trade-off", "recommended": true, "preview": "<optional HTML: mockup, bảng, sơ đồ nhỏ>" },
    { "label": "Phương án B", "description": "..." }
  ]
}
```

Luôn có đúng một phương án `recommended` (đặt đầu tiên), dựa trên code đã đọc. User luôn có thể chọn "Khác" + ghi chú.

Thêm câu hỏi cố định cuối danh sách: **"Có commit thư mục `.artifact/<feature-slug>/` lên git không?"** (Có / Không — không thì thêm thư mục vào `.gitignore`).

## Bước 3 — Chọn chế độ hỏi cho session này

Hỏi user bằng AskUserQuestion: "Dùng chế độ nào để trả lời bộ câu hỏi?" với 4 lựa chọn **HTML template / Lavish / Claude Artifact / Terminal**, mỗi lựa chọn kèm 1 dòng trade-off lấy từ `question-modes.md` (cùng thư mục skill này). Nếu bộ câu hỏi ≤ 4 câu đơn giản, đặt Terminal là Recommended; còn lại đặt HTML template là Recommended.

Sau khi user chọn, đọc mục tương ứng trong `question-modes.md` và làm theo.

## Bước 4 — Hỏi một lượt, tối đa thêm một lượt bổ sung

- Gửi toàn bộ câu hỏi theo chế độ đã chọn, chờ câu trả lời.
- Câu trả lời làm phát sinh điểm mù mới → hỏi thêm **tối đa 1 lượt** nữa (cùng chế độ). Sau đó còn gì chưa rõ thì tự chọn phương án hợp lý nhất và ghi vào "Quyết định".
- User trả lời "bạn quyết"/bỏ trống → dùng phương án recommended, ghi rõ là agent tự chọn.

## Bước 5 — Ghi artifact

Thư mục: `.artifact/<feature-slug>/` (slug kebab-case, ngắn, tiếng Anh không dấu).

**Luôn tạo** hai file sau. Các thư mục còn lại **chỉ tạo khi feature cần** (bảng ở dưới).

### `README.md`

```markdown
# <Tên feature>

Status: draft   <!-- draft → approved → in-progress → done -->
Owner: <git user.name>
Created: <YYYY-MM-DD>
Commit artifact: yes | no

## Mục tiêu
<1–3 câu: vấn đề gì, cho ai, vì sao bây giờ>

## Phạm vi
- Làm: ...
- Không làm: ...

## Acceptance criteria
- [ ] AC1: Given ... When ... Then ...
- [ ] AC2: ...

## Quyết định
- <quyết định> — <lý do> (user chốt | agent tự chọn)

## Mục lục
- [Technical approach](02-design/technical-approach.md)
- <link tới các phần khác đã tạo>

## Open questions
<!-- implement ghi gap vào đây: - [ ] Qn: câu hỏi — đề xuất: ... -->

## Deviations
<!-- implement ghi các chỗ làm khác technical approach: - <bước>: <khác gì> — <vì sao> -->
```

Nếu có > 6 acceptance criteria hoặc cần user story chi tiết, chuyển AC sang `01-requirements/requirements.md` và để README chỉ link tới.

### `02-design/technical-approach.md`

```markdown
# Technical approach — <Tên feature>

## Hướng tiếp cận
<1 đoạn: giải pháp tổng thể, vì sao chọn hướng này thay vì hướng khác>

## Ảnh hưởng
- Module/layer: ...
- File/symbol chính: `path` — thay đổi gì
- Tái dùng: <helper/pattern có sẵn>

## Các bước
<!-- Mỗi bước 1 dòng mô tả VIỆC CẦN LÀM, không mã giả. implement sẽ tick khi xong. -->
- [ ] 1. ...
- [ ] 2. ...
- [ ] N. Viết test cho AC1..ACn

## Rủi ro
- <rủi ro> — <cách giảm>

## Kiểm chứng
- Lệnh: <lệnh check/test của project, lấy từ CLAUDE.md/Taskfile/build file>
```

### Thư mục tùy chọn — chỉ tạo khi cần

| Thư mục | Tạo khi |
|---|---|
| `01-requirements/` | Nhiều user story/AC, hoặc user đưa PRD/tài liệu nghiệp vụ |
| `02-design/adr/ADR-NNN-<slug>.md` | Có quyết định kiến trúc đáng ghi lại (nhiều phương án, khó đảo ngược). Nếu repo đã có `docs/adr/`, hỏi user nên đặt ADR ở đâu |
| `02-design/diagrams/` | Luồng nhiều bước/nhiều thành phần (Mermaid trong `.md`) |
| `03-contracts/` | Thêm/đổi API, event, DTO — ghi endpoint, request/response, mã lỗi, ràng buộc độ dài |
| `04-data/` | Thêm/đổi bảng, cột, index, migration, backfill |
| `05-testing/` | Cần test plan riêng (dữ liệu test đặc biệt, performance). `final-check` sẽ ghi báo cáo vào đây |
| `06-operations/` | Có feature flag, cấu hình mới, cần rollback plan hoặc monitoring |
| `07-release/` | Thay đổi user nhìn thấy cần release note / ghi chú QA |

File user đính kèm (PDF, ảnh...) đặt trong `attachments/` của thư mục tương ứng.

Nếu user chọn **không commit**: thêm dòng `.artifact/<feature-slug>/` vào `.gitignore` của repo.

## Bước 6 — Duyệt

Tóm tắt trong ≤ 10 dòng: mục tiêu, các AC, các bước chính, thư mục đã tạo. Hỏi user duyệt.
- Duyệt → đổi `Status: approved`, gợi ý chạy `/phong-dn:implement <feature-slug>`.
- Có sửa → cập nhật artifact rồi hỏi lại.
