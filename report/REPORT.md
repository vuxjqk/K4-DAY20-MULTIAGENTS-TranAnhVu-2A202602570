# Báo cáo Lab: Self evolving Agentic

## 1. Thông tin nhóm và cấu hình

| Họ tên | Mã sinh viên | Phần đóng góp |
|---|---|---|
| Trần Anh Vũ | 2A202602570 | Toàn bộ (cá nhân) |

- Mô hình: `LAB_MODEL=google_genai:gemini-3.5-flash-lite` (Google AI Studio, free tier) qua `langchain-google-genai` 4.4.0. `LAB_TEMPERATURE=0` nhưng **không có hiệu lực**: thư viện cảnh báo "Model 'gemini-3.5-flash-lite' uses fixed sampling defaults; the sampling parameter(s) temperature will be ignored" — mô hình lấy mẫu ngẫu nhiên. `recursion_limit` = 60 (mặc định).
- Lý do không dùng Option 1 (cổng tương thích OpenAI của Gemini): Gemini 3.x yêu cầu gửi lại `thought_signature` kèm mỗi function call; `ChatOpenAI` làm rơi trường này nên lần gọi công cụ thứ hai luôn lỗi 400 `Function call is missing a thought_signature`. Tích hợp gốc `google_genai` (Option 2 của `model.py`, không sửa `model.py`) xử lý đúng. Đã thêm `langchain-google-genai>=4.0` vào `pyproject.toml`.
- Deep Agents 0.7.21. Máy chủ Windows 11; mọi lệnh chạy **trong Docker** (`python:3.12-slim`, image build từ `Dockerfile` của kho) vì shell của tác tử cần `/bin/sh`:
  `docker run --rm --env-file .env -e PYTHONDONTWRITEBYTECODE=1 -v <repo>:/lab lab-deepagents python -m lab.runner ...`
- Số lần chạy tác vụ: 21 lần chạy hợp lệ dùng trong báo cáo (chi tiết ở Phụ lục); ngân sách thực tế là giới hạn 500 request/ngày của free tier.
- Commit của tag `freeze`: `daaae35` (commit `hypotheses`: `3983d14`).

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
| `comprehensive-regression-testing-and-changelogging` | Tổng quát cho họ "sửa lỗi codebase": nêu quy ước Acme (type hints mọi hàm public, `tests/test_regressions.py` một test/lỗi, `CHANGELOG.md` `## Unreleased` `- fix(<function name>): ...`) — tên tệp/định dạng là bản thân quy ước nên được phép; không nêu tên hàm hay tệp nguồn của `inventory`. | Đúng với cả 3 `detail` của code-learn. Thiếu "ít nhất 3". Bước 4 "verify that all test files remain intact" hữu ích (phòng `tests_not_modified`). Có một dòng thừa `<body>` (mô hình chép nhãn trong prompt mẫu) — vô hại. | 10 dòng; `description` "Use when fixing bugs or updating codebases that require regression tests and changelog entries" — điều kiện kích hoạt có thể quá hẹp: đề không nói "require regression tests", tác tử phải tự liên hệ. Phần 3.4: được đọc ở cả 3 tác vụ học (`skills_read` = 2 mỗi tác vụ). Trên code-learn được làm theo **một phần**: `rule_regression_tests` và `rule_changelog` đạt (baseline trượt), `rule_type_hints` vẫn trượt → 9/10. |
| `adhere-to-strict-rule-specifications` | Tổng quát (mọi tác vụ có quy tắc định dạng đầu ra); nhắc gián tiếp các quy ước data/logs qua ví dụ. | Không sai, nhưng **không đủ**: không chứa giá trị quy ước nên khó giúp các check `rule_` của data/logs; giá trị chủ yếu là nhắc "viết script kiểm tra mọi ràng buộc". Có dòng `<body>` thừa. | 10 dòng; `description` "Use when completing tasks with specific schema, output naming, formatting, or organizational rules" — rộng, dễ kích hoạt. Phần 3.4: được đọc ở cả 3 tác vụ (vết data-learn/logs-learn chứa nội dung skill) nhưng data-learn 5/8 và logs-learn 6/9 **bằng baseline**: 0/6 check quy ước — skill không cho biết giá trị quy ước, và khi đề nói `north_q1_revenue` là "number" tác tử làm theo đề. |

