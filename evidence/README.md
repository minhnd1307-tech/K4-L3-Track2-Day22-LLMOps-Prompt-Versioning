# Báo Cáo Đánh Giá & Bằng Chứng Thực Nghiệm — Day 22: LLMOps Prompt Versioning

Học viên: **Nguyễn Đức Minh**  
Dự án: **LangSmith Tracing, Prompt Hub Versioning, RAGAS Evaluation, and Guardrails AI**

---

## 1. Danh mục các tệp bằng chứng (Evidence Inventory)

Thư mục `evidence/` chứa đầy đủ 7 tệp bằng chứng bắt buộc theo yêu cầu của bài lab:

| STT | Tên tệp | Mô tả | Trạng thái |
|:---:|---|---|:---:|
| 1 | `01_langsmith_traces.png` | Ảnh chụp giao diện LangSmith hiển thị 50 traces của pipeline RAG (`rag-query`) | ✅ Đầy đủ |
| 2 | `02_prompt_hub.png` | Ảnh chụp 2 phiên bản prompt (`minh-day22-rag-v1` và `minh-day22-rag-v2`) trên Prompt Hub | ✅ Đầy đủ |
| 3 | `02_ab_routing_log.txt` | Log thực thi A/B routing 50 câu hỏi tất định theo MD5 hash (pull từ Hub) | ✅ Đầy đủ |
| 4 | `03_ragas_scores.png` | Ảnh chụp kết quả đánh giá RAGAS cho cả V1 và V2 | ✅ Đầy đủ |
| 5 | `03_ragas_report.json` | Bản sao kết quả đánh giá RAGAS dưới dạng JSON chuẩn | ✅ Đầy đủ |
| 6 | `04_pii_demo_log.txt` | Log kiểm thử PIIDetector với 6 test cases (Email, Phone, SSN, Credit Card, Multi, Clean) | ✅ Đầy đủ |
| 7 | `04_json_demo_log.txt` | Log kiểm thử JSONFormatter với 5 test cases (hợp lệ, fences, nháy đơn, dấu phẩy thừa, invalid) | ✅ Đầy đủ |

---

## 2. Kết quả đánh giá định lượng RAGAS (V1 vs V2)

Toàn bộ 50 cặp QA chuẩn đã được đánh giá qua cả hai phiên bản prompt với 4 chỉ số RAGAS chuẩn công nghiệp:

| Chỉ số (Metric) | Phiên bản V1 (Ngắn gọn) | Phiên bản V2 (Có cấu trúc) | Phiên bản chiến thắng |
|---|:---:|:---:|:---:|
| **Faithfulness** | **0.9503** ⭐ | 0.8895 | **← V1 (+0.0608)** |
| **Answer Relevancy** | **0.9192** | 0.8922 | **← V1 (+0.0270)** |
| **Context Recall** | **1.0000** | **1.0000** | **Hòa (Tuyệt đối 100%)** |
| **Context Precision** | 0.9383 | **0.9417** | **← V2 (+0.0034)** |

- **Mục tiêu đề bài:** Faithfulness ≥ 0.8 ở ít nhất một phiên bản → **ĐẠT** (V1 đạt 0.9503, V2 đạt 0.8895).

---

## 3. Phân tích chuyên sâu: So sánh hiệu quả giữa Prompt V1 và V2

### 3.1. Phân tích chỉ số Faithfulness (Độ trung thực với ngữ cảnh)
- **V1 đạt 0.9503 so với V2 đạt 0.8895:**
  - **Prompt V1** chỉ thị: *"Trả lời trực tiếp và ngắn gọn trong 2–4 câu, chỉ dựa trên context được cung cấp"*. Yêu cầu này thúc đẩy mô hình tập trung trích xuất chính xác các sự kiện cốt lõi từ context mà không mở rộng thêm. Do không cần phải viết dài, xác suất mô hình tự ý suy diễn hoặc chèn thêm kiến thức ngoài văn bản được giảm thiểu tối đa, dẫn đến độ trung thực cực kỳ cao (0.9503).
  - **Prompt V2** yêu cầu: *"Đọc context và chọn những thông tin liên quan... Trả lời trong 3–5 câu theo cấu trúc: ý chính, giải thích dựa trên tài liệu, rồi kết luận"*. Để đáp ứng cấu trúc phức tạp này (đặc biệt là phần "giải thích" và "kết luận"), LLM có xu hướng sử dụng thêm các từ nối ngữ nghĩa, diễn đạt lại theo cách suy luận logic (deductive reasoning). Quá trình suy diễn này đôi khi làm câu trả lời vượt nhẹ khỏi phạm vi câu chữ nguyên bản của context, khiến điểm Faithfulness của V2 giảm nhẹ (0.8895), tuy nhiên vẫn vượt xa ngưỡng yêu cầu 0.80.

### 3.2. Phân tích chỉ số Answer Relevancy (Độ phù hợp của câu trả lời)
- **V1 đạt 0.9192 so với V2 đạt 0.8922:**
  - Do câu hỏi của người dùng thường mang tính chất truy vấn thông tin trực diện, phong cách trả lời ngắn gọn, thẳng vào trọng tâm của V1 mang lại độ tương đồng ngữ nghĩa cao hơn giữa câu hỏi và câu trả lời.
  - V2 với cấu trúc 3 phần (ý chính, giải thích, kết luận) đôi khi chứa các câu mở đầu hoặc diễn giải bối cảnh chung, làm loãng mật độ thông tin trả lời trực tiếp cho câu hỏi, dẫn đến điểm relevancy thấp hơn nhẹ.

### 3.3. Phân tích chỉ số Context Precision & Context Recall
- **Context Recall đạt 1.0000 ở cả hai phiên bản:**
  - FAISS VectorStore được cấu hình với chunk size = 500, overlap = 50 và `k = 3` trích xuất đầy đủ 100% các đoạn tài liệu cần thiết để giải quyết toàn bộ 50 câu hỏi kiểm thử.
- **Context Precision của V2 nhỉnh hơn (0.9417 vs 0.9383):**
  - Prompt V2 yêu cầu *"chọn những thông tin liên quan đến câu hỏi"* và phân tích ý chính trước khi trả lời, giúp mô hình tập trung khai thác các chunk có độ liên quan cao nhất ở đầu danh sách tài liệu retrieved.

---

## 4. Tổng kết triển khai Guardrails AI

- **PIIDetector:** Triển khai kế thừa từ `Validator` với decorator `@register_validator`. Sử dụng regex nhận diện 4 nhóm thông tin cá nhân: Email, Phone, SSN, Credit Card. Cơ chế sửa lỗi sử dụng `FailResult(fix_value=redacted_text)` kết hợp với `on_fail=OnFailAction.FIX` trong constructor, đảm bảo text nhạy cảm được che an toàn thành `[<TYPE>_REDACTED]`.
- **JSONFormatter:** Hỗ trợ kiểm tra cú pháp JSON, tự động xử lý:
  1. Gỡ bỏ Markdown fences (```json ... ```).
  2. Chuyển đổi nháy đơn `'` thành nháy kép `"`.
  3. Xóa dấu phẩy thừa trước ngoặc đóng (trailing commas).
  4. Trả về cấu trúc JSON fallback an toàn khi dữ liệu đầu vào hoàn toàn không thể sửa được.
