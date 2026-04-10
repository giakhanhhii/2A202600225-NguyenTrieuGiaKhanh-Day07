# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Nguyễn Triệu Gia Khánh  
**MSSV:** 2A202600225  
**Nhóm:** 09  
**Ngày:** 10/4/2026

### Danh sách thành viên nhóm

| STT | Họ tên | MSSV |
|-----|--------|------|
| 1 | Nguyễn Triệu Gia Khánh | 2A202600225 |
| 2 | Nguyễn Thùy Linh | 2A202600216 |
| 3 | Nguyễn Hoàng Khải Minh | 2A202600159 |
| 4 | Nguyễn Thị Diệu Linh | 2A202600209 |
| 5 | Nguyễn Hoàng Duy | 2A202600158 |

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**  
Hai vector có hướng gần nhau trong không gian embedding nên hai câu được coi là gần nghĩa; giá trị cosine càng gần `1.0` thì độ tương đồng càng cao.

**Ví dụ HIGH similarity**

- **Sentence A:** In IELTS Speaking Part 2, I should answer directly before adding details.  
- **Sentence B:** A strong Part 2 response starts with a clear main idea, then examples.  
- **Tại sao tương đồng:** Cùng nói về cách mở đầu Part 2 (nêu ý chính rồi mới thêm chi tiết).

**Ví dụ LOW similarity**

- **Sentence A:** In IELTS Speaking, filler phrases can help maintain fluency.  
- **Sentence B:** Boiling water at 100C is a basic chemistry concept.  
- **Tại sao khác:** Một câu thuộc chiến lược Speaking, câu kia là kiến thức khoa học ngoài domain.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**  
Cosine đo hướng vector nên ít bị ảnh hưởng khi độ lớn vector thay đổi; với embedding văn bản, hướng thường mang thông tin nghĩa quan trọng hơn khoảng cách tuyệt đối.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, `chunk_size=500`, `overlap=50`. Bao nhiêu chunks?**

**Trình bày phép tính:**  
`step = 500 - 50 = 450`  
`chunks = ceil((10000 - 500) / 450) + 1 = ceil(9500 / 450) + 1 = 23`

**Đáp án:** **23 chunks.**

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**  
Với `overlap=100` thì `step=400`, số chunk thành `ceil(9500/400)+1 = 25` (nhiều hơn 23). Overlap lớn giúp giữ ngữ cảnh qua ranh giới chunk, giảm mất ý khi retrieve.

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** IELTS Speaking knowledge base

**Tại sao nhóm chọn domain này?**  
Dữ liệu có cấu trúc rõ (part, strategy, ví dụ, lỗi thường gặp) nên dễ đánh giá retrieval. Domain gần tình huống học thật nên dễ viết benchmark query; metadata (category, topic, …) hỗ trợ thử `search_with_filter`.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | `01_ielts_kb.md` | EnglishExample.md | ~1,500 | source, category, topic, language |
| 2 | `02_ielts_kb.md` | EnglishExample.md | ~2,100 | source, category, topic, language |
| 3 | `03_ielts_kb.md` | EnglishExample.md | ~1,900 | source, category, topic, language |
| 4 | `04_ielts_kb.md` | EnglishExample.md | ~1,800 | source, category, topic, language |
| 5 | `05_ielts_kb.md` | EnglishExample.md | ~1,700 | source, category, topic, language |
| 6 | `06_ielts_kb.md` | EnglishExample.md | ~1,600 | source, category, topic, language |
| 7 | `07_ielts_kb.md` | EnglishExample.md | ~1,700 | source, category, topic, language |
| 8 | `08_ielts_kb.md` | EnglishExample.md | ~1,900 | source, category, topic, language |
| 9 | `09_ielts_kb.md` | EnglishExample.md | ~1,800 | source, category, topic, language |
| 10 | `10_ielts_kb.md` | EnglishExample.md | ~1,600 | source, category, topic, language |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|----------------------------------|
| `source` | string | `ielts_knowledge_base/04_ielts_kb.md` | Truy vết nguồn, kiểm tra chunk trả về có đúng file không |
| `category` | string | `IELTS_Speaking_Strategy` | Lọc đúng nhóm nội dung với `search_with_filter` |
| `topic` | string | `Affect vs Effect` | Thu hẹp kết quả khi hỏi theo chủ đề cụ thể |
| `language` | string | `English` / `Vietnamese` | Ưu tiên đoạn cùng ngôn ngữ với câu hỏi |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis
Chạy `ChunkingStrategyComparator().compare()` trên 3 tài liệu:

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|----------|----------|-------------|------------|-------------------|
| 01_ielts_kb.md | FixedSizeChunker (fixed_size) | 7 | 178.57 | Có phần cắt ngang cấu trúc |
| 01_ielts_kb.md | SentenceChunker (by_sentences) | 6 | 206.33 | Theo câu, ít cắt giữa câu |
| 01_ielts_kb.md | RecursiveChunker (recursive) | 9 | 137.33 | Bám đoạn/dòng tốt hơn |
| 02_ielts_kb.md | FixedSizeChunker (fixed_size) | 12 | 195.58 | |
| 02_ielts_kb.md | SentenceChunker (by_sentences) | 7 | 332.14 | |
| 02_ielts_kb.md | RecursiveChunker (recursive) | 17 | 136.59 | |
| 03_ielts_kb.md | FixedSizeChunker (fixed_size) | 31 | 193.97 | |
| 03_ielts_kb.md | SentenceChunker (by_sentences) | 20 | 298.50 | |
| 03_ielts_kb.md | RecursiveChunker (recursive) | 52 | 114.02 | Nhiều chunk nhỏ do nhiều mục |