## 7. Kết quả so sánh (Phần 4.3, 4.4)

`python -m lab.compare > report/table.md`:

```text
| Task | baseline | subagents | skills-auto |
|---|---|---|---|
| code-learn | 7/10 | 6/10 | 9/10 |
| data-learn | 5/8 | 1/8 | 5/8 |
| logs-learn | 6/9 | 6/9 | 6/9 |
| code-eval | 7/11 | 6/11 | 10/11 |
| data-eval | 5/9 | 4/9 | 5/9 |
| logs-eval | 6/10 | 2/10 | 6/10 |
| **Mean score - learning tasks** | 0.66 | 0.46 | 0.73 |
| **Mean score - evaluation tasks** | 0.60 | 0.40 | 0.69 |
| **Mean tokens per run** | 101,374 | 385,426 | 207,210 |
| **Runs that read a skill** | 0/6 | 0/6 | 6/6 |
```

`python scripts/check_breakdown.py`:

```text
condition     role    technical  house rules  mean tokens  read a skill
baseline      eval     18/18         0/12          93,048      0/3
baseline      learn    18/18         0/9          109,701      0/3
subagents     eval     12/18         0/12         406,199      0/3
subagents     learn    13/18         0/9          364,653      0/3
skills-auto   eval     18/18         3/12         184,833      3/3
skills-auto   learn    18/18         2/9          229,588      3/3
```

`python scripts/verify_freeze.py` → `checked 6 runs of skill conditions: OK` (chạy trong container Linux có `git`; xem ghi chú CRLF ở Phụ lục).

Lần chạy có `error` hoặc `skills_modified = true`:
- `skills-auto/code-eval` lần 1: `GraphRecursionError: Recursion limit of 60 reached` sau 193.009 token, điểm 8/11 (trượt `rule_type_hints`, `rule_changelog`, `rule_version_bump`). Theo GUIDE 4.2 đã chạy lại; kết quả trong bảng (10/11) là lần chạy lại. Lần 1 được giữ làm ước lượng nhiễu (mục 8.6), không đưa vào `results/`.
- Không lần chạy chính thức nào có `skills_modified = true`.

## 8. Phân tích

