# Đánh giá kiến trúc — VectifyAI/PageIndex (Rà soát repo)

> **Đối tượng:** CTO / Tech Lead  
> **Vai trò:** Kiến trúc sư phần mềm cấp Principal & reviewer mã cấp doanh nghiệp  
> **Ghi chú phạm vi:** Đánh giá này dựa trên **bằng chứng trực tiếp từ các file repo có thể truy cập** trong phiên hiện tại.  
> **Khoảng trống quan trọng:** `pageindex/page_index.py` (pipeline PDF lõi) và `pageindex/config.yaml` **không truy xuất được** trong phiên này; mọi kết luận phụ thuộc các file này được đánh dấu **Không rõ/Không tìm thấy** kèm bước kiểm chứng.

---

## 0) Bắt đầu ngay: Entrypoint + Luồng chính end‑to‑end

### Điểm vào (CLI/script)

Repo hiện có một entrypoint chạy end‑to‑end theo kiểu local/self-host:

- **`run_pageindex.py`** — script CLI nhận `--pdf_path` hoặc `--md_path`, cấu hình options và xuất JSON tree ra filesystem (`./results/*.json`).  
  - **Bằng chứng:** `run_pageindex.py::__main__ (line 0)` — argparse, validate input, gọi `page_index_main(...)` (PDF) hoặc `md_to_tree(...)` (Markdown), rồi `json.dump(...)` ra `./results`.

### Luồng chính: PDF → cây PageIndex → JSON artifact

1. User chạy: `python3 run_pageindex.py --pdf_path /path/to/doc.pdf`  
2. Script parse args, build `opt` bằng `config(...)` (SimpleNamespace)  
3. Gọi `page_index_main(pdf_path, opt)` để tạo tree (implementation nằm trong `pageindex/page_index.py`)  
4. Lưu output JSON ra `./results/<pdf_name>_structure.json`

- **Bằng chứng:** `run_pageindex.py::__main__ (line 0)` — nhánh `if args.pdf_path:` tạo `opt = config(...)` và gọi `toc_with_page_number = page_index_main(args.pdf_path, opt)` rồi save JSON.

> **Không rõ/Không tìm thấy:** Nội bộ pipeline PDF (parse/TOC-detect/tree build/summarize) vì **không truy xuất được nội dung `pageindex/page_index.py`** trong phiên này.  
> **Cách kiểm chứng:** Mở file `pageindex/page_index.py` locally và trace từ hàm `page_index_main(...)` (được gọi bởi entrypoint).

### Luồng chính: Markdown → tree → (tuỳ chọn) summaries → JSON artifact

1. User chạy: `python3 run_pageindex.py --md_path /path/to/doc.md`  
2. Script dùng `ConfigLoader` để nạp default config (được mô tả là từ `config.yaml`) và merge với args  
3. Gọi `asyncio.run(md_to_tree(...))` để:
   - Parse headings `#..######` (bỏ qua code blocks ```…```)
   - Cắt text theo vùng heading
   - (optional) thinning dựa trên token count
   - Build tree + node_id
   - (optional) generate summary cho từng node (async gather)
   - (optional) generate doc_description
4. Lưu output JSON ra `./results/<md_name>_structure.json`

- **Bằng chứng:**  
  - `run_pageindex.py::__main__ (line 0)` — nhánh `elif args.md_path:` gọi `ConfigLoader().load(user_opt)` và `asyncio.run(md_to_tree(...))`, sau đó `json.dump(... ensure_ascii=False)` ra `./results`.  
  - `pageindex/page_index_md.py::md_to_tree (line 0)` — read file, extract nodes, optional thinning, build tree, optional summary/doc_description, return dict `{doc_name, (doc_description), structure}`.

---

## 1) 8–12 file quan trọng nhất nên đọc trước (triage list)

1. `run_pageindex.py` — điểm vào CLI & điều phối (nhánh PDF/MD).  
   - **Bằng chứng:** `run_pageindex.py::__main__ (line 0)`
2. `pageindex/__init__.py` — bề mặt API public (re-export).  
   - **Bằng chứng:** `pageindex/__init__.py (line 0)`
3. `pageindex/page_index.py` — **pipeline PDF lõi** (không truy xuất được trong phiên này; cần đọc tại máy).  
   - **Bằng chứng:** `pageindex/__init__.py (line 0)` re-export; `run_pageindex.py (line 0)` gọi `page_index_main(...)`
4. `pageindex/page_index_md.py` — pipeline Markdown (build tree + thinning + summaries).  
   - **Bằng chứng:** `pageindex/page_index_md.py::md_to_tree (line 0)`
5. `pageindex/utils.py` — wrapper LLM, đếm token, trích xuất text PDF, JSON helpers, traversal helpers, config loader (tham chiếu).  
   - **Bằng chứng:** `pageindex/utils.py::ChatGPT_API* (line 0)`
6. `pageindex/config.yaml` — config defaults (không truy xuất được trong phiên này; cần đọc tại máy).  
   - **Bằng chứng:** `run_pageindex.py (line 0)` comment “Load config with defaults from config.yaml”
