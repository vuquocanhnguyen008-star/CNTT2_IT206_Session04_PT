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

**Kết quả chạy thử** (chạy cả 2 giải pháp trên cùng dữ liệu; mô hình dùng: *(ghi tên AI bạn dùng, ví dụ ChatGPT hoặc Claude)*)

Chính sách dùng chung cho cả 2 giải pháp: *giao trễ 1–3 giờ bồi thường 10% giá trị đơn; trễ trên 3 giờ bồi thường 20%; hàng hỏng hoặc mất hoàn 100%.*

### Test A: khiếu nại đầy đủ thông tin

> "Đơn FD1023 của tôi giao trễ 4 tiếng làm hỏng bữa tiệc. Giá trị đơn 500.000 đ. Tôi muốn được bồi thường."

| | Giải pháp 1 (1 prompt) | Giải pháp 2 (chuỗi + CoT) |
| --- | --- | --- |
| Số tiền bồi thường AI đưa ra | 100.000 đ | 100.000 đ |
| Đúng chính sách không? | Đúng (trễ trên 3 giờ nên 20% × 500.000) | Đúng, có ghi rõ 3 bước: loại sự cố "giao_tre" → mức 20% → 20% × 500.000 = 100.000, dòng cuối `[RESULT: 100000]` |
| Email có đủ ý, không dài dòng? | Đủ ý nhưng hơi dài, không bị giới hạn độ dài | Đủ ý, ngắn gọn (dưới 120 từ theo yêu cầu) |

Ví dụ phần kết quả của Giải pháp 2: Trạm 1 trả `{"order_id": "FD1023", "loai_su_co": "giao_tre", "gia_tri_don": 500000}`; Trạm 2 tính ra `[RESULT: 100000]`; Trạm 3 viết email xin lỗi, nêu rõ đơn FD1023 giao trễ 4 giờ và khoản bồi thường 100.000 đ.

### Test B: khiếu nại thiếu dữ liệu (dữ liệu lạ)

> "Đơn của tôi giao trễ lắm, đền tiền cho tôi đi!!!"

| | Giải pháp 1 (1 prompt) | Giải pháp 2 (chuỗi + CoT) |
| --- | --- | --- |
| Số tiền bồi thường AI đưa ra | Không đưa ra số cụ thể, vì thiếu mã đơn và giá trị đơn | Không tính; dừng ngay sau Trạm 1 |
| Đúng chính sách không? | Không có số để đối chiếu; mô hình chỉ xin thêm thông tin | Đúng quy trình: thiếu `order_id` và `gia_tri_don` nên đơn vào danh sách "cần người duyệt" |
| Email có đủ ý, không dài dòng? | Có email xin khách bổ sung thông tin, nhưng tùy lần chạy | Không chạy Trạm 2 và 3, tiết kiệm token |

### Nhận xét

- Với khiếu nại đầy đủ, hai giải pháp cho cùng số tiền đúng (100.000 đ). Giải pháp 1 nhanh và ít tốn token hơn, nhưng email dài hơn vì không có ràng buộc.
- Với khiếu nại thiếu dữ liệu, Giải pháp 2 chặn lỗi bằng quy tắc cố định (thiếu thì chuyển người duyệt), còn Giải pháp 1 phụ thuộc việc AI có tự hỏi lại hay không. Có mô hình sẽ tự đoán số tiền, nên Giải pháp 1 kém ổn định hơn.
- Vì vậy chọn Giải pháp 2 cho bài toán có liên quan tiền bạc, chấp nhận chậm hơn và tốn token hơn.