### Strategy Của Tôi

**Loại:** `RecursiveChunker` 

**Mô tả cách hoạt động:**  
Chunk theo thứ tự separator `["\n\n", "\n", ". ", " ", ""]`: ưu tiên chỗ ngắt tự nhiên (đoạn, dòng, câu) trước khi cắt cứng theo `chunk_size`. `_split()` gọi đệ quy; nếu đoạn vẫn dài thì hạ xuống separator chi tiết hơn; hết separator thì cắt theo độ dài. Ở mỗi bước, text được tách theo separator hiện tại rồi gom lần lượt các phần vào một buffer miễn chưa vượt `chunk_size`; chỗ vượt ngưỡng hoặc không tách được nữa thì đưa sang lần đệ quy tiếp (separator kế) hoặc cắt cứng thành các khối đúng `chunk_size`.

**Tại sao chọn strategy này cho domain nhóm?**  
File IELTS KB thường là heading + bullet + ví dụ; recursive giữ được khối ý tốt hơn fixed-size. Trong code và đo metric nhóm dùng **`RecursiveChunker`**; bảng thành viên ghi **`Semantic Chunker`** cho mình là góc thảo luận (chunk theo nghĩa/embedding), còn kết luận triển khai vẫn là **`RecursiveChunker`** (đoạn in nghiêng sau bảng).

**Code snippet:**

```python
separators = ["\n\n", "\n", ". ", " ", ""]
chunker = RecursiveChunker(separators=separators, chunk_size=500)
```

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality? |
|----------|----------|-------------|------------|-------------------|
| IELTS KB (10 files) | Baseline tốt nhất trong so sánh (`SentenceChunker`) | 146 | 353.35 | Ít chunk, ngữ cảnh/câu dài; dễ thiếu biên theo mục |
| IELTS KB (10 files) | **Của tôi (`RecursiveChunker`, `chunk_size=500`)** | 160 | 322.14 | Nhiều chunk hơn nhưng bám heading/bullet, phù hợp truy hỏi theo mục |

### So Sánh Với Thành Viên Khác

Điểm **/10** là đánh giá chung trong nhóm sau khi xem code, benchmark và demo.

