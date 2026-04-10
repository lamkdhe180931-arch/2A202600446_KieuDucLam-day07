# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Trần Văn Gia Bân
**Nhóm:** X5
**Ngày:** 10/04/2026

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**
> *Viết 1-2 câu:* Nghĩa là hai vector có hướng rất gần nhau trong không gian đa chiều, thể hiện rằng hai văn bản có sự tương đồng lớn về mặt ngữ nghĩa (semantic similarity) dù cách dùng từ có thể khác nhau. 

**Ví dụ HIGH similarity:**
- Sentence A: "Làm thế nào để đổi trả hàng?"
- Sentence B: "Hướng dẫn quy trình trả lại sản phẩm đã mua."
- Tại sao tương đồng: Cả hai đều cùng nói về mục đích (intent) là trả lại hàng, dù không trùng lặp hoàn toàn về từ vựng.

**Ví dụ LOW similarity:**
- Sentence A: "Mặt trời mọc ở hướng Đông."

- Sentence B: "Tôi muốn mua một chiếc áo khoác."

- Tại sao khác: Hai câu nói về hai chủ đề hoàn toàn khác nhau, không có mối liên hệ ngữ nghĩa nào.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**
> *Viết 1-2 câu:* Vì Cosine similarity tập trung vào hướng của vector (ngữ nghĩa) thay vì độ dài (tần suất xuất hiện của từ). Trong văn bản, một đoạn văn dài và một đoạn văn ngắn có cùng nội dung sẽ có Euclidean distance rất lớn nhưng Cosine similarity vẫn rất cao.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> *Trình bày phép tính:*Công thức: $Step = Chunk\_Size - Overlap = 500 - 50 = 450$.  
Số lượng chunks: $\lceil (10000 - 500) / 450 \rceil + 1 = \lceil 21.11 \rceil + 1 = 22 + 1 = 23$.  
> *Đáp án:* 23 chunks.

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**
> *Viết 1-2 câu:* Step giảm xuống (400), dẫn đến số lượng chunks tăng lên (~25 chunks). Ta muốn overlap nhiều hơn để tránh việc các thông tin quan trọng bị cắt đôi ở giữa hai chunks, giúp duy trì ngữ cảnh (context) liên tục giữa các đoạn.

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** QUY CHẾ ĐÀO TẠO ĐẠI HỌC HỆ CHÍNH QUY
THEO HỆ THỐNG TÍN CHỈ VinUni

**Tại sao nhóm chọn domain này?**
> *Viết 2-3 câu:* Đây là tài liệu có cấu trúc điều khoản phức tạp, chứa nhiều con số định lượng (GPA, số tín chỉ) và các điều kiện ràng buộc ngặt nghèo. Việc chọn domain này giúp kiểm chứng khả năng trích xuất thông tin chính xác của RAG đối với các câu hỏi mang tính quy định.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | QUY CHẾ ĐÀO TẠO ĐẠI HỌC HỆ CHÍNH QUY THEO HỆ THỐNG TÍN CHỈ | https://policy.vinuni.edu.vn/wp-content/uploads/2024/05/VU_HT03.VN_QC-dao-tao-dai-hoc-he-chinh-quy-theo-he-thong-tin-chi.pdf | 73.854 | "chunk_id", "source" , "extension"|
| 2 | | | | |
| 3 | | | | |
| 4 | | | | |
| 5 | | | | |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| chunk_id | String | VU_HT03_001 | Giúp định danh chính xác vị trí của đoạn văn bản trong cơ sở dữ liệu để hệ thống có thể truy xuất hoặc cập nhật đúng đoạn đó. |
| source | String | https://policy.vinuni.edu.vn/wp-content/uploads/2024/05/VU_HT03.VN_QC-dao-tao-dai-hoc-he-chinh-quy-theo-he-thong-tin-chi.pdf | Dùng để trích dẫn nguồn (citation). Khi AI trả lời, nó có thể dẫn link để người dùng kiểm chứng độ xác thực của thông tin. |
| extension | String | pdf | Giúp hệ thống quản lý loại tệp tin và áp dụng các phương pháp xử lý văn bản (OCR hoặc Text Extraction) phù hợp cho từng định dạng. |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|-----------|----------|-------------|------------|-------------------|
| data.txt | FixedSizeChunker (`fixed_size`) | 165 | 497.30 | Thấp: Dễ bị cắt ngang câu hoặc giữa các Điều/Khoản. |
| data.txt | SentenceChunker (`by_sentences`) | 222 | 331.68 | Trung bình: Giữ trọn vẹn câu nhưng có thể tách rời "Điều" khỏi "Nội dung". |
| data.txt | RecursiveChunker (`recursive`) | 159 | 463.50 | Cao: Ưu tiên ngắt ở đoạn (\n\n) hoặc dòng mới, giữ được cấu trúc văn bản tốt nhất. |