1. **Học và đánh giá.** So với `baseline`, chỉ `skills-auto` cải thiện: tác vụ học 0.66 → 0.73 (+0.07), tác vụ đánh giá 0.60 → 0.69 (+0.09). Toàn bộ mức tăng đến từ **họ `code`** (code-learn +2, code-eval +3 check); data và logs bằng baseline ở cả học lẫn đánh giá. `subagents` kém hơn ở cả hai vai trò (0.46 và 0.40). Không có điều kiện nào cải thiện tác vụ học mà không cải thiện tác vụ đánh giá, nên không thấy dấu hiệu quá khớp theo nghĩa "chỉ tốt trên tập học"; nhưng xem câu 2 về quy ước mới.
2. **Kỹ thuật và quy ước.** Check kỹ thuật: baseline và skills-auto đều đạt 18/18 ở cả hai vai trò — skill không giúp (cũng không hại) nhóm này vì đã ở mức trần. Check quy ước (`rule_`): baseline 0/9 học, 0/12 đánh giá; skills-auto 2/9 và 3/12. Cả 5 check quy ước đạt thêm đều thuộc họ code (`rule_type_hints`, `rule_regression_tests`, `rule_changelog`) — đúng ba quy ước mà skill `comprehensive-regression-testing-and-changelogging` mã hóa. **Quy ước mới** của mỗi tác vụ đánh giá (`rule_version_bump`, `rule_sorted_keys_format`, `rule_source_line`) thất bại ở **cả ba điều kiện (0/9)**: curator chỉ thấy phản hồi của tác vụ học nên không có skill nào chứa chúng, và đề vẫn không nêu nội dung quy ước.
3. **Một check skill giúp đạt, một check skill không giúp.**
   - Giúp: `code-eval / rule_changelog` và `rule_regression_tests` — baseline trượt; ở skills-auto tác tử đọc cả 2 SKILL.md ngay đầu (`skills_read = 2`), rồi tạo `tests/test_regressions.py` và ghi mục `## Unreleased` vào CHANGELOG theo mẫu `- fix(<function name>): ...` có trong skill.
   - Đọc nhưng không làm theo đầy đủ: `code-learn / rule_regression_tests` sau freeze — skill chỉ viết "(e.g., `tests/test_regressions.py`)"; vết cho thấy tác tử giao cho subagent `general-purpose` "Write a comprehensive test file tests/test_additional.py ..." nên tên tệp sai quy ước. Chữ "e.g." biến quy ước bắt buộc thành gợi ý.
   - Skill thiếu nội dung: `data-*/rule_money_in_cents`, `rule_meta_block`, `rule_clean_csv` và các `logs-*/rule_*` — skill `adhere-to-strict-rule-specifications` được đọc ở mọi lần chạy nhưng chỉ chứa ví dụ chung ("integer cents", "schema versions"), không có header `clean.csv`, khóa `meta`, `schema_version: 2`... Thêm vào đó, đề data ghi `north_q1_revenue` "(number)" và ví dụ logs ghi `"payment-service"`; khi skill mơ hồ, tác tử làm theo đề (đúng như hướng dẫn 05 mô tả).
4. **Chi phí.** Token trung bình/lần chạy: baseline 101.374, skills-auto 207.210 (×2,0; do đọc skill, viết test hồi quy, CHANGELOG, script kiểm tra), subagents 385.426 (×3,8). Điểm trung bình trên 6 tác vụ: baseline 0.63, skills-auto 0.71, subagents 0.43; tính theo điểm trên 100k token: baseline **0.62**, skills-auto 0.34, subagents 0.11. Baseline hiệu quả nhất theo điểm/token; skills-auto đổi +0.08 điểm lấy gấp đôi token (và lợi ích chỉ ở họ code). Đa tác tử **không đáng** chi phí ở thí nghiệm này: tốn gấp ~4 lần mà giảm điểm; tệ nhất ở logs-eval (2/10 với 655.311 token): lần giao việc cuối cho `implementer` chỉ ghi "Run python to generate workspace/errors.json cleanly ... and verify its final presence and validity" — không kèm quy tắc nên subagent sinh lại tệp sai cấu trúc (`valid_structure` trượt), và tác tử chính không kiểm tra lại.
5. **Rò rỉ và quá khớp.** Không thấy rò rỉ: `validate_skill` kiểm tra `eval_markers()` cho mọi skill; ở lần chạy curator thứ 2, skill `comprehensive-spec-verification` bị loại vì chứa chuỗi `orders` (marker của data-eval, dù ở đó là từ thông dụng — bộ lọc bảo thủ). Curator chỉ đọc `results/baseline/*-learn` (`role == "learn"`), tác vụ đánh giá chỉ được chạy sau tag `freeze`, và `detail` của tác vụ đánh giá luôn rỗng. Về quá khớp: skill code thực chất là danh sách quy ước của tác vụ học; nó chuyển được sang code-eval **vì** tác vụ đánh giá dùng lại các quy ước đó theo thiết kế, nhưng không giúp gì cho quy ước mới — lợi ích là "nhớ quy ước đã thấy", không phải năng lực tổng quát.
6. **Nhiễu.** Cùng bộ skill trên tác vụ học: Phần 3.4 (`results/skills-auto-dev`) 9/10, 5/8, 6/9 và sau freeze 9/10, 5/8, 6/9 — điểm trùng khớp, nhưng ở code-learn **check thất bại khác nhau** (dev trượt `rule_type_hints`, sau freeze trượt `rule_regression_tests`); token chênh 206k → 253k, 260k → 295k, 127k → 140k (+11–23%). Ngoài ra `skills-auto/code-eval` hai lần chạy cho 8/11 (lỗi recursion) và 10/11. Vậy dao động của một ô có thể tới 2–3 check (≈0.2–0.3 điểm tác vụ), cùng cỡ với hiệu ứng lớn nhất trong bảng (+3 check ở code-eval). Các chênh lệch 0.07–0.09 ở hàng trung bình chỉ dựa trên một lần chạy mỗi ô nên **không đủ để khẳng định**; điều đáng tin hơn là hướng nhất quán: cả 4 lần chạy skills-auto ở họ code (gồm dev và lần lỗi) đều đạt nhiều check quy ước hơn baseline (1–3 so với 0), và subagents không hơn baseline ở tác vụ nào, kém ở 5/6.

