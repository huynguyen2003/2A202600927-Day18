# Reflection: Production RAG Pipeline
**Họ và tên:** Nguyễn Quang Huy
**Nhóm:** Nguyễn Quang Huy
## Phần 1: Mapping bài giảng

| Lecture Concept | Module | Hàm cụ thể | Observation |
|----------------|--------|-------------|-------------|
| Semantic chunking | M1 | `chunk_semantic()` | Chunk dựa trên semantic similarity tốt hơn split cơ bản, bảo toàn được ý nghĩa của các đoạn văn logic liên tiếp. Việc áp dụng threshold 0.85 giúp tối ưu việc tách đoạn thay vì dựa trên fixed size tokens. |
| Hierarchical chunking | M1 | `chunk_hierarchical()` | Kỹ thuật Parent-Child giúp tìm kiếm nhanh trên Child (256 chars) nhưng cung cấp context đầy đủ từ Parent (2048 chars) khi ghép vào LLM prompt. |
| Structure-Aware chunking | M1 | `chunk_structure_aware()` | Duy trì cấu trúc Markdown, giúp gộp các list và section lại với nhau để không bị đứt gãy thông tin quan trọng. |
| BM25 + Dense fusion | M2 | `reciprocal_rank_fusion()` | Phương pháp RRF giải quyết điểm yếu của từng loại search: BM25 tìm tốt keyword (tên riêng, mã lỗi) trong khi Dense (bge-m3) hiểu tốt ngữ nghĩa. |
| Cross-encoder reranking | M3 | `CrossEncoderReranker.rerank()` | Reranking bằng Cross-Encoder (bge-reranker-v2-m3) cải thiện độ chính xác ranking top k, giúp loại bỏ nhiễu từ hybrid search ban đầu, mặc dù độ trễ (latency) sẽ tăng nhẹ so với bi-encoder. |
| RAGAS 4 metrics | M4 | `evaluate_ragas()` | RAGAS là framework tốt cho evaluation không cần ground truth thủ công (hoặc chỉ một phần), đánh giá đa chiều từ Faithfulness, Relevancy tới Precision/Recall của context. |
| Contextual embeddings | M5 | `contextual_prepend()` / `_enrich_single_call()` | Giảm retrieval failure bằng cách thêm tóm tắt hoặc bổ sung meta-data/hypothetical questions (HyQA). Phương pháp gọi gộp `_enrich_single_call()` tiết kiệm số lần gọi API so với gọi rời. |

## Phần 2: Khó khăn & giải quyết

- **Lỗi gặp phải**: Lỗi kết nối Qdrant khi không chạy Docker trên Windows.
- **Cách debug**: Thay vì dùng `QdrantClient(host=...)`, chuyển sang dùng local disk storage (`path="qdrant_db"`) để lưu index trực tiếp mà không cần cài đặt Qdrant container.
- **Lỗi gặp phải**: Lỗi mạng (timeout/connection reset) trong quá trình tải mô hình lớn (bge-m3, bge-reranker-v2-m3) từ thư viện sentence-transformers qua Hugging Face Hub (`[WinError 10054] An existing connection was forcibly closed by the remote host`).
- **Cách debug**: Cấu hình chạy lại tiến trình (retry logic hoặc khởi động lại pipeline). Thiết lập local model fallback nếu cần.
- **Kiến thức thiếu**: Tokenization đặc thù cho ngôn ngữ Tiếng Việt (chẳng hạn dùng Underthesea cho BM25) chưa thật sự hoàn hảo do yêu cầu tiền xử lý (thay thế khoảng trắng bằng dấu gạch dưới). Cách bổ sung là tiếp tục tìm hiểu thêm về các công cụ tokenizer chuyên sâu khác và tinh chỉnh logic replace string phù hợp.

## Phần 3: Action Plan cho project

### Hiện tại
- RAG pipeline hiện tại: Đã hoàn tất phiên bản naive, và đang nâng cấp sang mô hình chuẩn production RAG với hybrid search và LLM enrichment.
- Known issues: Vẫn còn thiếu hụt do mô hình chạy trên CPU chậm; RAGAS evaluation và Enrichment có thể bị giới hạn nếu thiếu OpenAI API key hợp lệ hoặc rate-limits của API.

### Plan áp dụng
1. [x] **Chunking strategy**: Kết hợp Hierarchical + Semantic chunking cho tài liệu phức tạp (có lợi cho context retrieval chính xác hơn).
2. [x] **Search**: Sử dụng Hybrid (BM25 + Dense BAAI/bge-m3), bởi vì Tiếng Việt có nhiều từ đồng nghĩa và từ vựng đặc thù. BM25 rất quan trọng để bù đắp các cụm từ khó.
3. [x] **Reranking**: Sử dụng BAAI/bge-reranker-v2-m3, bởi vì reranking đóng vai trò cực quan trọng trong cross-lingual/multilingual RAG.
4. [x] **Evaluation**: Dùng RAGAS kết hợp custom prompt evaluation khi test set lớn.
5. [x] **Enrichment**: Sử dụng Combined (1 prompt) để tối ưu hóa chi phí API, sinh ra Contextual pre-pend, Metadata và HyQA cùng lúc.

### Timeline
- Tuần 1: Tối ưu hoá phần tiền xử lý dữ liệu và fix các lỗi tokenizer.
- Tuần 2: Tích hợp Qdrant cloud hoặc docker thay vì local db, đánh giá bằng API Key xịn.
- Tuần 3: Deploy lên production API (FastAPI) + xây dựng UI.
