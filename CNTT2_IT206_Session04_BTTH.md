# Bài 3: Bài tập thực hành tổng hợp (Session 04: Prompting Nâng cao, Logic & Chuỗi)

## 1. Bối cảnh bài toán

FastDelivery nhận nhiều thư khiếu nại giao hàng mỗi ngày. Cần một quy trình tự động: phân loại khiếu nại, tính tiền bồi thường, viết email trả lời. Bài này dùng cả 3 kỹ thuật:
- **Chain-of-Thought (CoT):** AI suy nghĩ từng bước khi tính tiền để ít sai.
- **Prompt Chaining:** chia việc thành 3 trạm nối tiếp, kết quả trạm trước là đầu vào trạm sau.
- **Xử lý đứt gãy:** khi AI bị ngắt giữa chừng thì viết tiếp đúng chỗ, và không để phiên chat quá dài.

## 2. Sơ đồ chuỗi prompt

```
Khiếu nại (text)
      ↓
[Trạm 1] Phân loại → JSON         (model nhỏ, rẻ)
      ↓
{Kiểm tra 1} JSON hợp lệ? Loại sự cố có trong danh sách?
      ↓ sai → thử lại 1 lần → vẫn sai: đưa vào "cần người duyệt"
[Trạm 2] Tính bồi thường bằng CoT → [RESULT: số tiền]   (model mạnh)
      ↓
{Kiểm tra 2} Số tiền hợp lệ? (>= 0 và <= giá trị đơn)
      ↓ trên 10 triệu → dừng cho nhân viên CSKH duyệt
[Trạm 3] Viết email (dưới 120 từ)
      ↓
{Kiểm tra 3} Email có bị cắt không? Bị cắt → Continue Prompting
      ↓
Email hoàn chỉnh
```

## 3. Prompt thực thi từng bước

**Trạm 1 (phân loại):**
```
Phân loại khiếu nại. Chỉ trả về JSON thuần, không có chữ nào khác:
{"order_id": "...", "loai_su_co": "giao_tre|hang_hong|mat_hang", "gia_tri_don": số}
Khiếu nại: {noi_dung}
```

**Trạm 2 (CoT):**
```
Dữ liệu: {json_tram1}
Chính sách: {chinh_sach}
Hãy suy nghĩ từng bước: (1) xác định loại sự cố, (2) tìm mức bồi thường
trong chính sách, (3) tính số tiền. Nếu chính sách không nói, ghi "không rõ",
tuyệt đối không đoán. Dòng cuối ghi đúng dạng: [RESULT: số tiền]
```

**Trạm 3 (email):**
```
Bạn là nhân viên CSKH FastDelivery. Viết email xin lỗi dưới 120 từ, lịch sự.
Loại sự cố: {loai}. Số tiền bồi thường: {tien}. Chỉ trả về nội dung email.
```

**Xử lý đứt gãy (Continue Prompting):**
```
Hãy viết tiếp ngay sau cụm từ: "{3-5 từ cuối của đoạn bị cắt}".
Không lặp lại phần đã viết.
```

## 4. Cơ chế bắt lỗi để không crash (code mẫu Python)

```python
import json, re

LOAI_HOP_LE = {"giao_tre", "hang_hong", "mat_hang"}

def tram1(noi_dung):
    for _ in range(2):                      # thử tối đa 2 lần
        out = goi_ai(PROMPT_1.format(noi_dung=noi_dung))
        try:
            d = json.loads(out)             # JSON sai -> nhảy vào except
            if d.get("loai_su_co") in LOAI_HOP_LE and d.get("order_id"):
                return d
        except json.JSONDecodeError:
            pass
    return None                             # vẫn lỗi -> báo cho người duyệt

def tram2(d):
    out = goi_ai(PROMPT_2.format(json_tram1=d, chinh_sach=CHINH_SACH))
    m = re.search(r"\[RESULT:\s*([\d\.]+)\]", out)   # lấy số cuối cùng
    if not m:
        return None
    tien = float(m.group(1))
    return tien if 0 <= tien <= d["gia_tri_don"] else None

def xu_ly(danh_sach):
    xong, can_duyet = [], []
    for kn in danh_sach:
        try:
            d = tram1(kn)
            tien = tram2(d) if d else None
            if tien is None or tien > 10_000_000:
                can_duyet.append(kn)        # dữ liệu lạ hoặc tiền lớn -> người duyệt
                continue
            xong.append(tram3(d, tien))
        except Exception as e:              # lỗi bất ngờ vẫn không làm sập cả hệ thống
            can_duyet.append(kn)
    return xong, can_duyet
```

Cách này đáp ứng 3 ràng buộc:
- **Code sạch:** mỗi trạm là một hàm riêng, dễ đọc và dễ sửa.
- **Tối ưu token:** prompt ngắn, bỏ lời chào; CoT chỉ dùng ở Trạm 2; Trạm 1 dùng model nhỏ, rẻ; mỗi khiếu nại chạy phiên mới và chỉ truyền JSON sạch (giống Session Reset) nên không phải gửi lại lịch sử dài.
- **Bắt lỗi dữ liệu dị biệt:** `try-except`, kiểm tra JSON, kiểm tra danh sách loại sự cố, kiểm tra khoảng số tiền; đơn bất thường đưa vào danh sách người duyệt thay vì làm hệ thống crash.

## 5. Kết quả chạy thử

*(Bạn tự chạy các prompt trên AI với dữ liệu mẫu và thay bảng dưới bằng kết quả thật của mình, kèm ảnh chụp màn hình. Bảng sau chỉ là ví dụ cách trình bày.)*

| Khiếu nại | Trạm 1 (JSON) | Kiểm tra | Kết quả |
| --- | --- | --- | --- |
| Đơn A: giao trễ 4 tiếng, đơn 500.000 đ | Hợp lệ | Số tiền hợp lệ | Email hoàn chỉnh |
| Đơn B: nội dung lộn xộn, thiếu mã đơn | JSON thiếu `order_id` | Không qua, thử lại vẫn lỗi | Vào danh sách "cần người duyệt", hệ thống không crash |
| Đơn C: đòi bồi thường rất lớn | Hợp lệ | Tiền trên 10 triệu | Dừng để nhân viên CSKH duyệt |
| Đơn D: email bị cắt giữa chừng | Hợp lệ | Phát hiện email bị cắt | Dùng Continue Prompting, ghép thành email đủ |

## 6. Nhận xét cuối

- Chia chuỗi giúp biết lỗi ở trạm nào và sửa riêng.
- CoT giúp tính tiền chính xác hơn, nhưng chỉ dùng ở bước cần suy luận để đỡ tốn token.
- Nhược điểm: chậm hơn và tốn nhiều lần gọi AI hơn so với 1 prompt, nên chỉ dùng khi bài toán đủ phức tạp.