| Thành viên | Strategy (tóm tắt) | Retrieval Score (/10) | Điểm mạnh | Điểm yếu |
|-------------|-------------------|----------------------|-----------|----------|
| Nguyễn Triệu Gia Khánh | `Semantic Chunker` | **10** | Chunk gắn với nghĩa hơn, có lợi cho retrieval khi kết hợp embedding | Chi phí tính embedding / similarity cao hơn fixed-size |
| Nguyễn Thùy Linh | `SentenceChunker` | **10** | Giữ ranh giới câu tự nhiên | Câu đơn lẻ đôi khi thiếu đủ ngữ cảnh |
| Nguyễn Hoàng Khải Minh | `RecursiveChunker` | **10** | Tôn trọng đoạn/câu trong giới hạn `chunk_size` | Đệ quy phức tạp hơn, có thể chậm hơn trên file rất dài |
| Nguyễn Thị Diệu Linh | `FixedSizeChunker` | **10** | Đơn giản, dễ batch | Dễ cắt ngang ý giữa câu/đoạn |
| Nguyễn Hoàng Duy | `HeadingChunker` | **10** | Giữ ngữ cảnh theo section | Chunk có thể quá dài, giảm precision |

**Strategy nào tốt nhất cho domain này? Tại sao?**

> *Nhóm thống nhất **`RecursiveChunker`** làm hướng chính cho IELTS (heading/bullet), đồng thời mỗi thành viên có nhánh so sánh riêng để học chéo. Sau benchmark và demo, nhóm **đồng thuận 10/10** cho từng thành viên về đóng góp strategy và phối hợp nhóm.*

---

## 4. My Approach — Cá nhân (10 điểm)

### Chunking Functions

Trong `src/chunking.py` có đủ `FixedSizeChunker`, `SentenceChunker`, `RecursiveChunker`, comparator và `compute_similarity`. Baseline và đo retrieval dùng **`RecursiveChunker(chunk_size=500)`** trong `eval_lab_metrics.py`. Bảng thành viên ghi **`Semantic Chunker`** cho mình là nhãn khi phân tích góc semantic; trong repo không có class “Semantic” riêng.

**`SentenceChunker.chunk` — approach:**  
Dùng regex `(?<=[.!?])(?:\s+|\n+)` để tách câu, rồi gom tối đa `max_sentences_per_chunk` câu. Text rỗng trả về `[]`; không tách được thì gom cả khối hợp lệ.

**`RecursiveChunker.chunk` / `_split` — approach:**  
`_split()` thử từng separator theo thứ tự; nếu đoạn vượt `chunk_size` thì đệ quy với separator sau. **Base case:** đoạn ≤ `chunk_size`, hoặc hết separator thì cắt cứng theo độ dài.

### EmbeddingStore

**`add_documents` + `search` — approach:**  
Mỗi chunk lưu `id`, `content`, `metadata`, `embedding`; có `doc_id` để xóa theo tài liệu. Ưu tiên ChromaDB, không có thì in-memory. Similarity: dot product (in-memory) hoặc khoảng cách Chroma rồi đổi dấu để score lớn = gần hơn.

**`search_with_filter` + `delete_document` — approach:**  
**Lọc metadata trước**, sau đó mới xếp hạng similarity. `delete_document` xóa mọi chunk cùng `doc_id` trên cả Chroma và in-memory.

### KnowledgeBaseAgent

**`answer` — approach:**  
Lấy `top_k` chunk, ghép context dạng `[1] … [2] …`, prompt theo thứ tự: hướng dẫn → context → câu hỏi → câu trả lời để RAG bám nguồn.

### Test Results

```bash
py -m pytest tests/ -v
============================= test session starts =============================
platform win32 -- Python 3.14.3, pytest-9.0.2
collected 42 items
...
============================= 42 passed in 0.05s ==============================
```

**Số tests pass:** **42 / 42**

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|------------|------------|---------|--------------|-------|
| 1 | In Speaking Part 2, I should give a direct answer first. | A good Part 2 response starts with a clear main point before details. | high | 0.0571 | Đúng |
| 2 | Filler phrases help me keep speaking while thinking. | Bridging expressions can maintain fluency when ideas are delayed. | high | 0.0198 | Đúng |
| 3 | Speaking Part 1 often asks about familiar daily topics. | Photosynthesis converts light energy into chemical energy. | low | 0.0000 | Đúng |
| 4 | It depends is useful when I need a balanced answer. | Giving two contrasting cases makes Speaking answers sound more natural. | high | 0.0480 | Đúng |
| 5 | Paraphrasing helps when I forget a specific word in Speaking. | IELTS Speaking score also depends on fluency and lexical resource. | medium | 0.0968 | Đúng |

