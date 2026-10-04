# Individual Reflection — Lab 18: Production RAG

**Họ và tên:** Mai Quang Dũng  
**Mã số học viên (MSSV):** 2A202602966  
**Khóa:** K4 - Track 3A  
**Ngày hoàn thành:** 04/10/2026  

---

## Phần 1: Mapping bài giảng (Lecture Mapping)

Dưới đây là bảng đối chiếu chi tiết giữa các khái niệm lý thuyết trong bài giảng và việc hiện thực hóa trong mã nguồn thực tế của bài lab:

| Lecture Concept | Module | Hàm cụ thể | Observation & Phân tích thực nghiệm |
|----------------|:------:|------------|--------------------------------------|
| **Semantic Chunking & Hierarchical Chunking** | M1 | `chunk_semantic()`, `chunk_hierarchical()`, `chunk_structure_aware()` | Với threshold 0.85, semantic chunking bảo toàn câu cùng ngữ nghĩa tốt hơn cắt đoạn thô. Tuy nhiên, trong production thực tế, **Hierarchical Chunking** (Parent 2048 chars, Child 256 chars) tỏ ra vượt trội: Child chunk nhỏ giúp vector embedding cực kỳ sắc bén (high precision), khi truy xuất gắn `parent_id` giúp LLM nhận trọn vẹn ngữ cảnh xung quanh mà không lo đứt đoạn thông tin. |
| **Hybrid Search & Reciprocal Rank Fusion (RRF)** | M2 | `segment_vietnamese()`, `BM25Search.search()`, `reciprocal_rank_fusion()` | BM25 giải quyết triệt để điểm yếu của Vector Search khi tìm kiếm từ khóa chuyên ngành, số tiền chính xác (*"200.000.000 VNĐ"*), mã quy chế (*"v2.0"*, *"PVI"*). Công thức RRF ($k=60$) kết hợp thứ hạng từ lexical và dense mà không cần chuẩn hóa điểm số, mang lại Context Precision đạt tới 0.9667. |
| **Cross-Encoder Reranking** | M3 | `CrossEncoderReranker.rerank()` | Mô hình `BAAI/bge-reranker-v2-m3` xử lý full cross-attention giữa `(query, document)`, giúp lọc từ top-20 ứng viên xuống top-3 tinh hoa nhất. Reranker triệt tiêu các đoạn văn có chứa từ khóa trùng lặp nhưng ngữ cảnh sai lệch, tăng độ chính xác đầu vào cho LLM với độ trễ trung bình chỉ ~45ms/doc. |
| **RAGAS 4 Metrics & Diagnostic Tree** | M4 | `evaluate_ragas()`, `failure_analysis()` | Đánh giá toàn diện 4 chỉ số cốt lõi: Faithfulness (0.8783), Answer Relevancy (0.8089), Context Precision (0.9667), Context Recall (0.8917). Diagnostic Tree cho phép phân loại tự động nguyên nhân gốc rễ (Root Cause: LLM hallucination, thiếu chunk, nhiễu văn bản) và đưa ra gợi ý khắc phục có tính hệ thống. |
| **Contextual Enrichment (Single-Call Mode)** | M5 | `_enrich_single_call()`, `contextual_prepend()` | Bổ sung định vị ngữ cảnh tài liệu vào đầu chunk theo nghiên cứu của Anthropic (Contextual Prepend) kết hợp sinh câu hỏi giả định (HyQA) và trích xuất metadata. Tối ưu hóa gom 4 tác vụ vào **1 API call duy nhất** giúp tiết kiệm 75% chi phí API token và giảm độ trễ pipeline đáng kể. |

---

## Phần 2: Khó khăn & Cách giải quyết (Challenges & Debugging)

Trong quá trình thực hành, tôi đã gặp và vượt qua 3 thử thách kỹ thuật quan trọng:

1. **Vấn đề tách từ tiếng Việt của `underthesea` với BM25:**
   - **Hiện tượng:** Khi chạy thử nghiệm ban đầu, BM25 không tìm thấy kết quả cho câu hỏi *"nghỉ phép"* dù tài liệu có đầy đủ.
   - **Nguyên nhân gốc rễ:** `underthesea.word_tokenize(format="text")` tự động nối các từ ghép bằng dấu gạch dưới (ví dụ: `"nghỉ_phép"`). Trong khi đó, `rank_bm25` mặc định tách token bằng khoảng trắng (`split(" ")`). Khi người dùng gõ `"nghỉ phép"`, BM25 tách thành 2 token `["nghỉ", "phép"]` nên không thể khớp với token đơn `"nghỉ_phép"` trong chỉ mục.
   - **Cách giải quyết:** Trong hàm `segment_vietnamese()`, thực hiện chuẩn hóa `.replace("_", " ")` trước khi đưa vào BM25 tokenizer, đảm bảo sự đồng nhất tuyệt đối giữa câu hỏi và tài liệu.