**Đối chiếu giả thuyết.** H1 được ủng hộ (subagents 0.40 < 0.60 trên đánh giá, token ×4,4 trên đánh giá). H2 được ủng hộ (cải thiện chỉ ở code-eval: +3 check quy ước; data/logs bằng baseline; kỹ thuật không đổi). H3 đúng một phần: quy ước mới thất bại ở mọi điều kiện (0/9) như dự đoán, nhưng mức tăng trung bình trên đánh giá (+0.09) không nhỏ hơn trên học (+0.07) — vì tác vụ đánh giá dùng lại quy ước cũ, và vì nhiễu (code-learn sau freeze mất 1 check).

## 9. Hạn chế và tính hợp lệ

1. **Cỡ mẫu nhỏ, mỗi ô một lần chạy.** 3 tác vụ mỗi vai trò, 1 lần chạy mỗi cấu hình; mục 8.6 cho thấy dao động 2–3 check giữa các lần chạy, cùng cỡ với hiệu ứng. Kết luận chỉ nên đọc như xu hướng, không phải ước lượng hiệu ứng; cần lặp ≥3 lần (Phần 6e) để có khoảng dao động.
2. **Nhiệt độ không kiểm soát được.** `gemini-3.5-flash-lite` bỏ qua `temperature=0` (lấy mẫu cố định của nhà cung cấp), nên mọi lần chạy đều ngẫu nhiên, kể cả curator — tăng nhiễu ở mọi ô và khiến chất lượng bộ skill (3 lần chạy curator cho 3 bộ khác nhau) phụ thuộc may rủi.
3. **Một mô hình nhỏ, free tier.** Chỉ dùng `flash-lite`; mô hình mạnh hơn có thể giao việc cho subagent tốt hơn (kết luận về đa tác tử có thể không chuyển sang mô hình khác) hoặc tự suy ra quy ước. Giới hạn 500 request/ngày buộc chia thí nghiệm qua nhiều phiên (khác thời điểm, có thể khác tải máy chủ).
4. **Tác vụ do giảng viên thiết kế với quy ước ẩn.** Mọi lỗi baseline là nhóm E — quy ước không có trong đề; nên thí nghiệm chủ yếu đo "skill có truyền lại quy ước đã thấy không", chứ không đo năng lực giải quyết vấn đề. Tác vụ đánh giá dùng lại quy ước của tác vụ học, nên lợi ích chuyển giao ở họ code là do thiết kế.
5. **Lựa chọn bộ skill có yếu tố con người.** Bộ skill đóng băng là lần 1 trong 3 lần chạy curator, được chọn bằng đọc nội dung (chỉ dùng dữ liệu học). Không sửa tay, nhưng việc chọn vẫn là một can thiệp; bộ khác (lần 3, với "if requested") có thể cho kết quả thấp hơn.
6. **Đo đếm chỉ ở luồng chính.** `tool_calls`, `skills_read` không thấy hoạt động bên trong subagent; ở điều kiện `subagents` vết không cho biết subagent đã sửa gì (ví dụ việc sửa `tests/test_report.py` chỉ suy ra từ báo cáo cuối).

