---
name: final-check
description: Kiểm tra độc lập code của một feature trước khi merge — đối chiếu với rule project, với artifact .artifact/<feature>/, và săn lỗi tiềm ẩn (escape, độ dài chuỗi, N+1, quyền, lộ dữ liệu...). Ghi báo cáo và hỏi user trước khi sửa. Chỉ chạy khi user gọi /phong-dn:final-check.
argument-hint: "[feature-slug]"
disable-model-invocation: true
---

# /phong-dn:final-check

Feature: $ARGUMENTS

## Bước 1 — Xác định phạm vi

- Artifact: `.artifact/<feature-slug>/`. Không có tham số → lấy feature `in-progress` (nhiều cái thì hỏi user). Không có artifact nào → hỏi user có muốn check chỉ theo rule (bỏ nhóm B) không.
- Base: nhánh mà branch hiện tại tách ra (`develop` nếu có, không thì nhánh mặc định). Diff = `git merge-base <base> HEAD`..working tree, **gồm cả thay đổi chưa commit và file untracked** (`git status --porcelain`).
- Lệnh kiểm chứng: từ `technical-approach.md` → `CLAUDE.md` → Taskfile/Makefile/build file.

## Bước 2 — Chạy song song

**(a) Subagent review độc lập** — Agent tool, `subagent_type: general-purpose`, prompt theo mẫu dưới. Subagent có context mới nên không bị thiên vị bởi việc chính mình vừa viết code. Chỉ đưa đường dẫn, không dán nội dung code vào prompt.

**(b) Trong lúc đó tự chạy** lệnh check/test của project (Bash, `run_in_background` nếu lâu). Ghi nhận pass/fail và lỗi thật. Không chạy được → ghi rõ lý do.

### Prompt cho subagent

```
Bạn là reviewer độc lập cho feature "<feature-slug>" ở repo <đường dẫn repo>.
Chỉ đọc và báo cáo, KHÔNG sửa file.

Đầu vào:
- Diff: chạy `git diff <merge-base>` và `git status --porcelain` (đọc cả file untracked).
- Artifact: <đường dẫn .artifact/<feature-slug>/> — README.md (AC, quyết định, Deviations), 02-design/technical-approach.md, và 03-contracts/, 04-data/ nếu có.
- Rule: CLAUDE.md, AGENTS.md, docs/agents/, ADR liên quan.

Chỉ review code trong diff (đọc thêm code xung quanh khi cần để hiểu). Kiểm 3 nhóm:

A. Rule project — mọi rule bắt buộc trong CLAUDE.md/AGENTS.md: ranh giới kiến trúc/layer, naming, docs/JavaDoc bắt buộc, null safety, checkstyle, package/file bắt buộc, quy ước test, quy ước query/DB. Trích đúng rule bị vi phạm.

B. Logic so với artifact
- Từng AC: đã hiện thực chưa, đúng hành vi chưa, có test chứng minh chưa (chỉ ra tên test).
- Code làm điều artifact không yêu cầu, hoặc lệch thiết kế mà không ghi trong Deviations.
- API/DTO/event có khớp 03-contracts (path, method, field, kiểu, mã lỗi, ràng buộc) không.
- Migration/schema có khớp 04-data không.

C. Lỗi tiềm ẩn
- Escape/encode theo ngữ cảnh output: SQL (nối chuỗi input vào query, field không qua whitelist), HTML, URL, log injection (CR/LF), JSON, path traversal.
- Độ dài chuỗi: mọi String từ ngoài vào (DTO, query param, path var) và trong domain có giới hạn khớp cột DB không.
- Truy vấn: N+1, thiếu index cho điều kiện lọc mới, query không phân trang.
- Bảo mật: thiếu kiểm tra quyền/ownership, IDOR, lộ dữ liệu nhạy cảm qua log/response/exception message, secret hard-code.
- Null & edge case: input rỗng, list rỗng, giá trị biên, timezone, số âm, trùng lặp.
- Transaction & đồng thời: thao tác nhiều bước không cùng transaction, race condition, idempotency.
- Migration: rollback được không, khoá bảng lớn, backfill an toàn.
- Error handling: nuốt exception, mã lỗi HTTP sai, thông điệp lỗi lộ nội bộ.

Với mỗi vấn đề, chỉ báo khi đã xác minh trong code (không đoán). Trả về danh sách, xếp nghiêm trọng nhất trước:
- [Critical|High|Medium|Low] <Nhóm A/B/C> `path:line` — <vấn đề> — <kịch bản gây lỗi cụ thể> — <hướng sửa 1 dòng>
Cuối cùng: bảng AC → trạng thái (đạt / thiếu test / chưa làm / sai) → test chứng minh.
```

## Bước 3 — Ghi báo cáo

Ghi `.artifact/<feature-slug>/05-testing/final-check.md` (tạo thư mục nếu chưa có; đã có file cũ thì ghi đè, git giữ lịch sử nếu artifact được commit):

```markdown
# Final check — <Tên feature>
Date: <YYYY-MM-DD> · Branch: <branch> · Base: <base>

## Kết quả kiểm chứng
- `<lệnh check>`: pass | fail (<tóm tắt lỗi>) | không chạy (<lý do>)
- `<lệnh test>`: ...

## Acceptance criteria
| AC | Trạng thái | Test |
|---|---|---|

## Findings
| # | Mức | Nhóm | Vị trí | Vấn đề | Hướng sửa | Trạng thái |
|---|---|---|---|---|---|---|
```

Trước khi ghi, tự xác minh nhanh các finding Critical/High (đọc đúng dòng code) — finding sai thì bỏ, đừng chuyển tiếp nguyên văn.

## Bước 4 — Hỏi trước khi sửa

- Không có finding và check/test pass → đổi `Status: done` trong README, báo user có thể commit/merge.
- Có finding → tóm tắt (số lượng theo mức + top Critical/High), hỏi user sửa những cái nào (AskUserQuestion, multiSelect; gợi ý: sửa hết Critical/High). **Không sửa khi chưa được đồng ý.**
- Sửa xong → chạy lại check/test một lần, cập nhật cột trạng thái trong báo cáo (fixed / skipped). Chỉ đổi `Status: done` khi không còn Critical/High chưa xử lý và check/test pass.