2. **Quản lý bộ nhớ và tối ưu hóa nạp mô hình nặng:**
   - **Hiện tượng:** Các test case trong `test_m3.py` khởi tạo nhiều đối tượng `CrossEncoderReranker()`, dẫn đến việc mô hình `bge-reranker-v2-m3` nặng hơn 1GB bị nạp lại nhiều lần từ ổ cứng, gây tràn RAM và làm chậm quá trình kiểm thử.
   - **Cách debug & giải quyết:** Thiết kế mẫu Singleton / Class-level variable caching (`_shared_model` trong `CrossEncoderReranker` và `_shared_encoder` trong `DenseSearch`). Khi một instance mới được tạo, nó tái sử dụng ngay trọng số đã nạp vào bộ nhớ, giảm thời gian khởi tạo từ hàng chục giây xuống 0.0s.

3. **Mạng tải mô hình từ HuggingFace Hub trên môi trường Windows:**
   - **Hiện tượng:** Kết nối trực tiếp đến Hugging Face CDN quốc tế tải file trọng số 1.1GB đôi khi bị nghẽn băng thông dẫn đến timeout.
   - **Cách khắc phục:** Cấu hình biến môi trường `HF_ENDPOINT="https://hf-mirror.com"` kết hợp tải thông qua `snapshot_download(resume_download=True)`, giúp tốc độ tải tăng gấp 5 lần và hoạt động ổn định trên hạ tầng mạng trong nước.

---

## Phần 3: Action Plan cho Project cá nhân (Application Plan)

### Project: Trợ lý AI Tra cứu Quy chế & Pháp lý Doanh nghiệp (Enterprise Policy AI Assistant)

#### 1. Hiện trạng hệ thống hiện tại
- **Kiến trúc cũ:** Sử dụng Naive RAG cơ bản với LangChain, cắt văn bản theo độ dài ký tự cố định (500 ký tự) và tìm kiếm Dense Vector đơn thuần bằng ChromaDB.
- **Điểm nghẽn (Bottlenecks):**
  - Thường xuyên bị đứt gãy bảng biểu nhân sự, quy chế lương thưởng.
  - Khi người dùng tra cứu các con số cụ thể (mã điều khoản, hạn mức tạm ứng, số tiền phạt), vector search thường bỏ sót văn bản liên quan.
  - LLM đôi khi bị "bịa đặt" (hallucination) câu trả lời khi thông tin bị phân tán ở nhiều văn bản khác nhau.

#### 2. Kế hoạch cải tiến dựa trên Lab 18
1. **Chiến lược Chunking:**
   - Áp dụng **Hierarchical Chunking** kết hợp **Structure-Aware Chunking**.
   - Đối với các tài liệu quy chế có heading rõ ràng, phân đoạn theo Header cấp 1-2. Mỗi đoạn lớn làm Parent Chunk (2048 ký tự), chia nhỏ thành các Child Chunk (256 ký tự) để indexing.
2. **Hệ thống Tìm kiếm (Search & Retrieval):**
   - Chuyển đổi sang **Hybrid Search** kết hợp BM25 (đã chuẩn hóa tiếng Việt qua `underthesea`) và Dense Search qua **Qdrant Vector DB** chạy trên Docker.
   - Áp dụng **Reciprocal Rank Fusion (RRF)** với hằng số $k=60$ để hợp nhất kết quả tìm kiếm từ vựng và ngữ nghĩa.
3. **Reranking:**
   - Tích hợp `BAAI/bge-reranker-v2-m3` sau bước retrieval, lấy Top-20 ứng viên từ Hybrid Search và xếp hạng lại để chọn Top-3 có độ tương quan cao nhất chuyển đến LLM.
4. **Enrichment Pipeline:**
   - Áp dụng kỹ thuật **Contextual Prepend** (gắn tiêu đề văn bản và phạm vi điều khoản vào đầu mỗi chunk) bằng cách gọi single-call prompt tiết kiệm chi phí.
5. **Đánh giá & Giám sát liên tục:**
   - Xây dựng bộ test-set nội bộ gồm 50 câu hỏi nghiệp vụ thực tế.
   - Dùng framework **RAGAS** để benchmark định kỳ sau mỗi lần cập nhật kho dữ liệu (mục tiêu: duy trì Faithfulness $\ge 0.85$ và Context Precision $\ge 0.90$).

#### 3. Timeline triển khai cụ thể
- **Tuần 1:** Thiết lập hạ tầng Qdrant Docker, refactor module Ingestion sang Hierarchical Chunking và tích hợp Contextual Prepend.
- **Tuần 2:** Xây dựng module Hybrid Search (BM25 + Dense) và tích hợp Cross-Encoder Reranker.
- **Tuần 3:** Tinh chỉnh System Prompt cho LLM (chống hallucination, format trả lời kèm trích dẫn điều khoản), đo đạc RAGAS baseline.
- **Tuần 4:** Triển khai thử nghiệm cho 20 người dùng nội bộ, thu thập logs câu hỏi khó và tiếp tục tối ưu hóa theo Diagnostic Tree.
