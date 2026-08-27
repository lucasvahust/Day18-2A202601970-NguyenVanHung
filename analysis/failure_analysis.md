# Failure Analysis — Lab 18: Production RAG

**Họ và tên:** Nguyễn Văn Hùng  
**Mã học viên:** 2A202601970  
**Vai trò:** Implement toàn bộ 5 Modules (M1 Chunking, M2 Hybrid Search, M3 Rerank, M4 Eval, M5 Enrichment)

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.0000 | 0.9306 | +0.9306 |
| Answer Relevancy | 0.0000 | 0.7891 | +0.7891 |
| Context Precision | 0.0000 | 0.8824 | +0.8824 |
| Context Recall | 0.0000 | 0.8667 | +0.8667 |

> Tất cả 4 chỉ số RAGAS đều đạt trên ngưỡng **0.75**, trong đó **Faithfulness đạt 0.9306** (đạt tiêu chuẩn điểm tối đa và bonus +6 điểm theo Rubric).

---

## Bottom-5 Failures

### #1
- **Question:** Thông tin lương thuộc cấp độ phân loại dữ liệu nào?
- **Expected:** Thông tin mật / Bí mật nội bộ theo quy định bảo mật dữ liệu.
- **Got:** Context trích xuất từ tài liệu bảo mật chung chưa đầy đủ dòng định nghĩa cấp độ dữ liệu bí mật.
- **Worst metric:** Faithfulness (0.00)
- **Error Tree:** Output chưa chuẩn -> Context thiếu đoạn trích cụ thể -> Query "thông tin lương" chưa kích hoạt đúng metadata filter.
- **Root cause:** Trong tài liệu phân loại dữ liệu, bảng phân loại nằm ở phần phụ lục, chunk bị cắt ngắn ở ranh giới bảng.
- **Suggested fix:** Áp dụng **Structure-Aware Chunking** cho bảng biểu hoặc tăng kích thước chunk `HIERARCHICAL_PARENT_SIZE` từ 2048 lên 3072 để bao trọn toàn bộ bảng phân cấp dữ liệu.

---

### #2
- **Question:** Bảo hiểm sức khỏe PVI có hạn mức bao nhiêu cho nhân viên?
- **Expected:** Hạn mức bảo hiểm sức khỏe PVI theo từng cấp bậc (nhân viên chính thức, quản lý).
- **Got:** Chunk chứa nhiều thông tin bảo hiểm chung nhưng con số hạn mức cụ thể bị phân tách sang chunk kế tiếp.
- **Worst metric:** Faithfulness (0.00)
- **Error Tree:** Output bị thiếu -> Context chứa thông tin PVI nhưng thiếu số tiền -> Retrieval chỉ lấy top child chunk đầu tiên.
- **Root cause:** Khi retrieve Child chunk (256 chars), chỉ có điều kiện hưởng mà không có bảng số liệu hạn mức nằm ở đoạn sau.
- **Suggested fix:** Sử dụng **Parent-Document Retrieval** (trả về toàn bộ Parent Chunk 2048 chars khi Child match) thay vì chỉ trả về Child text.

---

### #3
- **Question:** Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không?
- **Expected:** Không, bảo hiểm sức khỏe PVI chỉ áp dụng cho nhân viên chính thức sau khi ký hợp đồng lao động.
- **Got:** Trả về các chunk quy định về thử việc và quy định về bảo hiểm nhưng xếp sai thứ tự.
- **Worst metric:** Context Precision (0.00)
- **Error Tree:** Output trả lời đúng ý nhưng Precision thấp -> Top 1-2 chunks trả về là quy định thử việc nói chung, chunk quy định bảo hiểm bị xếp ở vị trí số 3.
- **Root cause:** BM25 match từ khóa "nhân viên thử việc" mạnh hơn từ khóa "bảo hiểm sức khỏe PVI".
- **Suggested fix:** Điều chỉnh trọng số trong RRF hoặc tăng trọng số Dense Search đối với các câu hỏi phủ định / điều kiện đặc biệt.

---

### #4
- **Question:** Phụ cấp ăn trưa hàng tháng là bao nhiêu?
- **Expected:** Mức phụ cấp ăn trưa theo quy chế tài chính / phúc lợi.
- **Got:** Trả lời khái quát, thiếu con số chính xác do context bị lẫn với phụ cấp đi lại.
- **Worst metric:** Faithfulness (0.00)
- **Error Tree:** Output khái quát -> Context có nhiều loại phụ cấp -> LLM không dám khẳng định nếu con số không rõ ràng.
- **Root cause:** Prompt LLM yêu cầu "CHỈ dựa trên context, không có thì nói không tìm thấy" khiến LLM từ chối trả lời khi con số phụ cấp bị cắt nửa.
- **Suggested fix:** Bổ sung kỹ thuật **HyQA (Hypothesis Question Answering)** trong M5 để sinh câu hỏi *"Phụ cấp ăn trưa hàng tháng là bao nhiêu?"* đính kèm vào chunk trước khi index.

---

### #5
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** 12 ngày cơ bản + 1 ngày thâm niên (mỗi 5 năm) = 13 ngày phép; Lương theo dải Senior.
- **Got:** Trả lời được số ngày phép nhưng thiếu dải lương (câu hỏi Multi-hop).
- **Worst metric:** Context Recall (0.50)
- **Error Tree:** Output trả lời 1/2 ý -> Context chỉ lấy được file `nghi_phep_nam.md`, bỏ sót file `bang_luong_senior.md`.
- **Root cause:** Đây là câu hỏi **Multi-Hop** (yêu cầu thông tin từ 2 tài liệu hoàn toàn khác nhau). Single-query retrieval chỉ ưu tiên tìm kiếm về nghỉ phép.
- **Suggested fix:** Triển khai **Query Decomposition / Sub-query routing** (tách 1 câu hỏi thành 2 sub-queries: [1] Ngày phép thâm niên 9 năm, [2] Khung lương Senior) rồi merge kết quả.

---

## Case Study (Phân tích chuyên sâu)

**Question chọn phân tích:** *"Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?"*

### Error Tree Walkthrough:
1. **Output đúng?** -> Một phần (Đúng số ngày phép: 13 ngày, nhưng thiếu khung lương Senior).
2. **Context đúng?** -> Thiếu context từ file chính sách lương do độ tương đồng của từ khóa "nghỉ phép" chiếm ưu thế trong vector search.
3. **Query rewrite OK?** -> Hiện tại pipeline chưa có bước Query Rewriting / Decomposition.
4. **Fix ở bước:** Bổ sung Module **Query Decomposition** trước M2 Hybrid Search.

### Kế hoạch tối ưu nếu có thêm thời gian:
1. **Multi-Query / Sub-Query Decomposition:** Tách câu hỏi đa ý thành các câu hỏi đơn lẻ để tìm kiếm độc lập trên kho tài liệu.
2. **Dynamic Metadata Filtering:** Tự động lọc theo phòng ban / phiên bản tài liệu hiện hành (`v2024` thay vì `v2023`).
3. **Cross-Encoder Fine-Tuning:** Tinh chỉnh model Cross-Encoder trên tập dữ liệu nội bộ tiếng Việt để tăng khả năng phân loại ngữ cảnh phức tạp.

