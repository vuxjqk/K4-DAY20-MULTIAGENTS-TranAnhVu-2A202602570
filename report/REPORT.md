# Báo cáo Lab: Self evolving Agentic

## 1. Thông tin nhóm và cấu hình

| Họ tên | Mã sinh viên | Phần đóng góp |
|---|---|---|
| Trần Anh Vũ | 2A202602570 | Toàn bộ (cá nhân) |

- Mô hình: `LAB_MODEL=google_genai:gemini-3.5-flash-lite` (Google AI Studio, free tier) qua `langchain-google-genai` 4.4.0. `LAB_TEMPERATURE=0` nhưng **không có hiệu lực**: thư viện cảnh báo "Model 'gemini-3.5-flash-lite' uses fixed sampling defaults; the sampling parameter(s) temperature will be ignored" — mô hình lấy mẫu ngẫu nhiên. `recursion_limit` = 60 (mặc định).
- Lý do không dùng Option 1 (cổng tương thích OpenAI của Gemini): Gemini 3.x yêu cầu gửi lại `thought_signature` kèm mỗi function call; `ChatOpenAI` làm rơi trường này nên lần gọi công cụ thứ hai luôn lỗi 400 `Function call is missing a thought_signature`. Tích hợp gốc `google_genai` (Option 2 của `model.py`, không sửa `model.py`) xử lý đúng. Đã thêm `langchain-google-genai>=4.0` vào `pyproject.toml`.
- Deep Agents 0.7.21. Máy chủ Windows 11; mọi lệnh chạy **trong Docker** (`python:3.12-slim`, image build từ `Dockerfile` của kho) vì shell của tác tử cần `/bin/sh`:
  `docker run --rm --env-file .env -e PYTHONDONTWRITEBYTECODE=1 -v <repo>:/lab lab-deepagents python -m lab.runner ...`
- Số lần chạy tác vụ: xem Phụ lục.
- Commit của tag `freeze`: (điền sau Phần 4.1)

## 2. Giả thuyết (commit TRƯỚC tag `freeze`, Phần 4.0)

- H1 (subagents so với baseline): `subagents` sẽ có điểm trung bình trên tác vụ đánh giá **không cao hơn** (dự đoán thấp hơn) `baseline`, với token trung bình cao gấp khoảng 3 lần. Căn cứ: trên tác vụ học, subagents đạt 13/18 check kỹ thuật so với 18/18 và 364.653 so với 109.701 token (×3,3); cơ chế là cô lập ngữ cảnh — lời giao việc bỏ sót quy tắc của đề (data-learn: thiếu tên khóa `answer.json`, reviewer trả PASS sai). Báo cáo của Anthropic về hệ thống nghiên cứu đa tác tử ghi nhận chi phí khoảng ×15 so với hội thoại, và lợi ích chỉ đến ở tác vụ song song hóa được; ba tác vụ ở đây tuần tự và nhỏ.
- H2 (skills-auto so với baseline): `skills-auto` sẽ cao hơn `baseline` **chỉ ở họ `code`** (dự đoán +1 đến +2 check quy ước nhờ skill `comprehensive-regression-testing-and-changelogging` mã hóa các quy ước test hồi quy/CHANGELOG mà tác vụ đánh giá được nói là dùng lại), còn ở `data` và `logs` sẽ ngang baseline vì skill chung `adhere-to-strict-rule-specifications` không chứa giá trị quy ước. Check kỹ thuật không đổi (đã gần tối đa). Căn cứ: Phần 3.4 — code-learn 9/10 (baseline 7/10), data-learn 5/8 và logs-learn 6/9 bằng baseline dù `skills_read = 2`; SkillsBench ghi nhận skill do mô hình tự sinh trung bình không có lợi, nên kỳ vọng hiệu ứng nhỏ và cục bộ.
- H3 (tác vụ học so với tác vụ đánh giá): mức cải thiện của `skills-auto` trên tác vụ đánh giá sẽ **nhỏ hơn** trên tác vụ học, và quy ước **mới** của mỗi tác vụ đánh giá sẽ thất bại ở cả ba điều kiện vì không có phản hồi nào về nó trong dữ liệu học. Căn cứ: SkillEvolBench ghi nhận lợi ích trên tác vụ học thường không chuyển sang tác vụ mới; skill được rút ra từ đúng các check của tác vụ học nên có xu hướng quá khớp.

## 3. Làm quen Deep Agents (Phần 0.3)

