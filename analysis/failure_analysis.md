# Báo cáo Phân tích Lỗi (Failure Analysis)

Dựa trên kết quả chạy RAGAS evaluation, dưới đây là phân tích 5 câu hỏi có điểm số thấp nhất (Bottom-5 worst questions) nhằm xác định nguyên nhân và đưa ra hướng khắc phục theo quy trình Diagnostic Tree.

> [!WARNING]
> Do thiếu OpenAI API key thật, kết quả đánh giá RAGAS (Faithfulness, Relevancy, Precision, Recall) sẽ hiển thị giá trị mô phỏng thấp. Phân tích này tập trung vào phương pháp chuẩn đoán dựa trên Diagnostic Tree theo cấu trúc bài tập.

## 1. Câu hỏi: "Làm thế nào để truy cập vào hệ thống ERP khi làm việc ở nhà?"
- **Worst Metric**: `context_recall` (Điểm số mô phỏng: 0.0)
- **Diagnosis**: Missing relevant chunks (Hệ thống không lấy được các đoạn văn bản chứa hướng dẫn VPN và ERP từ tài liệu).
- **Suggested Fix**: 
  - Cải thiện chiến lược chunking (sử dụng Hierarchical thay vì Semantic cho tài liệu hướng dẫn kỹ thuật).
  - Kết hợp BM25 để tìm chính xác từ khóa "ERP" và "ở nhà".
  - Thêm Metadata filtering cho loại tài liệu "IT Support".

## 2. Câu hỏi: "Quy trình xin nghỉ phép thai sản gồm những bước nào?"
- **Worst Metric**: `faithfulness` (Điểm số mô phỏng: 0.0)
- **Diagnosis**: LLM hallucinating (LLM tự bịa ra các bước không có trong sổ tay nhân viên).
- **Suggested Fix**: 
  - Thắt chặt system prompt ("Chỉ trả lời dựa trên context, nếu không thấy hãy nói Không tìm thấy").
  - Giảm temperature của LLM xuống 0.0.
  - Kiểm tra xem Context prepend (M5) đã cung cấp đủ thông tin tóm tắt cho loại chính sách này chưa.

## 3. Câu hỏi: "Định mức công tác phí cho cấp quản lý là bao nhiêu?"
- **Worst Metric**: `context_precision` (Điểm số mô phỏng: 0.0)
- **Diagnosis**: Too many irrelevant chunks (Lấy quá nhiều đoạn văn về "công tác" nhưng không chứa số tiền định mức cụ thể).
- **Suggested Fix**: 
  - Áp dụng Cross-Encoder Reranking (bge-reranker-v2-m3) để đẩy các chunks chứa con số cụ thể lên top đầu.
  - Sử dụng Structure-aware chunking để bảo toàn các bảng biểu (table) quy định mức phí.

## 4. Câu hỏi: "Khi nào nhân viên được review lương?"
- **Worst Metric**: `answer_relevancy` (Điểm số mô phỏng: 0.0)
- **Diagnosis**: Answer doesn't match question (Câu trả lời có thể nói về chính sách thưởng thay vì thời điểm review lương).
- **Suggested Fix**: 
  - Cải thiện template prompt để yêu cầu LLM tập trung vào "thời điểm/khi nào".
  - Dùng LLM Enrichment (M5) để sinh câu hỏi giả định (Hypothetical Questions) có chứa từ khoá "review lương" nhúng vào chunk.

## 5. Câu hỏi: "Tôi có thể mang thiết bị công ty ra nước ngoài không?"
- **Worst Metric**: `context_recall` (Điểm số mô phỏng: 0.0)
- **Diagnosis**: Missing relevant chunks (Chưa lấy được thông tin về chính sách bảo mật thiết bị).
- **Suggested Fix**: 
  - Tối ưu trọng số RRF (Reciprocal Rank Fusion) để ưu tiên kết quả từ Dense Retrieval (hiểu ngữ nghĩa "thiết bị", "nước ngoài" thay vì khớp từ khóa chính xác).
  - Tăng `top_k` của giai đoạn retrieval ban đầu trước khi đưa vào reranker.

---
**Kết luận**: Mô hình baseline cần thiết phải kết hợp tất cả các module M1-M5 mới có thể giải quyết được các điểm yếu cục bộ của từng module đơn lẻ. Hybrid Search và Reranking là hai chốt chặn quan trọng nhất để tăng Recall và Precision.