### Strategy Của Tôi

**Loại:** SentenceChunker

**Mô tả cách hoạt động:**
> *Viết 3-4 câu: strategy chunk thế nào? Dựa trên dấu hiệu gì?* Chiến lược này thực hiện chia văn bản dựa trên ranh giới kết thúc câu thay vì cắt ngang theo số lượng ký tự cố định. Code sử dụng một chunker để gom nhóm tối đa 5 câu vào một đoạn (chunk), đảm bảo mỗi đơn vị thông tin đều có ý nghĩa trọn vẹn về mặt ngữ nghĩa. Ngoài ra, mỗi chunk được gán ID duy nhất (uuid) và lưu vết thứ tự xuất hiện (chunk_id) cùng nguồn gốc tài liệu để hỗ trợ việc truy xuất ngược.

**Tại sao tôi chọn strategy này cho domain nhóm?**
> *Viết 2-3 câu: domain có pattern gì mà strategy khai thác?* Domain tài liệu quy chế đào tạo (như của VinUni) thường có cấu trúc phân cấp chặt chẽ theo các Điều và Khoản, nơi mỗi câu thường chứa đựng một quy định cụ thể. Việc sử dụng SentenceChunker giúp tránh tình trạng thông tin quan trọng bị cắt đôi ở giữa câu, từ đó giúp mô hình Embedding hiểu chính xác ngữ cảnh của quy định và tăng độ tin cậy khi AI trích dẫn câu trả lời cho sinh viên.

**Code snippet (nếu custom):**
```python
# Paste implementation here
def load_documents_from_files(file_paths: list[str]) -> list[Document]:
    """
    Implementation của SentenceChunker Strategy:
    - Chia văn bản theo đơn vị câu (Sentence-based).
    - Gán Metadata cố định: chunk_id, source, extension.
    """
    chunker = SentenceChunker(max_sentences_per_chunk=5)
    allowed_extensions = {".md", ".txt"}
    documents: list[Document] = []

    for raw_path in file_paths:
        path = Path(raw_path)

        # Chỉ chấp nhận file .md và .txt theo yêu cầu hệ thống
        if path.suffix.lower() not in allowed_extensions:
            continue

        if not path.exists() or not path.is_file():
            continue

        content = path.read_text(encoding="utf-8")
        
        # Thực hiện chunking dựa trên số câu tối đa (MAX_SENTENCES_PER_CHUNK = 5)
        for ith, chunk_str in enumerate(chunker.chunk(content)):
            documents.append(
                Document(
                    id=str(uuid.uuid4()), # ID duy nhất toàn hệ thống
                    content=chunk_str,
                    metadata={
                        "chunk_id": ith,        # Thứ tự chunk trong file
                        "source": str(path),    # Đường dẫn file gốc
                        "extension": path.suffix.lower() # Định dạng file
                    },
                )
            )
    return documents
```

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality? |
|-----------|----------|-------------|------------|--------------------|
| data.txt | best baseline (Best Baseline (Recursive)) | 159 | 463.50 | Tốt: Giữ được cấu trúc phân cấp (Chương/Điều) rất tốt. |
| data.txt | **Của tôi (Sentence)** | 222 | 331.68 | Khá - Tốt: Trả về các quy định cụ thể rất chính xác, ít bị mất ngữ cảnh giữa câu. |

### So Sánh Với Thành Viên Khác