1. `python scripts/tour.py` liệt kê 9 công cụ: công cụ tệp `ls`, `read_file`, `write_file`, `edit_file`, `delete`, `glob`, `grep`; shell `execute`; giao việc `task`. Công cụ cho phép chạy lệnh là `execute` (chạy lệnh shell thật trong `root_dir` của backend). `task` gián tiếp cũng chạy lệnh vì subagent `general-purpose` có cùng bộ công cụ.
2. Mô tả `task` nói `general-purpose` là "General-purpose agent for researching complex questions, searching for files and content, and executing multi-step tasks ... This agent has access to all tools as the main agent". Về ngữ cảnh: "Each invocation is stateless by default: the agent sees only the prompt you give it and returns a single final report" — subagent **không** thấy lịch sử hội thoại hay system prompt của tác tử chính, chỉ thấy nội dung lời giao việc.
3. Từ `task`: "Put full detail in the prompt and state exactly what it should return". Từ `execute`: "You MUST avoid using search commands like find and grep. Instead use the grep, glob tools to search. Use read_file rather than cat/head/tail."

## 4. Đường cơ sở và phân loại lỗi (Phần 2.2)

Dữ liệu: `results/baseline/{code,data,logs}-learn/run.json` và `trace.md`. Điểm: code-learn 7/10, data-learn 5/8, logs-learn 6/9.

| Tác vụ | Check thất bại | Nhóm lỗi (A-G) | Bằng chứng (trích ngắn từ `detail` hoặc vết) |
|---|---|---|---|
| code-learn | `rule_type_hints` | E | "RULE: every public function ... has type annotations on all parameters and on the return value." Đề chỉ nói "checked ... against the Acme Python team conventions", không nêu nội dung. |
| code-learn | `rule_regression_tests` | E | "RULE: add tests/test_regressions.py with one test function per bug you fixed (at least 3)". Vết: tác tử chỉ sửa `inventory/*.py`, không tạo tệp test mới. |
| code-learn | `rule_changelog` | E | "RULE: record each fix in CHANGELOG.md under the heading '## Unreleased' as a bullet '- fix(<function name>): ...'". Vết: tác tử có `read_file workspace/CHANGELOG.md` nhưng không ghi gì vào đó. |
| data-learn | `rule_money_in_cents` | E | "RULE: money values in answer.json are integer cents (1606.67 USD is written 160667)". Mâu thuẫn bề ngoài với đề (`north_q1_revenue` "(number)"): tác tử làm theo đề. |
| data-learn | `rule_meta_block` | E | "RULE: answer.json has an object `meta` = {source, rows_in, rows_used}". |
| data-learn | `rule_clean_csv` | E | "RULE: write workspace/clean.csv with the header order_id,timestamp_utc,region,amount_cents ...". Không có ở đề hay README. |
| logs-learn | `rule_service_names` | E | "RULE: service names ... lower-case with '-' replaced by '_' (payment-service -> payment_service)". Ví dụ trong đề lại dùng `"payment-service"`; tác tử làm theo ví dụ của đề. |
| logs-learn | `rule_sorted_errors` | E | "RULE: `errors` is sorted by service, then by timestamp_utc, ascending." |
| logs-learn | `rule_schema_header` | E | "RULE: the top-level object has \"schema_version\": 2 and \"generated_by\": \"log-triage\"." |

**Bằng chứng phủ định cho nhóm A-D, F** (`python scripts/check_breakdown.py`): baseline đạt **18/18 check kỹ thuật** và **0/9 check quy ước** trên tác vụ học. Không có check kỹ thuật nào thất bại nên không có lỗi nhóm B (test hiển thị pass), C (`parse_price_all_formats`, `other_caller_fixed` đạt: đã sửa hàm dùng chung), D (`duplicate_rows_removed`, `missing_amount_orders`, `timestamps_utc`, `repeat_counts` đạt). Không có nhóm A: vết cho thấy tác tử đọc `workspace/README.md` ở cả 3 tác vụ (code-learn đọc cả `CHANGELOG.md`, cả 4 mô-đun `inventory/*.py` và `tests/test_report.py`). Không có nhóm F: câu trả lời cuối chỉ liệt kê các tệp thực sự đã sửa.

**Nhận xét.** 9/9 lỗi thuộc nhóm E. Nguyên nhân chung: quy ước Acme **không xuất hiện ở bất kỳ đâu** tác tử nhìn thấy (đề chỉ nói "plus whatever the Acme ... conventions require"), nên không có cách nào đạt chúng nếu không có phản hồi. Hai trong số đó còn mâu thuẫn bề ngoài với đề (kiểu "number" so với số nguyên cent; ví dụ `payment-service`). Đây đúng là loại tri thức mà skill có thể cung cấp: curator đọc được `detail` (chính là phát biểu quy tắc), nên về nguyên tắc skill có thể phòng ngừa nhóm E — với điều kiện skill ghi **cụ thể** quy ước (tên tệp, khóa, định dạng), không chỉ khuyên "đọc kỹ quy tắc".