7. `requirements.txt` — bề mặt dependency & mức độ pin phiên bản.  
   - **Bằng chứng:** `README.md (lines 25–27)` hướng dẫn install từ `requirements.txt`
8. `tests/` — fixtures (PDF/MD) + expected outputs (nếu có) để hiểu schema & determinism.  
   - **Bằng chứng:** `README.md (lines 22–24)` trỏ đến `tests/pdfs` và `tests/results`
9. `cookbook/*.ipynb` — usage patterns, expected workflow.  
   - **Bằng chứng:** `README.md (lines 20–21)` link notebooks
10. `tutorials/` — (khả năng) có demo retrieval/tree search (chưa kiểm chứng trong phiên này).  
    - **Bằng chứng:** Repo structure hiển thị có `tutorials/`
11. `CHANGELOG.md` — lịch sử thay đổi & breaking changes.  
    - **Bằng chứng:** Repo structure hiển thị có `CHANGELOG.md`
12. `LICENSE` — compliance.  
    - **Bằng chứng:** Repo structure hiển thị có `LICENSE` (MIT)

---

## 2) Báo cáo đánh giá kiến trúc

## 2.1 Tóm tắt điều hành (10–15 dòng)

- Repo cung cấp **local/self-host CLI pipeline** để tạo **PageIndex tree** từ **PDF** hoặc **Markdown**, xuất artifact JSON ra filesystem (`./results`).  
  - **Bằng chứng:** `run_pageindex.py::__main__ (line 0)`
- **LLM dependency là trung tâm**: wrapper gọi `openai.OpenAI().chat.completions.create(...)` với `temperature=0`, retry đơn giản và không có schema validation/caching mặc định.  
  - **Bằng chứng:** `pageindex/utils.py::ChatGPT_API (line 0)`
- **Architecture hiện nghiêng về “script + utils module”**: nhiều wildcard import/export (`import *`) làm tăng coupling, khó test, khó kiểm soát API boundary.  
  - **Bằng chứng:** `run_pageindex.py (line 0)`, `pageindex/__init__.py (line 0)`, `pageindex/page_index_md.py (line 0)`
- Nhánh Markdown pipeline có cơ chế **async summary generation** bằng `asyncio.gather` trên toàn bộ node list, nhưng **không có concurrency limit/rate-limit**, tiềm ẩn rủi ro cost/rate-limit.  
  - **Bằng chứng:** `pageindex/page_index_md.py::generate_summaries_for_structure_md (line 0)`
- Secret handling dựa vào `.env` và env var `CHATGPT_API_KEY`; `load_dotenv()` chạy ngay khi import `utils.py` → side effect.  
  - **Bằng chứng:** `README.md (lines 25–27)`, `pageindex/utils.py (line 0)`
- Reliability: wrapper LLM trả về `"Error"` string sau max retries (và có return type không nhất quán giữa hàm), rủi ro “silent failure” lan truyền vào pipeline/output.  
  - **Bằng chứng:** `pageindex/utils.py::ChatGPT_API (line 0)`, `pageindex/utils.py::ChatGPT_API_with_finish_reason (line 0)`
- Observability còn mỏng: script dùng `print(...)`, utils dùng `logging.error(...)` nhưng không thấy cấu hình structured logging/metrics/timing per stage trong phần code truy xuất được.  
  - **Bằng chứng:** `run_pageindex.py (line 0)`, `pageindex/utils.py (line 0)`
- Testing/CI: repo có `tests/` nhưng chưa đánh giá được mức coverage và loại test do hạn chế truy xuất nội dung.  
  - **Bằng chứng:** `README.md (lines 22–24)`
- Điểm cần ưu tiên: (1) làm rõ boundary & public API, (2) harden LLM layer (prompt-injection, caching, output validation, backoff), (3) kiểm soát concurrency/cost, (4) nâng observability & testability.

---

## 2.2 Tổng quan hệ thống (mục tiêu, phạm vi, giả định)

### Mục tiêu hệ thống (theo repo & code truy xuất)

- Generate **tree-structured document index** từ PDF/Markdown, phục vụ reasoning-based retrieval (mô tả ở README).  
  - **Bằng chứng:** `README.md (lines 12–16, 22–29)`, `run_pageindex.py (line 0)`

### Phạm vi đánh giá (đã đọc/đã có bằng chứng trực tiếp)

- Entrypoint & orchestration: `run_pageindex.py`  
- Markdown indexing path: `pageindex/page_index_md.py`  
- LLM wrappers + token utilities + PDF text extraction helpers (một phần): `pageindex/utils.py`  
- Public API surface: `pageindex/__init__.py`  
- Repo usage & options: `README.md`

### Assumptions (được user cung cấp)

- Repo là framework/pipeline xử lý tài liệu để tạo “PageIndex tree” và hỗ trợ reasoning-based retrieval.

### Không rõ/Không tìm thấy (ảnh hưởng đến độ sâu phân tích)

