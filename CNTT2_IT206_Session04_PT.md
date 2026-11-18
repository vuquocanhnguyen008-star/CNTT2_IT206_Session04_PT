# Bài 2: So sánh Trade-off các giải pháp ra lệnh cho Prompting Nâng cao (Logic & Chuỗi)

## Bài toán chọn

FastDelivery cần xử lý thư khiếu nại giao hàng trễ: phân loại sự cố, tính tiền bồi thường, rồi viết email trả lời khách. Đây là bài toán dùng cả Chain-of-Thought, Prompt Chaining và xử lý đứt gãy.

## Bước 1: Đề xuất 2 prompt mẫu

### Giải pháp 1 (đơn giản, ngắn gọn): 1 prompt duy nhất

```
Bạn là nhân viên CSKH của FastDelivery. Đọc khiếu nại dưới đây, tính tiền
bồi thường và viết email trả lời khách.
Khiếu nại: [dán nội dung khiếu nại]
```

### Giải pháp 2 (chi tiết, nhiều ràng buộc): chuỗi 3 prompt + CoT + kiểm tra

**Prompt 1 (Trạm 1, phân loại):**
```
Phân loại khiếu nại sau. Chỉ trả về JSON thuần, không giải thích:
{"order_id": "...", "loai_su_co": "giao_tre|hang_hong|mat_hang", "gia_tri_don": số}
Khiếu nại: [dán nội dung]
```

**Prompt 2 (Trạm 2, tính bồi thường bằng CoT):**
```
Dữ liệu: [JSON của Trạm 1]. Chính sách: [dán đoạn chính sách].
Hãy suy nghĩ từng bước: (1) xác định loại sự cố, (2) tìm mức bồi thường
trong chính sách, (3) tính số tiền. Nếu chính sách không có, ghi "không rõ".
Dòng cuối ghi đúng dạng: [RESULT: số tiền]
```

**Prompt 3 (Trạm 3, viết email):**
```
Viết email xin lỗi khách, dưới 120 từ, giọng lịch sự. Dùng dữ liệu:
loại sự cố, số tiền bồi thường: [dữ liệu sạch]. Chỉ trả về nội dung email.
```

## Bước 2: Bảng so sánh Trade-off

| Tiêu chí | Giải pháp 1 (đơn giản) | Giải pháp 2 (chi tiết) |
| --- | --- | --- |
| Độ dễ hiểu | Cao (chỉ 1 prompt) | Trung bình (3 trạm, phải hiểu luồng) |
| Tốn token | Ít | Nhiều hơn (nhiều lần gọi, thêm CoT) |
| Chặn lỗi biên | Trung bình (gần như không có) | Cực tốt (có kiểm tra giữa các trạm) |
| Độ chính xác tiền bồi thường | Thấp, AI dễ tính sai | Cao nhờ CoT và đối chiếu chính sách |
| Độ trễ | Nhanh (1 lần gọi) | Chậm hơn (cộng dồn các trạm) |
| Dễ sửa lỗi | Khó, không biết sai ở đâu | Dễ, sai trạm nào sửa trạm đó |
| Dễ bị lỗi lan truyền | Không có chuỗi nên ít lan | Có nguy cơ nếu thiếu chốt chặn |

## Bước 3: Lựa chọn và chạy thử

**Chọn Giải pháp 2.** Lý do:
- Bài toán có tính tiền nên sai là mất tiền, cần độ chính xác cao hơn tốc độ.
- Có chốt chặn kiểm tra nên dữ liệu lạ không làm hỏng cả hệ thống.
- Chỉ dùng CoT ở bước tính tiền nên không tốn token vô ích ở các bước khác.
- Nhược điểm (chậm, tốn token) chấp nhận được vì khiếu nại không cần trả lời trong 1–2 giây.

**Kết quả chạy thử:** *(bạn tự chạy 2 giải pháp trên AI và dán kết quả thật vào đây, kèm ảnh chụp màn hình)*

| | Giải pháp 1 | Giải pháp 2 |
| --- | --- | --- |
| Số tiền bồi thường AI đưa ra | *(điền)* | *(điền)* |
| Đúng chính sách không? | *(điền)* | *(điền)* |
| Email có đủ ý, không dài dòng? | *(điền)* | *(điền)* |

**Nhận xét:** *(viết 2–3 câu theo kết quả thật của bạn, ví dụ Giải pháp 1 có tính sai số tiền hay không)*
