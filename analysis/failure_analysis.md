# Failure Analysis — Lab 18: Production RAG

**Họ và tên học viên:** Mai Quang Dũng  
**Mã số học viên (MSSV):** 2A202602966  
**Khóa:** K4 - Track 3A  

---

## 1. RAGAS Scores Comparison

| Metric | Naive Baseline | Production Pipeline | Δ (Biến thiên) | Đánh giá |
|--------|:--------------:|:-------------------:|:--------------:|:---------|
| **Faithfulness** | 0.8167 | **0.8783** | **+0.0617** | Đạt chuẩn xuất sắc (≥ 0.85), nhận điểm Bonus |
| **Answer Relevancy** | 0.7615 | **0.8089** | **+0.0474** | Cải thiện đáng kể, trả lời súc tích, bám sát câu hỏi |
| **Context Precision** | 0.9250 | **0.9667** | **+0.0417** | Reranker loại bỏ nhiễu cực tốt, đưa đúng context lên rank 1-2 |
| **Context Recall** | 0.9250 | **0.8917** | **-0.0333** | Cắt top-3 từ reranker giúp lọc sạch nhưng đôi khi lọc mất vế phụ |

> **Nhận xét tổng quan:** 
> - Cả 4 metrics của Production Pipeline đều đạt trên **0.80** (vượt xa tiêu chuẩn 0.70 và thỏa mãn điều kiện Bonus ≥ 0.75 cho toàn bộ metrics).
> - **Faithfulness đạt 0.8783** (vượt mốc 0.85) chứng minh việc kết hợp Contextual Prepend ở M5 cùng Cross-Encoder Reranking ở M3 đã triệt tiêu phần lớn hiện tượng hallucination của LLM.

---

## 2. Latency Breakdown Report (Bảng thời gian từng bước)

Dưới đây là thời gian xử lý thực tế đo đạc được trên toàn bộ tài liệu (26 files, 104 chunks, 20 test queries):

| Giai đoạn Pipeline | Thành phần Module | Thời gian thực thi | Tỷ trọng | Ghi chú & Đánh giá |
|--------------------|-------------------|:------------------:|:--------:|---------------------|
| **1. Chunking** | M1: Hierarchical Chunking | 0.1s | 0.02% | Tách Parent-Child cực nhanh bằng logic bộ nhớ |
| **2. Enrichment** | M5: Combined Single-call (`gpt-4o-mini`) | 326.5s | 74.8% | 1 API call/chunk cho 104 chunks; bổ sung context, hyqa |
| **3. Indexing** | M2: BM25 + Qdrant Dense (`bge-m3`) | 32.1s | 7.4% | Encode 104 chunks bằng bge-m3 và upsert vào Qdrant |
| **4. Retrieval & Rerank** | M2 Hybrid + M3 Cross-Encoder | 18.2s | 4.2% | ~0.9s/query (BM25 + Dense + RRF top-20 → Cross-Encoder top-3) |
| **5. Generation** | LLM Answer Generation (`gpt-4o-mini`) | 25.4s | 5.8% | ~1.2s/query sinh câu trả lời có trích dẫn |
| **6. Evaluation** | M4: RAGAS Evaluation (4 metrics) | 33.5s | 7.7% | Chấm tự động 80 evaluations (4 metrics × 20 questions) |
| **Tổng cộng** | **End-to-End Pipeline** | **435.8s** | **100%** | Pipeline chạy ổn định, hoàn thành toàn bộ bài lab |

---

## 3. Bottom-5 Failures Analysis (Phân tích 5 ca lỗi điển hình)

### #1
- **Question:** Nghỉ phép không lương 20 ngày cần ai phê duyệt?
- **Expected (Ground Truth):** Nghỉ 16-30 ngày cần phê duyệt của Giám đốc điều hành (CEO). Lưu ý: nghỉ trên 14 ngày không lương, nhân viên phải tự đóng phần bảo hiểm của mình.
- **Got (Answer):** Nghỉ phép không lương 20 ngày cần phê duyệt của Giám đốc điều hành (CEO).
- **Worst metric:** Faithfulness (0.50) / Context Recall
- **Error Tree:** Output đúng người phê duyệt → Thiếu ý phụ về bảo hiểm xã hội → Context có chứa điều khoản bảo hiểm nhưng câu hỏi chỉ hỏi "ai phê duyệt" nên LLM lược bỏ.
- **Root cause:** Câu hỏi có tính chất hỏi đơn giản ("ai phê duyệt?"), câu trả lời hoàn toàn chính xác về mặt nghiệp vụ, nhưng ground-truth lại chứa thêm điều khoản bổ sung ("tự đóng phần bảo hiểm"). Ragas chấm điểm faithfulness/recall bị phạt do thiếu ý phụ này.
- **Suggested fix:** Cải tiến prompt để chỉ dẫn LLM: *"Khi trả lời quy chế hoặc điều kiện nhân sự, hãy đính kèm các điều kiện ràng buộc hoặc lưu ý quan trọng liên quan."*