- **Không truy xuất được `pageindex/page_index.py`** (core PDF pipeline) trong phiên này → runtime flow chi tiết cho PDF, prompt templates, tree construction logic, schema validation cho PDF path: **Không rõ**.  
  - **Bằng chứng:** `run_pageindex.py (line 0)` gọi `page_index_main(...)`; `pageindex/__init__.py (line 0)` re-export `.page_index`.
- **Không truy xuất được `pageindex/config.yaml`** → default config keys/values và config schema: **Không rõ**.  
  - **Bằng chứng:** `run_pageindex.py (line 0)` comment “Load config with defaults from config.yaml”.

**Cách kiểm chứng tối thiểu để “đóng khoảng trống”:**

- Mở local repo và đọc:
  - `pageindex/page_index.py` từ hàm `page_index_main(...)` và các prompt gọi LLM
  - `pageindex/config.yaml` + `ConfigLoader` implementation (trong `pageindex/utils.py`)

---

## 2.3 Sơ đồ kiến trúc (C4 Container/Component)

### C4 — Góc nhìn Container (text)

- **Container A: Tiến trình PageIndex CLI (Python)**  
  - Chạy local, đọc input PDF/MD từ filesystem.  
  - Gọi **OpenAI Chat Completions API** qua `openai` SDK cho TOC detection / summarization / description (ít nhất đối với MD summaries, chắc chắn; với PDF thì entrypoint gọi `page_index_main` nhưng nội bộ không rõ).  
  - Ghi JSON artifacts ra filesystem (`./results`).  
  - **Bằng chứng:** `run_pageindex.py (line 0)`, `pageindex/utils.py::ChatGPT_API (line 0)`

- **Hệ thống ngoài: OpenAI API**  
  - Được gọi qua `openai.OpenAI(...).chat.completions.create(...)`.  
  - **Bằng chứng:** `pageindex/utils.py::ChatGPT_API (line 0)`

- **Kho dữ liệu: Filesystem local**  
  - Input: PDF/MD path  
  - Output: `./results/<name>_structure.json`  
  - **Bằng chứng:** `run_pageindex.py (line 0)`

### Mermaid — Góc nhìn Container

```mermaid
flowchart LR
  User["User"] --> CLI["PageIndex CLI - run_pageindex.py"]
  CLI --> FS["Local FS"]
  CLI --> OpenAI["OpenAI Chat Completions API"]
  FS --> CLI
  CLI --> Out["./results/NAME_structure.json"]
```

### C4 — Góc nhìn Component (trong tiến trình CLI)

**Các component có bằng chứng trực tiếp:**

1. **CLI Orchestrator**: parse args, validation, rẽ nhánh PDF/MD  
   - **Bằng chứng:** `run_pageindex.py::__main__ (line 0)`
2. **Markdown Tree Builder**: parse heading, build tree, thinning, sinh summary  
   - **Bằng chứng:** `pageindex/page_index_md.py::md_to_tree (line 0)`
3. **LLM Client Wrapper**: gọi OpenAI chat sync & async + retry  
   - **Bằng chứng:** `pageindex/utils.py::ChatGPT_API`, `ChatGPT_API_async`, `ChatGPT_API_with_finish_reason (line 0)`
4. **Token Utilities**: đếm token qua `tiktoken.encoding_for_model`  
   - **Bằng chứng:** `pageindex/utils.py::count_tokens (line 0)`
5. **PDF Text Extraction Helpers** (đã thấy): `PyPDF2.PdfReader`, `page.extract_text()`  
   - **Bằng chứng:** `pageindex/utils.py::extract_text_from_pdf`, `get_text_of_pages (line 0)`
6. **Tree Traversal Helpers**: `structure_to_list`, `write_node_id`, leaf utilities  
   - **Bằng chứng:** `pageindex/utils.py::structure_to_list`, `write_node_id`, `get_leaf_nodes (line 0)`

**Các component được tham chiếu nhưng chưa xác minh được implementation trong phiên này:**

- **PDF Index Builder**: `page_index_main(...)`  
  - **Bằng chứng:** `run_pageindex.py (line 0)`, `pageindex/__init__.py (line 0)`  
  - **Không rõ:** chi tiết implementation
- **Config Loader**: `ConfigLoader.load(...)` + `config.yaml`  
  - **Bằng chứng:** `run_pageindex.py (line 0)`  
  - **Không rõ:** chi tiết implementation

---

## 2.4 Luồng chính (build index; retrieval/query; error flows)

### 2.4.1 Luồng tạo chỉ mục — PDF

**Luồng (đã biết):**

1. Validate PDF path + extension  
2. Build options `opt = config(...)`  
3. Call `page_index_main(pdf_path, opt)`  
4. Save JSON to `./results/<pdf>_structure.json`

- **Bằng chứng:** `run_pageindex.py::__main__ (line 0)` nhánh PDF.

**Không rõ (do thiếu code `pageindex/page_index.py`):**

- Parsing/segmentation strategy theo page/section
- TOC detection logic (nếu có)
- LLM prompt templates & output validation
- Node splitting logic theo `--max-pages-per-node`, `--max-tokens-per-node`
- Caching/checkpoint

