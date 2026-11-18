**Bước 1: Thiết kế giải pháp (I/O)**

Xác định rõ Input của bài toán gồm những gì, và cấu trúc Output mong muốn.
Input:

Nguồn	Nội dung
Tin nhắn khách	Văn bản tự do: sản phẩm, số lượng, địa chỉ, số điện thoại, mã giảm giá
Bảng giá	Danh sách sản phẩm và đơn giá
Chính sách	Mã giảm giá và điều kiện, phí ship, ngưỡng duyệt tay

Output từng trạm (JSON sạch, không lời chào):

Trạm	Output
1	{"status","items":[{"name","qty"}],"address","phone","promo_code","missing_fields","errors"}
2	{"status","reasoning_steps","subtotal","discount","shipping_fee","total","need_human_review"}
3	Tin nhắn phản hồi: xác nhận đơn, xin bổ sung thông tin hoặc báo lỗi

**Bước 2: Viết prompt và chạy thử**

Viết prompt hoàn chỉnh theo định dạng chuyên nghiệp và copy kết quả phản hồi của AI.

**Prompt:**
```
#VAI TRÒ

- Bạn là hệ thống xử lý đơn hàng tự động của cửa hàng. Hãy thực hiện tuần tự 3 TRẠM trong cùng một lần trả lời. Output của trạm trước là Input của trạm sau.

#DỮ LIỆU CỬA HÀNG

- Bảng giá: Sản phẩm A: 100.000đ | Sản phẩm B: 50.000đ | Sản phẩm C: 30.000đ
Chính sách:
    - Mã GIAM10: giảm 10% tiền hàng, chỉ áp dụng khi tiền hàng ≥ 200.000đ.
    - Phí ship 20.000đ; miễn phí nếu tiền hàng sau giảm ≥ 300.000đ.
    - Tổng thanh toán > 5.000.000đ → need_human_review=true.

**TRẠM 1: TRÍCH XUẤT VÀ KIỂM TRA (ưu tiên từ trên xuống)**

Đọc tin nhắn khách, trích xuất items (name, qty), address, phone, promo_code.
1. Sản phẩm không có trong bảng giá → status="ERROR", ghi vào errors, không đoán.
2. Thiếu địa chỉ, số điện thoại hoặc số lượng → status="NEEDS_INFO", liệt kê trong missing_fields.
3. Số điện thoại không đúng 10 chữ số → status="NEEDS_INFO", thêm "phone" vào missing_fields.
4. Số lượng ≤ 0 hoặc > 20 → status="ERROR".
5. Hợp lệ → status="OK".
Không bịa thông tin không có trong tin nhắn; trường không có để "".

**TRẠM 2: TÍNH TIỀN BẰNG CHAIN-OF-THOUGHT**
- Nếu status Trạm 1 khác "OK" → tram_2 = {"status":"STOPPED"}, KHÔNG tính toán.
- Nếu OK, hãy suy luận TỪNG BƯỚC:
  Bước 1: Thành tiền từng dòng (đơn giá × số lượng).
  Bước 2: Cộng thành subtotal.
  Bước 3: Kiểm tra điều kiện mã giảm giá, tính discount (không hợp lệ → discount=0 và ghi lý do).
  Bước 4: Xác định phí ship dựa trên tiền hàng sau giảm.
  Bước 5: total = subtotal − discount + phí ship; kiểm tra lại từng phép tính; so với ngưỡng 5.000.000đ.
- Thiếu giá sản phẩm nào → status="ERROR", không tự điền số.

**TRẠM 3: SOẠN TIN PHẢN HỒI (tối đa 100 từ, giọng thân thiện, lịch sự)**
- Trạm 1 = "NEEDS_INFO": xin khách bổ sung đúng các trường trong missing_fields.
- Trạm 1 = "ERROR": báo sản phẩm/thông tin không hợp lệ, đề nghị chọn lại theo bảng giá.
- Đơn hợp lệ: xác nhận sản phẩm, tiền hàng, giảm giá, phí ship, tổng thanh toán.
- Chỉ dùng số liệu từ Trạm 1 và Trạm 2, không bịa thêm.

#ĐỊNH DẠNG OUTPUT

Chỉ trả về MỘT JSON hợp lệ, không thêm lời chào hay giải thích bên ngoài:
{"tram_1":{"status":"","items":[{"name":"","qty":0}],"address":"","phone":"","promo_code":"","missing_fields":[],"errors":[]},"tram_2":{"status":"","reasoning_steps":[],"subtotal":0,"discount":0,"shipping_fee":0,"total":0,"need_human_review":false},"tram_3":{"reply_message":""}}

#TIN NHẮN KHÁCH

"""Cho mình đặt 2 sản phẩm A và 1 sản phẩm B. Giao đến 25 đường Lê Lợi, Quận 1. SĐT 0901234567. Mình có mã GIAM10."""
```

**Kết quả:**
```
{
  "tram_1": {
    "status": "OK",
    "items": [
      {
        "name": "Sản phẩm A",
        "qty": 2
      },
      {
        "name": "Sản phẩm B",
        "qty": 1
      }
    ],
    "address": "25 đường Lê Lợi, Quận 1",
    "phone": "0901234567",
    "promo_code": "GIAM10",
    "missing_fields": [],
    "errors": []
  },
  "tram_2": {
    "status": "OK",
    "reasoning_steps": [
      "Bước 1: Thành tiền từng dòng: Sản phẩm A (100.000đ x 2 = 200.000đ), Sản phẩm B (50.000đ x 1 = 50.000đ).",
      "Bước 2: Cộng thành subtotal: 200.000 + 50.000 = 250.000đ.",
      "Bước 3: Kiểm tra mã giảm giá GIAM10: Tiền hàng 250.000đ >= 200.000đ, thỏa mãn điều kiện giảm 10% -> discount = 250.000 * 10% = 25.000đ.",
      "Bước 4: Tiền hàng sau giảm = 250.000 - 25.000 = 225.000đ (< 300.000đ) -> Phí ship = 20.000đ.",
      "Bước 5: Tổng thanh toán = 250.000 - 25.000 + 20.000 = 245.000đ (<= 5.000.000đ, need_human_review = false)."
    ],
    "subtotal": 250000,
    "discount": 25000,
    "shipping_fee": 20000,
    "total": 245000,
    "need_human_review": false
  },
  "tram_3": {
    "reply_message": "Cảm ơn bạn đã đặt hàng! Đơn hàng của bạn gồm 2 Sản phẩm A và 1 Sản phẩm B. Tiền hàng: 250.000đ, giảm giá (mã GIAM10): 25.000đ, phí ship: 20.000đ. Tổng thanh toán là 245.000đ. Chúng tôi sẽ giao đến địa chỉ của bạn sớm nhất!"
  }
}
```