---

### #2
- **Question:** Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không?
- **Expected (Ground Truth):** KHÔNG. Nhân viên thử việc chưa được hưởng gói bảo hiểm sức khỏe PVI. Chỉ được tham gia bảo hiểm xã hội bắt buộc.
- **Got (Answer):** Không tìm thấy.
- **Worst metric:** Answer Relevancy (0.00)
- **Error Tree:** Output: "Không tìm thấy" → Context bị miss thông tin về bảo hiểm cho nhân viên thử việc → Query tìm kiếm bị rơi vào file "thử việc" chung chung thay vì file "bảo hiểm sức khỏe".
- **Root cause:** Trong tài liệu `thu_viec.md` không đề cập cụ thể tên gói "PVI", còn tài liệu `bao_hiem_suc_khoe.md` chỉ ghi "áp dụng cho nhân viên chính thức ký HĐLĐ". Reranker đã xếp đoạn về thử việc lên trên, nhưng đoạn này lại không nhắc đến PVI, khiến LLM tuân thủ nghiêm ngặt prompt an toàn và trả lời "Không tìm thấy".
- **Suggested fix:** Bổ sung Cross-document reasoning hoặc HyQA: khi chunking tài liệu `bao_hiem_suc_khoe.md`, sinh câu hỏi giả định: *"Nhân viên thử việc có được hưởng bảo hiểm PVI không?"* $\rightarrow$ Query sẽ match trực tiếp vào chunk này.

---

### #3
- **Question:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Expected (Ground Truth):** Thời hạn thanh toán là 15 ngày. Quá hạn 5 ngày, bị tính phí 2%/tháng trên 15.000.000 VNĐ = 300.000 VNĐ/tháng (tính pro-rata khoảng 50.000 VNĐ cho 5 ngày).
- **Got (Answer):** Nhân viên sẽ bị phạt 300,000 VNĐ (tính 2% của 15 triệu VNĐ = 300,000 VNĐ).
- **Worst metric:** Faithfulness (0.40)
- **Error Tree:** Output tính nguyên 1 tháng thay vì pro-rata theo ngày → Context ghi quy định phạt chung chung 2%/tháng → LLM không tự chia nhỏ pro-rata 5 ngày.
- **Root cause:** Lỗi suy luận toán học (Mathematical reasoning). Context chỉ ghi công thức lãi suất phạt tháng, không ghi rõ ví dụ tính theo ngày, khiến LLM nhân trực tiếp 2% vào số tiền gốc.
- **Suggested fix:** Thêm few-shot reasoning trong system prompt hoặc hướng dẫn Chain-of-Thought (CoT) cho các câu hỏi tính toán phạt tài chính.

---

### #4
- **Question:** Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai phê duyệt và cần gì từ phòng CNTT?
- **Expected (Ground Truth):** Laptop 30 triệu nằm trong khoảng 5-50 triệu nên cần Giám đốc phòng ban (Director) phê duyệt. Ngoài ra, mua sắm thiết bị CNTT cần có xác nhận cấu hình kỹ thuật từ phòng CNTT trước khi đề xuất. Cần đính kèm ít nhất 3 báo giá vì trên 10 triệu.
- **Got (Answer):** Phê duyệt sẽ cần được thực hiện bởi Kế toán trưởng vì giá trị đơn hàng trên 5.000.000 VNĐ. Ngoài ra, cần có xác nhận của phòng CNTT về cấu hình kỹ thuật của laptop trước khi đề xuất.
- **Worst metric:** Context Recall (0.33)
- **Error Tree:** Output sai thẩm quyền phê duyệt (Kế toán trưởng vs Giám đốc phòng ban) → Context retrieved từ file `mua_sam.md` bị cắt khúc giữa bảng thẩm quyền phê duyệt và quy trình CNTT.
- **Root cause:** Mâu thuẫn giữa 2 tài liệu: một văn bản quy định chung về duyệt chi (Kế toán trưởng duyệt đơn > 5 triệu) và một văn bản quy định riêng về mua sắm tài sản cố định (Giám đốc phòng ban duyệt 5-50 triệu). Reranker top-3 chỉ lấy được 1 trong 2 ngữ cảnh.
- **Suggested fix:** Áp dụng Structure-aware chunking giữ nguyên bảng ma trận thẩm quyền phê duyệt không bị ngắt nhỏ, đồng thời tăng rerank top_k từ 3 lên 5 đối với các câu hỏi phức hợp đa điều kiện.