**Cách kiểm chứng:** đọc `pageindex/page_index.py` các hàm được gọi (tối thiểu: `page_index_main`).

### 2.4.2 Luồng tạo chỉ mục — Markdown

**Luồng (đã biết & có code):**

1. Read markdown file (`utf-8`)  
2. Extract headings, bỏ qua fenced code blocks  
3. Build node list: `{title, level, line_num, text}`  
4. Optional thinning: compute token count cho node + descendants; merge nodes dưới ngưỡng  
5. Build tree (stack-based) và assign `node_id`  
6. Optional: generate summaries (async gather)  
   - leaf: `summary`  
   - non-leaf: `prefix_summary`  
7. Optional: remove `text` field nếu không yêu cầu  
8. Optional: generate `doc_description`  
9. Return `{doc_name, (doc_description), structure}`

- **Bằng chứng:**  
  - `pageindex/page_index_md.py::extract_nodes_from_markdown (line 0)` — regex headers + toggle code block  
  - `pageindex/page_index_md.py::extract_node_text_content (line 0)` — slice markdown_lines theo line_num  
  - `pageindex/page_index_md.py::tree_thinning_for_index (line 0)` — merge children nếu token < threshold  
  - `pageindex/page_index_md.py::build_tree_from_nodes (line 0)` — stack-based build  
  - `pageindex/page_index_md.py::generate_summaries_for_structure_md (line 0)` — `asyncio.gather` summaries  
  - `pageindex/page_index_md.py::md_to_tree (line 0)` — orchestration + format_structure + return schema

### 2.4.3 Luồng truy vấn / tìm kiếm

- **Không tìm thấy / Không rõ.**  
  Trong phần code truy xuất được (CLI + MD pipeline + utils), **không thấy** API/query path để “tree search” hoặc assemble context trả lời query.
  - **Bằng chứng:** `README.md (lines 12–16)` mô tả retrieval ở mức concept; không thấy code tương ứng trong các file đã truy xuất.

**Cách kiểm chứng:** scan `tutorials/` và `pageindex/page_index.py` để tìm các entrypoints liên quan query/search (keywords: `search`, `retrieve`, `tree_search`, `query`, `doc_search`).

### 2.4.4 Luồng lỗi (những gì thấy trực tiếp)

1. **Input validation errors** → raise `ValueError` (fail fast)  
   - **Bằng chứng:** `run_pageindex.py::__main__ (line 0)` — validate pdf/md, file exists, mutually exclusive args
2. **OpenAI API failures** → retry 10 lần, mỗi lần sleep 1s; sau đó trả `"Error"` (string) thay vì raise  
   - **Bằng chứng:** `pageindex/utils.py::ChatGPT_API (line 0)`, `ChatGPT_API_async (line 0)`
3. **LLM output quá dài** (`finish_reason == "length"`) → `ChatGPT_API_with_finish_reason` trả `(content, "max_output_reached")`  
   - **Bằng chứng:** `pageindex/utils.py::ChatGPT_API_with_finish_reason (line 0)`

---

## 2.5 Bản đồ module & phụ thuộc (bảng)

| Module/Path | Vai trò | Phụ thuộc vào (inbound) | Phụ thuộc ra (outbound) | Ký hiệu chính (bằng chứng) |
|---|---|---|---|---|
| `run_pageindex.py` | Điều phối CLI, validate input, chạy pipeline PDF/MD, ghi output JSON | user CLI | `pageindex` (`import *`), `pageindex.page_index_md.md_to_tree`, `pageindex.utils.ConfigLoader`, `asyncio`, `json`, FS | `__main__ (line 0)` |
| `pageindex/__init__.py` | Bề mặt API public: re-export | `run_pageindex.py` | `.page_index`, `.page_index_md` | `from .page_index import *` / `md_to_tree (line 0)` |
| `pageindex/page_index_md.py` | Markdown → nodes → tree; optional thinning; optional summaries/doc_description | `run_pageindex.py` | `.utils` qua `import *`, `asyncio`, `re`, `os` | `md_to_tree`, `build_tree_from_nodes`, `tree_thinning_for_index`, `generate_summaries_for_structure_md (line 0)` |
| `pageindex/utils.py` | Wrapper LLM (OpenAI), token counting (tiktoken), PDF text extraction (PyPDF2), JSON extraction, tree traversal helpers | `page_index_md.py`, `run_pageindex.py` | `openai`, `tiktoken`, `dotenv`, `PyPDF2`, `yaml`, `asyncio`, v.v. | `ChatGPT_API`, `ChatGPT_API_async`, `count_tokens`, `extract_text_from_pdf`, `structure_to_list`, `write_node_id (line 0)` |
| `pageindex/page_index.py` | PDF indexing core | `run_pageindex.py` (calls) + `__init__.py` re-export | Không rõ | `page_index_main` referenced (Không rõ impl) |
| `pageindex/config.yaml` | Default config values | `ConfigLoader.load` (referenced) | N/A | Chỉ được tham chiếu (Không rõ nội dung) |
| `tests/` | Fixtures + expected outputs | N/A | N/A | Referenced in README |
| `cookbook/`, `tutorials/` | Ví dụ & hướng dẫn | N/A | N/A | Repo structure |