Ghi chú hạ tầng (không tính là lỗi tác tử): lần chạy baseline đầu tiên bị loại vì Git trên Windows (`core.autocrlf=true`) chuyển `tasks/**` sang CRLF; `tests/test_report.py` có SHA-256 khác giá trị `check.py` mong đợi nên `tests_not_modified` luôn thất bại dù tác tử không sửa. Đã đặt `git config --local core.autocrlf false`, checkout lại `tasks/` (hash khớp `79e05f4c…`), và chạy lại toàn bộ; kết quả cũ không dùng.

## 5. Điều kiện `subagents` (Phần 2.3)

- **Các subagent đã định nghĩa** (`src/lab/subagents.py`):
  - `explorer` — chỉ đọc; khảo sát README/CHANGELOG/docstring/test/mẫu dữ liệu và báo cáo quy tắc kèm trích dẫn nguồn. Lý do: lỗi phổ biến của tác tử là bỏ qua đặc tả (nhóm A).
  - `implementer` — thực hiện thay đổi, sửa nguyên nhân gốc, chạy test, liệt kê tệp đã đổi kèm bằng chứng. `description` yêu cầu lời giao việc chứa ĐỦ quy tắc và đường dẫn.
  - `reviewer` — kiểm tra độc lập (mở tệp, chạy lại test, tính lại), không sửa, trả PASS hoặc danh sách lỗi. Lý do: phòng nhóm B và F.
- **`subagent_calls`**: code-learn 4 (explorer, implementer, reviewer, explorer), data-learn 3 (explorer, implementer, reviewer), logs-learn 5 (explorer, general-purpose, implementer ×3). Tác tử chính giao việc ở cả 3 tác vụ, gần như theo đúng chuỗi explorer → implementer → reviewer gợi ý trong `description`.
- **Thông tin thiếu khi giao việc — nguyên nhân chính của điểm thấp:**
  - data-learn (1/8, baseline 5/8): lời giao cho `implementer` là "compute all required metrics for answer.json (plus any Acme reporting convention keys)" — **không chứa tên 5 khóa bắt buộc** (`north_q1_revenue`, ...) hay định nghĩa quý 1. Subagent chỉ thấy lời giao việc nên tự đặt tên khóa; checker đọc `got None` cho 4 khóa. Lời giao cho `reviewer` cũng không kèm đề ("check ... against the user prompt"), nên reviewer trả `PASS`. Tác tử chính tin báo cáo đó và kết thúc — vi phạm "Check what a subagent returns" của `SUBAGENTS_NOTE`. Đây là lỗi nhóm A/F do cô lập ngữ cảnh.
  - code-learn (6/10, baseline 7/10): thêm thất bại `tests_not_modified`: báo cáo cuối ghi "`workspace/tests/test_report.py`: Added comprehensive test cases covering accounting price notation, ...", tức subagent đã sửa tệp test gốc (lẽ ra phải tạo tệp test mới) dù lời giao cho implementer có câu "Do not modify files in tests/". Tác tử chính không kiểm tra lại.
  - logs-learn (6/9, như baseline): lời giao việc lần 2 cho implementer chép đầy đủ quy tắc của đề; kỹ thuật 6/6, quy ước 0/3 như baseline.
- **Token và thời gian**: trung bình 364.653 token/tác vụ so với 109.701 của baseline (**×3,3**); từng tác vụ ×2,3 (code), ×3,8 (data), ×4,6 (logs). Thời gian 174.8–461.8 s so với 12.4–70.6 s. Check kỹ thuật giảm từ 18/18 xuống 13/18. Đa tác tử tốn nhiều hơn và kém hơn trong thí nghiệm này. Lưu ý `tool_calls` của luồng chính nhỏ hơn (6–9) vì công việc nằm bên trong subagent và không hiện trong `trace.md`.

## 6. Self-evolving: skill do curator sinh (Phần 3)