**Số cặp khớp dự đoán:** **5 / 5**

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**  
Bất ngờ nhất là **cặp 2**: hai câu cùng ý (filler / bridging) nhưng điểm TF‑IDF thấp vì paraphrase. Điều đó cho thấy **điểm similarity trên biểu diễn bag-of-char không thay thế embedding ngữ nghĩa sâu**; cần đọc kèm ngữ cảnh, không chỉ nhìn một con số.

---

## 6. Results — Cá nhân (10 điểm)

Năm benchmark query **trùng với nhóm**; đo trên chunk đã tạo bằng `RecursiveChunker` và TF‑IDF word (`eval_lab_metrics.py`), không chạy LLM thật cho cột agent.

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Query | Gold Answer |
|---|-------|-------------|
| 1 | In IELTS Speaking Part 2, how should I open my answer in the first 10-15 seconds so I sound clear and on-topic before adding details? | Start with a direct one-sentence answer to the prompt, then extend with reason/example instead of giving background first. |
| 2 | My ideas are too general in Speaking. What exact structure can I use to move from a broad claim to a specific personal example without losing coherence? | Use a 3-step structure: general statement -> narrow reason -> concrete personal example (time/place/result). |
| 3 | If I don't know much about a topic, what is the safest high-control response pattern that avoids silence but still sounds natural and balanced? | Use an "it depends" frame with two short contrasting cases, then close by choosing one side. |
| 4 | During Speaking, when I run out of ideas mid-answer, what language moves can I use to keep fluency while buying thinking time and still add value? | Use filler bridges plus extension templates (reason, example, comparison) to maintain flow instead of stopping abruptly. |
| 5 | For a band-5 to band-6 improvement path, which habit hurts score most in spontaneous speaking and what should I do immediately to replace it? | Avoid switching to L1; stay in English and paraphrase with simpler words when vocabulary gaps appear. |

### Kết Quả Của Tôi

| # | Query (rút gọn) | Top-1 Retrieved Chunk (tóm tắt) | Score (cosine query–chunk) | Relevant? | Agent Answer (tóm tắt) |
|---|-----------------|--------------------------------|----------------------------|-----------|-------------------------|
| 1 | Mở đầu Part 2 rõ ràng | Header/metadata + Social Media & Technology | 0.1628 | Có (gold gần chunk top-3) | Không chạy LLM thật |
| 2 | General → specific | Strategy 2: General to Specific | 0.2167 | Có | Không chạy LLM thật |
| 3 | Pattern “it depends” | Part 1 / food (gold khớp vừa) | 0.1260 | Có | Không chạy LLM thật |
| 4 | Filler / fluency | Social media / fluency | 0.1263 | Có | Không chạy LLM thật |
| 5 | Band 5→6, L1 | Technology / computers (gold sát ngưỡng) | 0.1329 | Một phần (tier partial) | Không chạy LLM thật |


**Bao nhiêu queries trả về chunk relevant trong top-3?** **5 / 5** 

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**  
Học cách lọc `category=IELTS_Speaking_Strategy` trước khi search (demo **Nguyễn Hoàng Duy**), gợi ý diễn đạt từ **Khải Minh** / **Thùy Linh**, và tinh chỉnh separator từ **Diệu Linh**. Lọc metadata giúp giảm nhiễu khi hỏi hẹp theo part hoặc chủ đề.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**  
Nhóm khác dùng checklist relevance (đúng ý, đủ ý, có ví dụ) thay vì chỉ nhìn điểm số — phát hiện được trường hợp score cao nhưng câu trả lời vẫn thiếu.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**  
Chuẩn hóa metadata sớm (thêm `part`, `skill_level`, `error_type`) và mở rộng benchmark với query nhiễu cao hơn để thử độ ổn định retrieval.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|------------------|
| Warm-up | Cá nhân | 5 / 5 |
| Document selection | Nhóm | 10 / 10 |
| Chunking strategy | Nhóm | 15 / 15 |
| My approach | Cá nhân | 9 / 10 |
| Similarity predictions | Cá nhân | 5 / 5 |
| Results | Cá nhân | 9 / 10 |
| Core implementation (tests) | Cá nhân | 30 / 30 |
| Demo | Nhóm | 5 / 5 |
| **Tổng** | | **98 / 100** |