---

## 2.6 Đánh giá thuộc tính chất lượng

> Ghi chú: Với PDF pipeline (`pageindex/page_index.py`) và `config.yaml`, nhiều chi tiết sẽ ghi **Không rõ** do không truy xuất được file trong phiên này.

### 2.6.1 Hiệu năng

**Hiện trạng (dựa trên bằng chứng):**

- Markdown summary generation tạo **task cho mọi node** và chạy `asyncio.gather` không giới hạn → nguy cơ bắn quá nhiều request song song (rate-limit, latency spikes).  
  - **Bằng chứng:** `pageindex/page_index_md.py::generate_summaries_for_structure_md (line 0)` — `tasks = [...]` + `await asyncio.gather(*tasks)`
- Thinning và token count có pattern “scan forward for children” cho từng node → worst-case **O(n²)** theo số heading nodes.  
  - **Bằng chứng:** `pageindex/page_index_md.py::update_node_list_with_text_token_count.find_all_children` + vòng lặp over nodes (line 0)
- Token counting dùng `tiktoken.encoding_for_model(model)` và encode toàn bộ text → cost theo O(token_count).  
  - **Bằng chứng:** `pageindex/utils.py::count_tokens (line 0)`

**Rủi ro:**

- Với Markdown dài (n nodes lớn), concurrency + O(n²) thinning có thể gây chậm/timeout hoặc vượt rate limit OpenAI.
- Với PDF (Không rõ) nhưng khả năng tương tự nếu summary cho nhiều node.

**Đề xuất (actionable):**

1. **Giới hạn concurrency cho async LLM calls** bằng `asyncio.Semaphore` + batch scheduling.  
   - Target: `pageindex/page_index_md.py::generate_summaries_for_structure_md`  
   - Effort: **S** | Risk: Low | Impact: High  
   - Bằng chứng: `asyncio.gather` không giới hạn (line 0)
2. **Thêm caching** theo hash(prompt+model+version) (filesystem cache) để tránh gọi lại khi rerun.  
   - Target: `pageindex/utils.py::ChatGPT_API*` và callers  
   - Effort: **M** | Risk: Medium | Impact: High  
   - Bằng chứng: không có caching trong wrappers (line 0)
3. **Tối ưu thinning**: thay scan-forward nhiều lần bằng precomputed parent/child relations (stack pass) để về O(n).  
   - Target: `pageindex/page_index_md.py::update_node_list_with_text_token_count`, `tree_thinning_for_index`  
   - Effort: **M** | Risk: Medium | Impact: Medium/High  
   - Bằng chứng: find_all_children scan forward per node (line 0)

### 2.6.2 Độ tin cậy

**Hiện trạng:**

- Retry LLM: `max_retries=10`, sleep 1s, không exponential backoff, không phân loại lỗi (429/5xx), và **không raise** sau khi fail mà trả `"Error"`.  
  - **Bằng chứng:** `pageindex/utils.py::ChatGPT_API (line 0)`, `ChatGPT_API_async (line 0)`
- `ChatGPT_API_with_finish_reason` có **return type không nhất quán**: thường trả tuple `(content, status)` nhưng error path trả `"Error"` string.  
  - **Bằng chứng:** `pageindex/utils.py::ChatGPT_API_with_finish_reason (line 0)`
- CLI fail fast cho input sai (điểm tốt).  
  - **Bằng chứng:** `run_pageindex.py (line 0)`

**Rủi ro:**

- Silent failure: “Error” string bị đẩy vào pipeline downstream → JSON output sai/khó debug.
- Không có checkpoint/resume → fail giữa chừng có thể tốn chi phí rerun.

**Đề xuất:**

1. Chuẩn hoá error handling: raise exception typed (`LLMCallError`) thay vì `"Error"`, và thống nhất return type.  
   - Target: `pageindex/utils.py::ChatGPT_API*`  
   - Effort: **S** | Risk: Medium (breaking callers) | Impact: High  
   - Bằng chứng: returns `"Error"` (line 0)
2. Exponential backoff + jitter + xử lý riêng 429/5xx + timeout.  
   - Target: `pageindex/utils.py::ChatGPT_API*`  
   - Effort: **M** | Risk: Low/Med | Impact: High  
   - Bằng chứng: sleep cố định 1s (line 0)
3. Add checkpointing: persist intermediate artifacts per stage (TOC detect, node split, summary) → resume.  
   - Target: `run_pageindex.py` orchestration + PDF/MD pipelines  
   - Effort: **M/L** | Risk: Medium | Impact: High  
   - Bằng chứng: hiện chỉ output cuối cùng (line 0)

### 2.6.3 Bảo mật

**Hiện trạng:**

- Secrets: dùng `.env` và env var `CHATGPT_API_KEY`; `load_dotenv()` chạy khi import `utils.py`.  
  - **Bằng chứng:** `README.md (lines 25–27)`, `pageindex/utils.py (line 0)`