- **Số lần chạy curator: 3** (lần đầu + 2 lần chạy lại, mức tối đa). Bộ skill giữ lại là **đầu ra nguyên văn của lần 1**; không sửa tay (đã đối chiếu `diff -r` với bản sao lưu).
  - Lần 1 → 2 skill hợp lệ: `adhere-to-strict-rule-specifications`, `comprehensive-regression-testing-and-changelogging`. Quyết định xóa `adhere-to-strict-rule-specifications` và chạy lại vì nó quá chung: chỉ nêu ví dụ ("integer cents", "schema versions", "mandatory metadata blocks") mà không ghi giá trị quy ước (header `clean.csv`, khóa `meta`, `schema_version: 2`, `generated_by`, thứ tự sắp xếp) — tác tử đọc nó vẫn không biết phải ghi gì.
  - Lần 2 → skill cho data/logs (`comprehensive-spec-verification`) bị `validate_skill` **từ chối** vì chứa chuỗi `orders` (một marker của tác vụ đánh giá) — bộ chặn rò rỉ hoạt động. Skill còn lại `rigorous-codebase-maintenance` mất chi tiết cụ thể (không còn `tests/test_regressions.py`, định dạng `fix(<function name>)`). Kém hơn lần 1.
  - Lần 3 → `adhere-to-strict-rules-and-schemas` (chung chung như lần 1) và `comprehensive-testing-and-bug-tracking`: bỏ định dạng gạch đầu dòng ("using the required bullet format") và làm yếu quy tắc type hints thành "if requested" — có hại vì quy ước không bao giờ được "request" trong đề. Kém hơn lần 1.
  - Kết luận: chọn lại bộ của lần 1 (cả 2 skill). Việc chọn chỉ dựa trên nội dung skill và `detail` của tác vụ học; không dùng dữ liệu tác vụ đánh giá.

| Skill | Tổng quát hay riêng cho tác vụ học? | Đúng hay sai (nêu chỗ sai nếu có) | Độ dài, `description` và `skills_read` ở Phần 3.4 |
|---|---|---|---|
| `comprehensive-regression-testing-and-changelogging` | Tổng quát cho họ "sửa lỗi codebase": nêu quy ước Acme (type hints mọi hàm public, `tests/test_regressions.py` một test/lỗi, `CHANGELOG.md` `## Unreleased` `- fix(<function name>): ...`) — tên tệp/định dạng là bản thân quy ước nên được phép; không nêu tên hàm hay tệp nguồn của `inventory`. | Đúng với cả 3 `detail` của code-learn. Thiếu "ít nhất 3". Bước 4 "verify that all test files remain intact" hữu ích (phòng `tests_not_modified`). Có một dòng thừa `<body>` (mô hình chép nhãn trong prompt mẫu) — vô hại. | 10 dòng; `description` "Use when fixing bugs or updating codebases that require regression tests and changelog entries" — điều kiện kích hoạt có thể quá hẹp: đề không nói "require regression tests", tác tử phải tự liên hệ. `skills_read`: (điền sau 3.4) |
| `adhere-to-strict-rule-specifications` | Tổng quát (mọi tác vụ có quy tắc định dạng đầu ra); nhắc gián tiếp các quy ước data/logs qua ví dụ. | Không sai, nhưng **không đủ**: không chứa giá trị quy ước nên khó giúp các check `rule_` của data/logs; giá trị chủ yếu là nhắc "viết script kiểm tra mọi ràng buộc". Có dòng `<body>` thừa. | 10 dòng; `description` "Use when completing tasks with specific schema, output naming, formatting, or organizational rules" — rộng, dễ kích hoạt. Phần 3.4: được đọc ở cả 3 tác vụ (vết data-learn/logs-learn chứa nội dung skill) nhưng data-learn 5/8 và logs-learn 6/9 **bằng baseline**: 0/6 check quy ước — skill không cho biết giá trị quy ước, và khi đề nói `north_q1_revenue` là "number" tác tử làm theo đề. |

## 7. Kết quả so sánh (Phần 4.3, 4.4)

```text
(dán bảng ở đây)
```

## 8. Phân tích

(hoàn thiện sau Phần 4)

## 9. Hạn chế và tính hợp lệ

(hoàn thiện sau Phần 4)

## 10. Kết luận

(hoàn thiện sau Phần 4)

## Phụ lục

- Lệnh đã chạy (theo thứ tự, mọi lệnh `python` chạy trong container Docker như mục 1):
  1. `pytest tests/test_01_provided.py` → 12 passed; `pytest` → 29 passed.
  2. `python scripts/tour.py`.
  3. `python -m lab.runner --condition baseline --tasks data-learn` (lần đầu: lỗi 400 thought_signature qua Option 1 → chuyển sang `google_genai`).
  4. Chạy baseline/subagents trên tác vụ học lần 1 → **loại bỏ** do lỗi CRLF (mục 4).
  5. `python -m lab.runner --condition baseline --tasks learn`; `python -m lab.runner --condition subagents --tasks learn`.
  6. `python -m lab.curator` ×3 (mục 6).
  7. `python -m lab.runner --condition skills-auto --tasks learn` lần 1 → **loại bỏ**: lỗi hạ tầng 429 `RESOURCE_EXHAUSTED` (free tier giới hạn 500 request/ngày/mô hình). Chạy lại sau khi quota reset.
- Ghi chú: Gemini free tier giới hạn 500 request/ngày cho `gemini-3.5-flash-lite`; điều kiện `subagents` tiêu thụ nhiều request nhất.