---

### #5
- **Question:** Nhân viên được tài trợ khóa học 25 triệu, nghỉ việc sau 8 tháng hoàn thành khóa học. Phải hoàn trả bao nhiêu?
- **Expected (Ground Truth):** Nhân viên phải cam kết làm việc ít nhất 1 năm sau khi hoàn thành khóa học. Nghỉ sau 8 tháng là trước hạn cam kết, phải hoàn trả 100% chi phí tức 25.000.000 VNĐ.
- **Got (Answer):** Nhân viên phải hoàn trả **100% chi phí** đào tạo đã được tài trợ, tức là **25 triệu VNĐ**.
- **Worst metric:** Faithfulness (0.50)
- **Error Tree:** Output đúng 100% số tiền → Thiếu giải thích mốc cam kết 1 năm → RAGAS faithfulness phạt do không nêu tiền đề cam kết.
- **Root cause:** LLM trả lời quá ngắn gọn (chỉ nêu kết quả cuối cùng 25 triệu VNĐ) mà không trích dẫn lại điều kiện cam kết 1 năm có trong context.
- **Suggested fix:** Căn chỉnh prompt: *"Hãy giải thích căn cứ điều khoản quy định trước khi đưa ra kết luận số tiền/quyết định."*

---

## 4. Case Study chuyên sâu (Phục vụ Presentation)

**Câu hỏi nghiên cứu:**  
> *"Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai phê duyệt và cần gì từ phòng CNTT?"*

**Error Tree Walkthrough từng bước:**
1. **Output đúng không?**  
   $\rightarrow$ Đúng 50%: Trả lời đúng phần cần xác nhận cấu hình từ phòng CNTT, nhưng sai cấp phê duyệt (trả lời Kế toán trưởng thay vì Giám đốc phòng ban) và thiếu yêu cầu 3 báo giá.
2. **Context tìm về có đúng không?**  
   $\rightarrow$ Một phần đúng. Context lấy được đoạn quy định về thiết bị CNTT trong `mua_sam.md`, nhưng đoạn ma trận phân quyền chi tiết (5 - 50 triệu $\rightarrow$ Director) bị rank thấp hơn do từ khóa "laptop" khớp mạnh hơn với đoạn CNTT.
3. **Query Retrieval & Fusion có ổn không?**  
   $\rightarrow$ BM25 tìm từ khóa "laptop", Dense tìm theo "mua thiết bị cho nhân viên mới". Cả hai nguồn đều bị phân mảnh vì quy định mua sắm laptop nằm rải rác ở 2 section khác nhau.
4. **Điểm can thiệp sửa đổi (Fix):**  
   $\rightarrow$ Sửa ở khâu **M1 (Chunking) & M5 (Enrichment)**:
   - Tại M1: Giữ nguyên toàn bộ bảng Phân quyền phê duyệt bằng Structure-aware Chunking.
   - Tại M5: Contextual Prepend bổ sung rõ: *"Quy trình mua sắm thiết bị CNTT trên 10 triệu bao gồm ma trận phê duyệt của Director và xác nhận kỹ thuật từ phòng CNTT."*

---

## 5. Nếu có thêm 1 giờ, tôi sẽ tối ưu hóa những gì?

1. **Thêm cơ chế Query Rewriting / Sub-query Decomposition:**  
   Đối với các câu hỏi phức hợp gồm 2 vế (ví dụ: vừa hỏi thâm niên phép năm, vừa hỏi dải lương Senior), tách thành 2 sub-queries riêng biệt để retrieval độc lập rồi gộp context trước khi đưa vào LLM.
2. **Dynamic Top-K Reranking dựa trên độ tin cậy của Cross-Encoder:**  
   Thay vì cố định top 3 chunks, áp dụng ngưỡng score threshold (ví dụ: chỉ giữ các chunk có `rerank_score > 0.0`). Nếu câu hỏi cần nhiều ngữ cảnh liên quan, linh hoạt lấy từ 3 đến 5 chunks.
3. **Fine-tune Prompt Generation với Chain-of-Thought (CoT):**  
   Bắt buộc LLM trích dẫn nguyên văn điều khoản liên quan trước khi suy luận ra con số cuối cùng, giúp Faithfulness tăng lên mức tiệm cận 0.95+.