- Prompt injection hardening: wrapper gửi messages **chỉ có role `"user"`**, không có `"system"` guardrails.  
  - **Bằng chứng:** `pageindex/utils.py::ChatGPT_API (line 0)` — `messages = [{"role": "user", "content": prompt}]`
- Không thấy redaction PII hay policy logging.

**Rủi ro:**

- **Prompt injection**: nội dung PDF/MD có thể chứa chỉ dẫn phá prompt hoặc exfiltrate data, do không có system message và không có “content delimitation / instruction hierarchy”.
- Secret hygiene: `.env` dễ bị commit nhầm (không đánh giá được `.gitignore` trong phiên này → Không rõ).

**Đề xuất:**

1. Thêm system message bắt buộc + “treat document text as untrusted data” + enforce JSON schema output.  
   - Target: `pageindex/utils.py::ChatGPT_API*`  
   - Effort: **S** | Risk: Medium (prompt changes) | Impact: High  
   - Bằng chứng: chỉ user role (line 0)
2. Thêm output validation (JSON schema / Pydantic) + reject/repair với bounded retries.  
   - Target: callers trong `pageindex/page_index_md.py` và PDF pipeline  
   - Effort: **M** | Risk: Medium | Impact: High  
   - Bằng chứng: JSON helpers hiện chỉ “clean whitespace” và parse (line 0)
3. Secret scanning & guard: document rõ env var, hỗ trợ `OPENAI_API_KEY` alias, validate presence at startup.  
   - Target: `run_pageindex.py` + `pageindex/utils.py`  
   - Effort: **S** | Risk: Low | Impact: Medium  
   - Bằng chứng: `CHATGPT_API_KEY = os.getenv("CHATGPT_API_KEY")` (line 0)

### 2.6.4 Khả năng bảo trì

**Hiện trạng:**

- Wildcard imports/exports lan rộng:  
  - `run_pageindex.py`: `from pageindex import *`  
  - `pageindex/__init__.py`: `from .page_index import *`  
  - `pageindex/page_index_md.py`: `from .utils import *`  
  - **Bằng chứng:** `run_pageindex.py (line 0)`, `pageindex/__init__.py (line 0)`, `pageindex/page_index_md.py (line 0)`
- `utils.py` có dấu hiệu “God module”: LLM calls + token + PDF parsing + JSON extraction + tree traversal.  
  - **Bằng chứng:** `pageindex/utils.py (line 0)` — nhiều nhóm hàm & dependencies
- Config object dùng `SimpleNamespace as config` → không có type safety/schema.  
  - **Bằng chứng:** `pageindex/utils.py (line 0)`, `run_pageindex.py (line 0)`

**Rủi ro:**

- Coupling cao, khó refactor, khó static analysis, dễ name collision.
- Testability thấp do tạo OpenAI client trực tiếp trong hàm.

**Đề xuất:**

1. Loại bỏ wildcard imports; định nghĩa public API rõ ràng (`pageindex/api.py`), explicit imports.  
   - Target: `run_pageindex.py`, `pageindex/__init__.py`, `pageindex/page_index_md.py`  
   - Effort: **S/M** | Risk: Medium | Impact: High
2. Tách `utils.py` thành các module: `llm_client.py`, `pdf_extract.py`, `tree_utils.py`, `json_utils.py`, `config.py`.  
   - Effort: **M/L** | Risk: Medium/High | Impact: High
3. Introduce typed config (Pydantic dataclass) thay SimpleNamespace.  
   - Effort: **M** | Risk: Medium | Impact: High

### 2.6.5 Nhắc về kiểm thử

**Hiện trạng:**  

- Repo có `tests/pdfs` và `tests/results` được README nhắc tới, nhưng chưa xác minh có unit/integration tests runnable.  
  - **Bằng chứng:** `README.md (lines 22–24)`

**Không rõ:** test framework (pytest?), CI pipeline, determinism baseline outputs.

**Đề xuất:**

- Tạo integration tests chạy offline bằng mock LLM (record/replay), verify schema JSON & stable node_id ordering.  
  - Target: `tests/` + new `llm_client` abstraction  
  - Effort: **M** | Risk: Medium | Impact: High  
  - **Bằng chứng:** LLM calls hard-coded trong `pageindex/utils.py` (line 0)

### 2.6.6 Quan sát/giám sát

**Hiện trạng:**

- Script dùng `print(...)`; utils dùng `logging.error(...)` nhưng không thấy structured logs/metrics.  
  - **Bằng chứng:** `run_pageindex.py (line 0)`, `pageindex/utils.py (line 0)`
- Không thấy stage timing; dù có import `time`.  
  - **Bằng chứng:** `pageindex/utils.py (line 0)` — `import time`

**Đề xuất:**

- Structured logging (JSON) + correlation id per run + timing per stage (pdf_extract, toc_detect, node_split, summary).  
  - Effort: **S/M** | Impact: Medium/High

### 2.6.7 Tính di động

**Hiện trạng:**