## 10. Kết luận

Trên `gemini-3.5-flash-lite`, mọi lỗi của tác tử mặc định là vi phạm quy ước Acme không được nêu trong đề (0/21 check quy ước, 36/36 check kỹ thuật). Skill do curator tự sinh giúp đúng ở nơi nó chứa quy ước cụ thể (họ code: code-eval 7/11 → 10/11) với chi phí gấp đôi token, nhưng không giúp ở data/logs (skill quá chung) và không giúp quy ước mới của tác vụ đánh giá (0/9 ở mọi điều kiện). Đa tác tử tốn gấp ~3,8 lần token và giảm điểm (0.63 → 0.43) vì lời giao việc làm rơi quy tắc của đề. Với một lần chạy mỗi ô và nhiễu 2–3 check, các chênh lệch trung bình chưa đủ để khẳng định. Đề xuất: sửa prompt curator để buộc chép nguyên văn giá trị quy ước từ `detail` (cấm "e.g." cho tên tệp/khóa bắt buộc), và lặp mỗi cấu hình ≥3 lần để đo nhiễu.

## Phụ lục

- Lệnh đã chạy (theo thứ tự; mọi lệnh `python` chạy trong container Docker như mục 1):
  1. `pytest tests/test_01_provided.py` → 12 passed; `pytest` → 29 passed (sau khi cài đặt Phần 1–3).
  2. `python scripts/tour.py`.
  3. `python -m lab.runner --condition baseline --tasks data-learn` (lần đầu qua Option 1: lỗi 400 thought_signature → chuyển sang `google_genai`).
  4. Baseline/subagents tác vụ học lần 1 → **loại bỏ** do lỗi CRLF (mục 4).
  5. `python -m lab.runner --condition baseline --tasks learn`; `python -m lab.runner --condition subagents --tasks learn`.
  6. `python -m lab.curator` ×3 (mục 6); giữ bộ của lần 1.
  7. `python -m lab.runner --condition skills-auto --tasks learn` lần 1 → **loại bỏ**: lỗi hạ tầng 429 `RESOURCE_EXHAUSTED` (free tier 500 request/ngày/mô hình). Chạy lại sau khi quota reset, rồi đổi tên thành `results/skills-auto-dev` (GUIDE 4.2).
  8. Commit `hypotheses` (`3983d14`), commit + tag `freeze` (`daaae35`).
  9. `python -m lab.runner --condition baseline --tasks eval`; `--condition subagents --tasks eval`; `--condition skills-auto --tasks all` (tạm dừng giữa chừng, sau đó chạy tiếp `--tasks code-eval logs-eval logs-learn`; code-eval chạy lại do `GraphRecursionError`).
  10. `python scripts/verify_freeze.py` → OK; `python -m lab.compare > report/table.md`; `python scripts/check_breakdown.py`.
- Số lần chạy tác vụ hợp lệ dùng trong báo cáo: 21 (18 chính thức + 3 của Phần 3.4), thêm 1 lần chạy lỗi recursion dùng làm ước lượng nhiễu. Các lần chạy bị loại do hạ tầng (CRLF, 429, lỗi 400) không được tính.
- Ghi chú CRLF: kho được clone trên Windows với `core.autocrlf=true` nên tệp văn bản trong cây làm việc là CRLF. Ảnh hưởng (1) `tests_not_modified` (so hash tệp test) và (2) `verify_freeze.py` chạy trong Linux thấy `skills/auto/README.md` khác tag. Khắc phục: `git config --local core.autocrlf false`, checkout lại `tasks/` và `skills/auto/README.md`; nội dung trong git không đổi. Nên thêm `.gitattributes` (`* text=auto eol=lf`) vào kho gốc.
- Thử thách mở rộng: không thực hiện.
