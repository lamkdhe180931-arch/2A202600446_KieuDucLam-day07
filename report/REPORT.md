# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Kiều Đức Lâm
**Nhóm:** X5
**Ngày:** 10/4/2026

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**
high cosine similarity nghĩa là có độ tương đồng cao, nó thể hiện mức độ ngữ nghĩa, tương quan gần nhau của hai vector, từ đó ta có thể chọn các vector gần nhất với câu hoie để đưa ra câu trả lời chính xác.

**Ví dụ HIGH similarity:**
- Sentence A: Tôi thích học machine learning vì nó giúp tôi giải quyết các vấn đề thực tế
- Sentence B: Tôi yêu việc học machine learning vì nó giúp tôi xử lý các bài toán trong thực tiễn.
- Tại sao tương đồng: ý nghĩa tương đối giống nhau 

**Ví dụ LOW similarity:**
- Sentence A: Tôi thích học machine learning vì nó giúp tôi giải quyết các vấn đề thực tế.
- Sentence B: Hôm nay trời mưa rất to và tôi phải ở nhà cả ngày.
- Tại sao khác: hai câu không liên quan về ngữ nghĩa (một câu về học tập, một câu về thời tiết).
                Không có từ khóa chung hoặc chủ đề chung.


**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**
Cosine similarity đo góc giữa hai vector nên tập trung vào ý nghĩa (hướng) thay vì độ lớn. Trong khi đó, Euclidean distance bị ảnh hưởng bởi độ dài vector, nên không phản ánh tốt mức độ tương đồng ngữ nghĩa của văn bản

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
bước dịch mỗi lần 
= 500 - 50 = 450
Số chunks:
= ceil((10000 - 500) / 450) + 1
= ceil(9500 / 450) + 1
≈ ceil(21.11) + 1
= 22 + 1 = 23 chunks
**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**
Stride mới:
= 500 - 100 = 400
Số chunks:
= ceil((10000 - 500) / 400) + 1
= ceil(9500 / 400) + 1
≈ ceil(23.75) + 1
= 24 + 1 = 25 chunks

- Overlap lớn giúp các chunk giữ được ngữ cảnh liên tục giữa các đoạn, tránh mất thông tin quan trọng ở ranh giới, từ đó cải thiện chất lượng tìm kiếm và trả lời.

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

**Loại:** FixedSizeChunker

**Mô tả cách hoạt động:**
Strategy này chia văn bản thành các đoạn có độ dài cố định (chunk_size = 500) và có overlap giữa các chunk (overlap = 50). Mỗi lần cắt sẽ dịch một bước step = chunk_size - overlap = 450, giúp các chunk liền kề có phần nội dung trùng nhau. Điều này đảm bảo không bị mất thông tin ở ranh giới giữa các đoạn. Nếu văn bản ngắn hơn chunk_size thì chỉ tạo một chunk duy nhất.

**Tại sao tôi chọn strategy này cho domain nhóm?**
FixedSizeChunker đơn giản, dễ kiểm soát và đảm bảo độ dài chunk ổn định, phù hợp khi làm việc với embedding model có giới hạn token. Ngoài ra, việc overlap giúp giảm mất mát ngữ cảnh, từ đó vẫn đảm bảo khả năng truy xuất thông tin chính xác trong tài liệu quy chế.    