- Install qua `requirements.txt`; chạy local; phụ thuộc OpenAI API key.  
  - **Bằng chứng:** `README.md (lines 25–27)`, `run_pageindex.py (line 0)`
- Dùng PyPDF2 và PyMuPDF (import `pymupdf`) → có thể yêu cầu binary wheels phù hợp OS.  
  - **Bằng chứng:** `pageindex/utils.py (line 0)`

**Không rõ:** version pinning chi tiết trong requirements (chưa review được nội dung đầy đủ).

---

## 2.7 Ghi chú bảo mật & tuân thủ

- License: MIT (repo listing).  
  - **Bằng chứng:** file `LICENSE` (repo listing)
- Secret management: `.env` + `CHATGPT_API_KEY`.  
  - **Bằng chứng:** `README.md (lines 25–27)`, `pageindex/utils.py (line 0)`
- Prompt injection: hiện chưa có system prompt; risk cao khi ingest untrusted document.  
  - **Bằng chứng:** `pageindex/utils.py::ChatGPT_API (line 0)`
- Data handling/PII: không thấy cơ chế redaction trước khi gửi lên LLM; cần policy theo domain (finance/legal).  
  - **Bằng chứng:** trong phần code truy xuất được không có logic redaction

---

## 2.8 Đánh giá kiểm thử & quan sát

### Testing

- **Không rõ** về độ phủ và loại test (chưa đọc `tests/` nội dung).  
  - **Bằng chứng:** `README.md (lines 22–24)`

### Observability

- Logging rời rạc (`print` + `logging.error`), thiếu metrics/traces.  
  - **Bằng chứng:** `run_pageindex.py (line 0)`, `pageindex/utils.py (line 0)`

---

## 2.9 Rủi ro hàng đầu

| Rủi ro | Mức độ | Bằng chứng | Giảm thiểu |
|---|---|---|---|
| Prompt injection do không có system prompt; doc content có thể “override” hướng dẫn | Cao | `pageindex/utils.py::ChatGPT_API (line 0)` chỉ user role | Thêm system message + delimiting + JSON schema validation |
| Cost/rate-limit spike do `asyncio.gather` không giới hạn cho summaries | Cao | `pageindex/page_index_md.py::generate_summaries_for_structure_md (line 0)` | Semaphore limit + batching + caching |
| Silent failure: LLM wrapper trả `"Error"` thay vì raise; return type không nhất quán | Cao | `pageindex/utils.py::ChatGPT_API (line 0)`, `ChatGPT_API_with_finish_reason (line 0)` | Raise typed exceptions + consistent return types + fail fast |
| Coupling cao do wildcard imports/exports; khó test/refactor | Trung bình/Cao | `run_pageindex.py (line 0)`, `pageindex/__init__.py (line 0)`, `pageindex/page_index_md.py (line 0)` | Explicit imports + stable public API module |
| Global side effects: `load_dotenv()` tại import time | Trung bình | `pageindex/utils.py (line 0)` | Move env loading to entrypoint; validate config explicitly |
| Thinning O(n²) theo số headings (scan-forward per node) | Trung bình | `pageindex/page_index_md.py::update_node_list_with_text_token_count (line 0)` | Refactor to single-pass stack relations |
| PDF extraction chất lượng phụ thuộc PyPDF2 `extract_text()` (known limitations) | Trung bình | `pageindex/utils.py::extract_text_from_pdf (line 0)` | Add extractor strategy + fallback (PyMuPDF/vision) |
| Thiếu checkpoint/resume; fail mid-run phải rerun tốn token | Trung bình | `run_pageindex.py (line 0)` chỉ save cuối | Stage outputs + resume + cache |
| Schema output không versioned/validated | Trung bình | `pageindex/page_index_md.py::md_to_tree (line 0)` trả dict tự do | Define schema_version + Pydantic validation |
| Config defaults phụ thuộc `config.yaml` nhưng file không load được trong phiên này | Thấp/Trung bình | `run_pageindex.py (line 0)` nhắc config.yaml + `ConfigLoader` | Document schema + validate + surface config in CLI help |

---

## 2.10 Khuyến nghị & Lộ trình (Ngắn/Trung/Dài)

### Tác vụ nhanh (≤ 1 tuần)

1. **Harden LLM wrappers (fail fast + consistent types)**  
   - Mục tiêu: tránh silent failure, dễ debug  
   - Thay đổi: `pageindex/utils.py::ChatGPT_API`, `ChatGPT_API_async`, `ChatGPT_API_with_finish_reason`  
   - Effort: **S** | Risk: Medium (caller changes) | Impact: High  
   - **Bằng chứng:** wrappers trả `"Error"` / inconsistent return (line 0)

2. **Thêm system prompt + instruction hierarchy + JSON schema expectation**  
   - Mục tiêu: giảm prompt injection, tăng output validity  
   - Thay đổi: `pageindex/utils.py::ChatGPT_API*` (prepend system message; wrap doc text in delimiters)  
   - Effort: **S** | Risk: Medium | Impact: High  
   - **Bằng chứng:** messages chỉ user role (line 0)