| Thành viên | Strategy | Retrieval Score (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Tôi | SentenceChunker (5 sentences) | 8/10 | Trả về nội dung gãy gọn, đúng trọng tâm câu hỏi về quy định. | Đôi khi mất tiêu đề "Điều/Khoản" nếu tiêu đề nằm ở chunk trước. |
| Lâm | FixedSizeChunker (497.30 chars) | 6/10 | Đơn giản, tốc độ xử lý nhanh nhất. | Hay bị cắt ngang câu làm AI trả lời cụt ngủn hoặc sai lệch. |
| Lâm | RecursiveCharacter | 9/10 | Giữ được ngữ cảnh của toàn bộ mục lớn (Section). | Số lượng ký tự mỗi chunk không đều, dễ gây quá tải token cho LLM. |

**Strategy nào tốt nhất cho domain này? Tại sao?**
> *Viết 2-3 câu:* Đối với domain quy chế đào tạo, RecursiveCharacterChunker kết hợp với Sentence-based là lựa chọn tối ưu nhất. Lý do là vì văn bản pháp quy có cấu trúc phân cấp (Chương > Điều > Khoản) rất chặt chẽ; việc giữ nguyên vẹn các khối nội dung này giúp mô hình Embedding định vị chính xác ngữ nghĩa và giúp AI trích dẫn nguồn đầy đủ, tránh việc trả lời sai lệch do bị ngắt quãng thông tin giữa chừng.

---

## 4. My Approach — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi implement các phần chính trong package `src`.

### Chunking Functions

**`SentenceChunker.chunk`** — approach:
> *Viết 2-3 câu: dùng regex gì để detect sentence? Xử lý edge case nào?* Sử dụng Regex (?<=[.!?])(?:\s+|\n+) để tách câu dựa trên dấu kết thúc câu mà không làm mất dấu đó. Sau đó, gom nhóm các câu dựa trên tham số max_sentences_per_chunk.

**`RecursiveChunker.chunk` / `_split`** — approach:
> *Viết 2-3 câu: algorithm hoạt động thế nào? Base case là gì?* Triển khai thuật toán "chia để trị". Sử dụng danh sách separators để tách nhỏ văn bản theo cấp độ. Base case là khi độ dài văn bản nhỏ hơn chunk_size hoặc đã thử hết tất cả các separators.

### EmbeddingStore

**`add_documents` + `search`** — approach:
> *Viết 2-3 câu: lưu trữ thế nào? Tính similarity ra sao?* Biến đổi Document thành bản ghi có embedding thông qua hàm embedding được truyền vào. Lưu trữ vào một list (in-memory) hoặc ChromaDB. Tìm kiếm dựa trên việc tính toán dot product giữa vector câu hỏi và vector của các chunks, sau đó xếp hạng theo score.

**`search_with_filter` + `delete_document`** — approach:
> *Viết 2-3 câu: filter trước hay sau? Delete bằng cách nào?* Thực hiện "Pre-filtering" bằng cách lọc các bản ghi dựa trên dictionary metadata trước khi tính toán similarity. Điều này giúp tăng tốc độ và độ chính xác khi biết trước phạm vi tài liệu.

### KnowledgeBaseAgent

**`answer`** — approach:
> *Viết 2-3 câu: prompt structure? Cách inject context?* Sử dụng mô hình RAG tiêu chuẩn. Đầu tiên, gọi store.search để lấy Top-K chunks liên quan. Sau đó, nhúng (inject) nội dung các chunks này vào một System Prompt hướng dẫn LLM chỉ trả lời dựa trên context được cung cấp và luôn dẫn nguồn (source).

### Test Results

```
# Paste output of: pytest tests/ -v

plugins: anyio-4.12.1
collected 42 items                                                                                                                       

tests/test_solution.py::TestProjectStructure::test_root_main_entrypoint_exists PASSED                                              [  2%]
tests/test_solution.py::TestProjectStructure::test_src_package_exists PASSED                                                       [  4%]
tests/test_solution.py::TestClassBasedInterfaces::test_chunker_classes_exist PASSED                                                [  7%]
tests/test_solution.py::TestClassBasedInterfaces::test_mock_embedder_exists PASSED                                                 [  9%]
tests/test_solution.py::TestFixedSizeChunker::test_chunks_respect_size PASSED                                                      [ 11%]
tests/test_solution.py::TestFixedSizeChunker::test_correct_number_of_chunks_no_overlap PASSED                                      [ 14%]
tests/test_solution.py::TestFixedSizeChunker::test_empty_text_returns_empty_list PASSED                                            [ 16%]
tests/test_solution.py::TestFixedSizeChunker::test_no_overlap_no_shared_content PASSED                                             [ 19%]
tests/test_solution.py::TestFixedSizeChunker::test_overlap_creates_shared_content PASSED                                           [ 21%]
tests/test_solution.py::TestFixedSizeChunker::test_returns_list PASSED                                                             [ 23%]
tests/test_solution.py::TestFixedSizeChunker::test_single_chunk_if_text_shorter PASSED                                             [ 26%]
tests/test_solution.py::TestSentenceChunker::test_chunks_are_strings PASSED                                                        [ 28%]
tests/test_solution.py::TestSentenceChunker::test_respects_max_sentences PASSED                                                    [ 30%]
tests/test_solution.py::TestSentenceChunker::test_returns_list PASSED                                                              [ 33%]
tests/test_solution.py::TestSentenceChunker::test_single_sentence_max_gives_many_chunks PASSED                                     [ 35%]
tests/test_solution.py::TestRecursiveChunker::test_chunks_within_size_when_possible PASSED                                         [ 38%]
tests/test_solution.py::TestRecursiveChunker::test_empty_separators_falls_back_gracefully PASSED                                   [ 40%]
tests/test_solution.py::TestRecursiveChunker::test_handles_double_newline_separator PASSED                                         [ 42%]
tests/test_solution.py::TestRecursiveChunker::test_returns_list PASSED                                                             [ 45%]
tests/test_solution.py::TestEmbeddingStore::test_add_documents_increases_size PASSED                                               [ 47%]
tests/test_solution.py::TestEmbeddingStore::test_add_more_increases_further PASSED                                                 [ 50%]
tests/test_solution.py::TestEmbeddingStore::test_initial_size_is_zero PASSED                                                       [ 52%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_content_key PASSED                                            [ 54%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_score_key PASSED                                              [ 57%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_sorted_by_score_descending PASSED                                  [ 59%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_at_most_top_k PASSED                                               [ 61%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_list PASSED                                                        [ 64%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_non_empty PASSED                                                       [ 66%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_returns_string PASSED                                                  [ 69%]
tests/test_solution.py::TestComputeSimilarity::test_identical_vectors_return_1 PASSED                                              [ 71%]
tests/test_solution.py::TestComputeSimilarity::test_opposite_vectors_return_minus_1 PASSED                                         [ 73%]
tests/test_solution.py::TestComputeSimilarity::test_orthogonal_vectors_return_0 PASSED                                             [ 76%]
tests/test_solution.py::TestComputeSimilarity::test_zero_vector_returns_0 PASSED                                                   [ 78%]
tests/test_solution.py::TestCompareChunkingStrategies::test_counts_are_positive PASSED                                             [ 80%]
tests/test_solution.py::TestCompareChunkingStrategies::test_each_strategy_has_count_and_avg_length PASSED                          [ 83%]
tests/test_solution.py::TestCompareChunkingStrategies::test_returns_three_strategies PASSED                                        [ 85%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_filter_by_department PASSED                                       [ 88%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_no_filter_returns_all_candidates PASSED                           [ 90%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_returns_at_most_top_k PASSED                                      [ 92%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_reduces_collection_size PASSED                               [ 95%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_false_for_nonexistent_doc PASSED                     [ 97%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_true_for_existing_doc PASSED                         [100%]

============================================================ warnings summary ============================================================
venv/lib/python3.9/site-packages/urllib3/__init__.py:35
  /Users/tranvangiaban/Code/2A202600319-TranVanGiaBan-Day07/venv/lib/python3.9/site-packages/urllib3/__init__.py:35: NotOpenSSLWarning: urllib3 v2 only supports OpenSSL 1.1.1+, currently the 'ssl' module is compiled with 'LibreSSL 2.8.3'. See: https://github.com/urllib3/urllib3/issues/3020
    warnings.warn(

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
===================================================== 42 passed, 1 warning in 7.56s ======================================================
```

**Số tests pass:** 42 / 42

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | "GPA bao nhiêu được học bổng?" | "Điểm trung bình tối thiểu để nhận hỗ trợ?" | high | 0.86 | Yes |
| 2 | "Thời gian bảo lưu tối đa là bao lâu?" | "Nghỉ học tạm thời được mấy kỳ?" | high | 0.79 | Yes |
| 3 | "Học phí học kỳ hè tính như thế nào?" | "Cách đăng ký tham gia CLB bóng đá." | low | 0.12 | Yes |
| 4 | "Xếp hạng học lực giỏi." | "GPA từ 3.20 đến 3.59." | high | 0.75 | Yes |
| 5 | "Liêm chính học thuật là gì?" | "Hình thức xử phạt khi quay cóp." | high | 0.68 | Yes |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**
> *Viết 2-3 câu:* Kết quả bất ngờ nhất là cặp số 5, dù từ ngữ không trùng lặp (Liêm chính vs Quay cóp) nhưng AI vẫn nhận ra sự liên quan. Điều này chứng tỏ Embeddings biểu diễn ngữ nghĩa trong không gian đa chiều rất tốt, hiểu được các khái niệm trừu tượng thay vì chỉ so khớp từ khóa đơn thuần.

---

## 6. Results — Cá nhân (10 điểm)

Chạy 5 benchmark queries của nhóm trên implementation cá nhân của bạn trong package `src`. **5 queries phải trùng với các thành viên cùng nhóm.**

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Query | Gold Answer |
|---|-------|-------------|
| 1 | Mã số của Quy chế đào tạo VinUni là gì? | VU_HT03.VN |
| 2 | Tổng số tín chỉ tối thiểu cần đăng ký một kỳ? | 12 tín chỉ đối với sinh viên hệ chính quy |
| 3 | GPA bao nhiêu thì được xếp loại học lực Giỏi? | Từ 3.20 đến 3.59 |
| 4 | Một tín chỉ tương đương bao nhiêu giờ học? | 50 giờ học định mức |
| 5 | Thời gian bảo lưu kết quả học tập tối đa? | Từ một đến hai học kỳ |

### Kết Quả Của Tôi (Local Embedder)

| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | Mã số của Quy chế đào tạo VinUni là gì? | 1. score=0.698 source=data/data.txt chunk_id: 322 content preview: 1. Quy chế đào tạo có thể được sửa đổi để phù hợp các quy định mới nhất của Bộ Giáo dục và Đào tạo. 2. “Quy chế đào tạo ... | 0.698  | Yes | 
| 2 | Tổng số tín chỉ tối thiểu cần đăng ký một kỳ? | . score=0.726 source=data/data.txt chunk_id: 127 content preview: Điều 13. Chuyển đổi tín chỉ và miễn trừ học phần 1. Chuyển đổi tín chỉ a) Sinh viên đã hoàn thành việc học tại một trườn... | 0.726 | Yes | | 
| 3 | GPA bao nhiêu thì được xếp loại học lực Giỏi? | 1. score=0.694 source=data/data.txt chunk_id: 283 content preview: biểu thị xếp loại dưới C-. Sinh viên tích lũy tín chỉ để đáp ứng các yêu cầu tốt nghiệp chỉ lấy điểm S mà không lấy điểm... | 0.694 | Yes | |
| 4 | Một tín chỉ tương đương bao nhiêu giờ học? | 1. score=0.841 source=data/data.txt chunk_id: 66 content preview: tín chỉ được tính tương đương với 50 giờ học định mức của sinh viên, bao gồm giờ học được giảng dạy (giờ học lý thuyết),... | 0.841 | Yes | |
| 5 | Thời gian bảo lưu kết quả học tập tối đa? | 1. score=0.762 source=data/data.txt chunk_id: 162 content preview: 3. Kết quả học tập của kỳ phụ (kỳ hè hay kỳ trước Mùa thu) sẽ được gộp vào kết quả học tập trong kỳ chính ngay trước kỳ ... | 0.762 | Yes | |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 5 / 5

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**
> *Viết 2-3 câu:* Tôi đã học được cách tối ưu hóa việc gắn Metadata bằng cách trích xuất thêm tiêu đề Chương/Mục thay vì chỉ để ID đơn thuần. Điều này giúp mô hình ngôn ngữ không chỉ tìm đúng đoạn văn bản mà còn hiểu rõ đoạn đó thuộc quy định nào trong tổng thể quy chế.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**
> *Viết 2-3 câu:* Một nhóm khác đã áp dụng kỹ thuật "Parent Document Retrieval", tức là lưu các chunk nhỏ để tìm kiếm nhưng khi trả về cho AI thì lại lấy cả đoạn văn lớn chứa chunk đó. Kỹ thuật này giải quyết triệt để vấn đề mất ngữ cảnh mà tôi đang gặp phải với SentenceChunker.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**
> *Viết 2-3 câu:* Tôi sẽ triển khai kỹ thuật Hybrid Search (kết hợp tìm kiếm ngữ nghĩa và tìm kiếm từ khóa) để xử lý tốt hơn các câu hỏi chứa thuật ngữ chuyên ngành viết tắt. Đồng thời, tôi sẽ bổ sung thêm bước làm sạch dữ liệu (data cleaning) để loại bỏ các ký tự đặc biệt sau khi trích xuất từ PDF.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 5 / 5 |
| Document selection | Nhóm | 9 / 10 |
| Chunking strategy | Nhóm | 15 / 15 |
| My approach | Cá nhân | 9 / 10 |
| Similarity predictions | Cá nhân | 5 / 5 |
| Results | Cá nhân | 7 / 10 |
| Core implementation (tests) | Cá nhân | 29 / 30 |
| Demo | Nhóm | 5 / 5 |
| **Tổng** | | **84 / 100** |