**Code snippet (nếu custom):**
```python
# Paste implementation here
def load_documents_from_files(file_paths: list[str]) -> list[Document]:
    """
    Implementation của FixedSizeChunker Strategy:
    - Chia văn bản theo kích thước cố định (chunk_size=500, overlap=50)
    - Gán metadata cho từng chunk
    """
    chunker = FixedSizeChunker(chunk_size=500, overlap=50)
    documents: list[Document] = []

    for raw_path in file_paths:
        path = Path(raw_path)

        if not path.exists() or not path.is_file():
            continue

        content = path.read_text(encoding="utf-8")

        # Thực hiện chunking theo fixed size
        for i, chunk_str in enumerate(chunker.chunk(content)):
            documents.append(
                Document(
                    id=str(i),
                    content=chunk_str,
                    metadata={
                        "chunk_id": i,
                        "source": str(path),
                        "extension": path.suffix.lower()
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
| Tôi | FixedSizeChunker ( 497.30 chars) | 8/10 | Trả về nội dung gãy gọn, đúng trọng tâm câu hỏi về quy định. | Đôi khi mất tiêu đề "Điều/Khoản" nếu tiêu đề nằm ở chunk trước. |
| Lâm | RecursiveCharacter | 9/10 | Giữ được ngữ cảnh của toàn bộ mục lớn (Section). | Số lượng ký tự mỗi chunk không đều, dễ gây quá tải token cho LLM. |

**Strategy nào tốt nhất cho domain này? Tại sao?**
Đối với domain quy chế đào tạo, RecursiveChunker là tốt nhất vì giữ được cấu trúc Điều/Khoản. Tuy nhiên, FixedSizeChunker vẫn là lựa chọn hợp lý khi cần đảm bảo độ dài chunk ổn định và tối ưu cho embedding model.

---

## 4. My Approach — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi implement các phần chính trong package `src`.

### Chunking Functions

**`SentenceChunker.chunk`** — approach:
Sử dụng regex (?<=[.!?])(?:\s+|\n+) để tách câu dựa trên dấu ., !, ? và giữ lại dấu câu ở cuối câu. Sau đó nhóm các câu lại theo max_sentences_per_chunk. Có xử lý edge case như text rỗng, nhiều khoảng trắng hoặc newline liên tiếp.

**`RecursiveChunker.chunk` / `_split`** — approach:
Thuật toán chia văn bản theo thứ tự ưu tiên separator (\n\n → \n → ". " → " " → ""), cố gắng giữ chunk nhỏ hơn chunk_size. Nếu không thể chia bằng separator hiện tại thì đệ quy với separator tiếp theo. Base case là khi độ dài đủ nhỏ hoặc không còn separator thì cắt cứng theo chunk_size.

### EmbeddingStore

**`add_documents` + `search`** — approach:
Mỗi document được chuyển thành embedding vector bằng embedding_fn và lưu dưới dạng record (id, content, embedding, metadata). Nếu có ChromaDB thì lưu vào collection, nếu không thì lưu in-memory list. Khi search, query được embed và tính similarity bằng dot product với từng vector, sau đó sort giảm dần và lấy top-k.

**`search_with_filter` + `delete_document`** — approach:
Filter được áp dụng trước khi tính similarity (pre-filter) để giảm số lượng record cần so sánh. Với ChromaDB dùng where, còn in-memory thì lọc bằng metadata. Delete document bằng cách xóa tất cả record có doc_id tương ứng (dựa trên metadata), cả trong Chroma hoặc list.

### KnowledgeBaseAgent

**`answer`** — approach:
Tạo prompt gồm câu hỏi của user và các đoạn context được retrieve từ EmbeddingStore. Context được inject trực tiếp vào prompt (thường dạng “Context: … Question: …”). Model sẽ dựa trên các đoạn này để sinh câu trả lời chính xác và bám sát dữ liệu.

### Test Results


# Paste output of: pytest tests/ -v
lam@Mac 2A202600446_KieuDucLam-day07 % pytest tests/ -v
====================================================== test session starts ======================================================
platform darwin -- Python 3.12.12, pytest-9.0.2, pluggy-1.6.0 -- /opt/homebrew/opt/python@3.12/bin/python3.12
cachedir: .pytest_cache
rootdir: /Users/lam/Documents/AI thực chiến/assignments /2A202600446_KieuDucLam-day07
plugins: anyio-4.11.0
collected 42 items                                                                                                              

tests/test_solution.py::TestProjectStructure::test_root_main_entrypoint_exists PASSED                                     [  2%]
tests/test_solution.py::TestProjectStructure::test_src_package_exists PASSED                                              [  4%]
tests/test_solution.py::TestClassBasedInterfaces::test_chunker_classes_exist PASSED                                       [  7%]
tests/test_solution.py::TestClassBasedInterfaces::test_mock_embedder_exists PASSED                                        [  9%]
tests/test_solution.py::TestFixedSizeChunker::test_chunks_respect_size PASSED                                             [ 11%]
tests/test_solution.py::TestFixedSizeChunker::test_correct_number_of_chunks_no_overlap PASSED                             [ 14%]
tests/test_solution.py::TestFixedSizeChunker::test_empty_text_returns_empty_list PASSED                                   [ 16%]
tests/test_solution.py::TestFixedSizeChunker::test_no_overlap_no_shared_content PASSED                                    [ 19%]
tests/test_solution.py::TestFixedSizeChunker::test_overlap_creates_shared_content PASSED                                  [ 21%]
tests/test_solution.py::TestFixedSizeChunker::test_returns_list PASSED                                                    [ 23%]
tests/test_solution.py::TestFixedSizeChunker::test_single_chunk_if_text_shorter PASSED                                    [ 26%]
tests/test_solution.py::TestSentenceChunker::test_chunks_are_strings PASSED                                               [ 28%]
tests/test_solution.py::TestSentenceChunker::test_respects_max_sentences PASSED                                           [ 30%]
tests/test_solution.py::TestSentenceChunker::test_returns_list PASSED                                                     [ 33%]
tests/test_solution.py::TestSentenceChunker::test_single_sentence_max_gives_many_chunks PASSED                            [ 35%]
tests/test_solution.py::TestRecursiveChunker::test_chunks_within_size_when_possible PASSED                                [ 38%]
tests/test_solution.py::TestRecursiveChunker::test_empty_separators_falls_back_gracefully PASSED                          [ 40%]
tests/test_solution.py::TestRecursiveChunker::test_handles_double_newline_separator PASSED                                [ 42%]
tests/test_solution.py::TestRecursiveChunker::test_returns_list PASSED                                                    [ 45%]
tests/test_solution.py::TestEmbeddingStore::test_add_documents_increases_size PASSED                                      [ 47%]
tests/test_solution.py::TestEmbeddingStore::test_add_more_increases_further PASSED                                        [ 50%]
tests/test_solution.py::TestEmbeddingStore::test_initial_size_is_zero PASSED                                              [ 52%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_content_key PASSED                                   [ 54%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_score_key PASSED                                     [ 57%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_sorted_by_score_descending PASSED                         [ 59%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_at_most_top_k PASSED                                      [ 61%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_list PASSED                                               [ 64%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_non_empty PASSED                                              [ 66%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_returns_string PASSED                                         [ 69%]
tests/test_solution.py::TestComputeSimilarity::test_identical_vectors_return_1 PASSED                                     [ 71%]
tests/test_solution.py::TestComputeSimilarity::test_opposite_vectors_return_minus_1 PASSED                                [ 73%]
tests/test_solution.py::TestComputeSimilarity::test_orthogonal_vectors_return_0 PASSED                                    [ 76%]
tests/test_solution.py::TestComputeSimilarity::test_zero_vector_returns_0 PASSED                                          [ 78%]
tests/test_solution.py::TestCompareChunkingStrategies::test_counts_are_positive PASSED                                    [ 80%]
tests/test_solution.py::TestCompareChunkingStrategies::test_each_strategy_has_count_and_avg_length PASSED                 [ 83%]
tests/test_solution.py::TestCompareChunkingStrategies::test_returns_three_strategies PASSED                               [ 85%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_filter_by_department PASSED                              [ 88%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_no_filter_returns_all_candidates PASSED                  [ 90%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_returns_at_most_top_k PASSED                             [ 92%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_reduces_collection_size PASSED                      [ 95%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_false_for_nonexistent_doc PASSED            [ 97%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_true_for_existing_doc PASSED                [100%]

====================================================== 42 passed in 0.06s =======================================================
lam@Mac 2A202600446_KieuDucLam-day07 % 

**Số tests pass:** 42 test pass


## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | Sinh viên phải hoàn thành đầy đủ học phí trước khi đăng ký học phần| Việc đăng ký môn học chỉ được thực hiện sau khi sinh viên đóng đủ học phí| high  |0.90 | đúng|
| 2 | Sinh viên bị cảnh báo học vụ nếu GPA dưới mức quy định|Nếu điểm trung bình tích lũy thấp, sinh viên sẽ bị cảnh báo học tập     | high  |0.85 | đúng|
| 3 |Sinh viên được đăng ký tối đa 25 tín chỉ trong một học kỳ |Mỗi kỳ học, số tín chỉ sinh viên có thể đăng ký không vượt quá 25 | high  |0.92 | đúng|
| 4 | Sinh viên phải tham gia kỳ thi cuối kỳ theo lịch của nhà trường|Nhà trường tổ chức các hoạt động ngoại khóa cho sinh viên | low  |0.15 | đúng|
| 5 |Sinh viên được phép học lại học phần đã trượt |Sinh viên có thể đăng ký học lại môn đã không đạt | high  |0.88 | đúng|

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**
Cặp 2 là trường hợp khá thú vị vì hai câu không giống nhau về từ vựng nhưng vẫn có similarity cao. Điều này cho thấy embedding không chỉ dựa vào từ khóa mà còn hiểu được ngữ nghĩa tổng thể của câu. Ngoài ra, cặp 4 có similarity rất thấp dù cùng nói về “sinh viên”, chứng tỏ embedding phân biệt tốt sự khác biệt về nội dung chứ không chỉ dựa vào chủ đề chung.

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

### Kết Quả Của Tôi

| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | Mã số của Quy chế đào tạo VinUni là gì? | # QUY CHẾ ĐÀO TẠO ĐẠI HỌC HỆ CHÍNH QUY THEO HỆ THỐNG TÍN CHỈ  Mã số : VU\_HT03.VN  Đơn vị p... | 0.623 | Yes | mã số là VU_HT03.VN |
| 2 | Tổng số tín chỉ tối thiểu cần đăng ký một kỳ? |Học cùng lúc hai chương trình | 0.22|No |ko có thông tin |
| 3 | GPA bao nhiêu thì được xếp loại học lực Giỏi? |Điều 10. Đăng ký học phần | 0.31|No |không thấy thông tin trong phần đó |
| 4 | Một tín chỉ tương đương bao nhiêu giờ học?|Điều 10. Đăng ký học phần  - 1.... | 0.72`| Yes |50 giờ |
| 5 | Thời gian bảo lưu kết quả học tập tối đa? |Sinh viên phải đăng ký tốt nghiệp trong học kỳ tốt nghiệp dự kiến theo các thủ tục và hướng dẫn của Nhà trường...| 0.63 |Yes |1-2 học kỳ |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 3/ 5

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**
Mình học được cách sử dụng SentenceChunker để giữ câu hoàn chỉnh, giúp câu trả lời tự nhiên hơn. Ngoài ra, việc tuning chunk_size cũng ảnh hưởng lớn đến kết quả.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**
Mình học được cách sử dụng SentenceChunker để giữ câu hoàn chỉnh, giúp câu trả lời tự nhiên hơn. Ngoài ra, việc tuning chunk_size cũng ảnh hưởng lớn đến kết quả.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**
Mình sẽ thêm metadata chi tiết hơn như section, subsection để filter tốt hơn. Ngoài ra, sẽ thử hybrid search (keyword + embedding) để cải thiện kết quả.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 5/ 5 |
| Document selection | Nhóm | 9/ 10 |
| Chunking strategy | Nhóm | 14/ 15 |
| My approach | Cá nhân | 9/ 10 |
| Similarity predictions | Cá nhân | 4/ 5 |
| Results | Cá nhân | 8/ 10 |
| Core implementation (tests) | Cá nhân | 28/ 30 |
| Demo | Nhóm | 5/ 5 |
| **Tổng** | | **82/ 90** |