3. **Giới hạn concurrency cho async summaries**  
   - Mục tiêu: tránh rate limit/cost spike  
   - Thay đổi: `pageindex/page_index_md.py::generate_summaries_for_structure_md`  
   - Effort: **S** | Risk: Low | Impact: High  
   - **Bằng chứng:** `asyncio.gather(*tasks)` không limit (line 0)

4. **Bỏ wildcard imports ở entrypoint và MD module**  
   - Mục tiêu: giảm coupling, dễ static analysis  
   - Thay đổi: `run_pageindex.py`, `pageindex/page_index_md.py`, `pageindex/__init__.py`  
   - Effort: **S** | Risk: Medium | Impact: Medium/High  
   - **Bằng chứng:** `import *` (line 0)

5. **Startup validation** (API key + config)  
   - Mục tiêu: fail fast rõ ràng  
   - Thay đổi: `run_pageindex.py` trước khi chạy pipeline; `utils.py` không load dotenv ở import time  
   - Effort: **S** | Risk: Low | Impact: Medium  
   - **Bằng chứng:** `load_dotenv()` hiện chạy khi import (line 0)

### Trung hạn (2–6 tuần)

1. **Refactor kiến trúc module (tách utils “God module”)**  
   - Mục tiêu: maintainability, testability, clear boundaries  
   - Thay đổi: split `pageindex/utils.py` thành modules (`llm_client`, `pdf_extract`, `tree_utils`, `json_utils`, `config`)  
   - Effort: **M/L** | Risk: Medium/High | Impact: High  
   - **Bằng chứng:** utils chứa nhiều concerns (line 0)

2. **Introduce typed config + schema_version cho output**  
   - Mục tiêu: ổn định API, giảm drift, validate output  
   - Thay đổi: config model (Pydantic/dataclass), update `run_pageindex.py` và pipelines trả output có `schema_version`  
   - Effort: **M** | Risk: Medium | Impact: High  
   - **Bằng chứng:** `SimpleNamespace as config` + output dict tự do (utils line 0; `md_to_tree` line 0)

3. **Caching & checkpoint**  
   - Mục tiêu: kiểm soát chi phí, resume cho tài liệu dài  
   - Thay đổi: llm_client caching; stage artifacts to `./results/<doc>/stage_*.json`  
   - Effort: **M** | Risk: Medium | Impact: High  
   - **Bằng chứng:** hiện chỉ output cuối (run_pageindex line 0)

4. **Test harness with mock LLM (record/replay)**  
   - Mục tiêu: determinism, CI reliable  
   - Thay đổi: abstract LLM client; add integration tests verifying schema & stability  
   - Effort: **M** | Risk: Medium | Impact: High  
   - **Bằng chứng:** LLM calls trực tiếp trong utils (line 0)

### Dài hạn (6–12 tuần)

1. **Define Domain vs Infrastructure boundary (clean architecture)**  
   - Mục tiêu: mở rộng (PDF extractor variants, vision-based pipeline), maintain long-term  
   - Thay đổi: domain objects (`Node`, `Tree`, `Document`), ports/adapters cho LLM & extractors  
   - Effort: **L** | Risk: High | Impact: High  
   - **Bằng chứng:** hiện là script-centric + wildcard coupling (multiple files line 0)

2. **Operationalization**: structured logs/metrics/traces + profiling + cost governance  
   - Mục tiêu: enterprise readiness  
   - Thay đổi: logging framework, metrics per stage, token/cost accounting, rate-limit governance  
   - Effort: **M/L** | Risk: Medium | Impact: High  
   - **Bằng chứng:** thiếu observability (run_pageindex line 0; utils line 0)

3. **Retrieval package (nếu repo muốn cover end-to-end)**  
   - Mục tiêu: hiện thực hóa “tree search / reasoning retrieval” trong OSS (nếu chưa có)  
   - Thay đổi: thêm module `retrieval/` + interfaces + examples  
   - Effort: **L** | Risk: Medium/High | Impact: High  
   - **Bằng chứng:** README mô tả retrieval nhưng chưa thấy trong code đã truy xuất (README lines 12–16)

---

## 2.11 Phụ lục: Danh sách bằng chứng theo file (đã dùng)

1. `run_pageindex.py` — `__main__` CLI orchestration (line 0)  
2. `pageindex/__init__.py` — re-export public API (line 0)  
3. `pageindex/page_index_md.py` — `md_to_tree`, thinning, summaries (line 0)  
4. `pageindex/utils.py` — `count_tokens`, `ChatGPT_API*`, PDF extraction helpers, JSON helpers (line 0–1)  
5. `README.md` — usage, deployment options, required `.env` key, references to tests/results (lines 8–31)

---

## Ghi chú về giới hạn “line numbers”

Một số file Python trong phiên này được hiển thị theo kiểu “gộp” (ví dụ: line 0 rất dài). Đây là giới hạn cách nội dung được cung cấp trong môi trường review này. Vì vậy, citations dùng `line 0`/`line 1` là theo **điểm bắt đầu của file** trong dữ liệu hiện có, kèm theo **symbol/function** để định danh chính xác.
