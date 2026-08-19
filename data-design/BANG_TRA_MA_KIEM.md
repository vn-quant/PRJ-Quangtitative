# Bảng tra mã phép kiểm C-01 … C-18

Sinh từ `C:\Users\OS\Lucius\Projects\PRJ Quantitative Investment\2. Data design\main\OHLCV.md` §8.
Đây là chỗ tra khi gặp một mã `C-xx` mà không nhớ nó là gì.

Trạng thái cưỡng chế bằng
`C:\Users\OS\Lucius\Projects\PRJ Quantitative Investment\3. Build\test_spec_vs_code.py`
— test đó ĐỎ nếu spec khai một mã mà code không cài, hoặc hoãn mà không nêu lý do.

| Mã | Kiểm cái gì | Bắt lỗi thật nào | Trạng thái |
|---|---|---|---|
| **C-01** | Bar tự mâu thuẫn ⇒ **GHI NHẬN, không xoá cột**. Chỉ bỏ dòng khi `close` không dùng được | Cổng không có trọng tài (trần/sàn nằm ở cửa khác) nên **không quy được trách nhiệm cho cột nào** — hai lần sửa trước đều đổ nhầm cột | ✅ |
| **C-01n** | Vết của bar tự mâu thuẫn. Tra `ops.quarantine` để lấy giá trị gốc | dòng vẫn nằm trong `obs`, không mất gì | ✅ |
| **C-02** | Neo thang giá so `close` phiên trước; nới biên nếu là `ex_date` | CTG 2013 `close=6` · VIX 2014 `close>200.000` | ✅ |
| **C-03** | Nguồn trả rỗng khi đáng lẽ phải có data ⇒ FAIL | scraper nuốt lỗi, 0 dòng nhiều tháng | ⏳ |
| **C-04** | Phân trang: `totalElements` khớp số dòng nhận | `events()` mặc định chỉ lấy 6/30 sự kiện | ⏳ |
| **C-05** | `REVOKE UPDATE, DELETE` — Postgres cấm sửa dữ liệu cũ | job đêm xoá backfill 5 đêm liền | ✅ |
| **C-06** | `CHECK price_unit` · `CHECK price_basis` | lẫn thang ×1000 · trộn raw/adj | ✅ |
| **C-07** | `UNIQUE` trên khoá + `payload_hash` | ghi trùng khi chạy lại | ✅ |
| **C-08** | Đủ mã: số mã hỏi được = số mã kỳ vọng | `get_vn100()` tụt 100→30, chỉ ghi `log.warning` | ✅ |
| **C-09** | Đủ ngày: so với **lịch giao dịch của sàn** | máy tắt một hôm, thủng âm thầm mãi mãi | ✅ |
| **C-10** | `observed_at ≥ trade_date` | lệch múi giờ ICT/UTC | ✅ |
| **C-11** | `px_adj(d, as_of=d) == px_raw(d)` | **sai chiều công thức lộ ngay** · chuỗi adj gãy vì không nạp lại sau sự kiện | ✅ |
| **C-12** | Nhảy giá >50%/ngày mà không có sự kiện | 70 bậc gãy chưa giải thích được | ⏳ |
| **C-13** | `SSI_raw × Π hệ số` vs `DNSE_adj` ngoài ex-date | trộn raw/adj · thiếu sự kiện | ⏳ |
| **C-14** | Hai lần quan sát khác nhau phải dựng ra **cùng** một giá thô | thiếu sự kiện trong khoảng giữa | ⏳ |
| **C-15** | **Hệ số ta tính vs hệ số SÀN công bố** (giải từ trần/sàn) | sai loại sự kiện · sai tỷ lệ · **thiếu sự kiện cùng ngày** · điều chỉnh nhầm ESOP | ✅ |
| **C-16** | >1 bar cho cùng `(nguồn, mã, ngày)` | lỗi feed DNSE 2022-12-27: 93/100 mã có bar thừa | ✅ |
| **C-17** | Tỷ lệ DNSE vs VCI, so **tập với tập** (cấm `.first()`) | 12/575 ca lệch thật; cài sai cho **116 báo động giả** | ⏳ |
| **C-18** | Dải hợp lý cho mọi số parse ra | 6 ca cổ tức sai ×10 · 30 ca sai ×100 · 5 ca nguồn mất dấu thập phân | ✅ |
| **C-19** | **CẮT NGANG THEO NGÀY** — tỉ lệ mã hỏng vượt nền cục bộ ≥20 điểm ⇒ nghi NGUỒN hỏng cả ngày | 2009-09-16: 38/53 mã cùng hỏng (71,7% vs nền 3,8%) · 2021-04-16: 97/185 · 2021-05-10: 73/184. **Mọi cổng khác chỉ nhìn một dòng nên mù hẳn lớp lỗi này** | ✅ |

**✅ đã cài 13 · ⏳ hoãn 6** (C-03, C-04, C-12, C-13, C-14, C-17 — mỗi cái có điều kiện gỡ ghi trong
`test_spec_vs_code.py`).

### Vì sao C-19 phải so với NỀN chứ không dùng ngưỡng tuyệt đối

Chất lượng nền không đều theo thời gian: 2008–2009 ngày nào cũng 5–11% bar hỏng, còn từ 2021
nền là 0%. Ngưỡng tuyệt đối "≥10% ⇒ ĐỎ" cho ra **422 ngày** — vừa bắt oan cả giai đoạn đầu vừa
bỏ lọt ngày hỏng ở giai đoạn sạch. So với trung vị 61 phiên quanh nó thì còn **13 ngày**, vừa đủ
để đi xử lý tay.

---

## Phép kiểm nào mạnh nhất

**C-15.** Sàn công bố `Ceiling` và `Floor` mỗi phiên, và `ref_sàn = (trần + sàn) / 2` — đó là
**đáp án**, thứ duy nhất trong toàn hệ không do ta tính ra. Nó bắt được bốn lớp lỗi hệ số cùng lúc.

**C-11** đứng thứ hai: nó so hai thang giá độc lập, nên bất kỳ sai lệch nào của công thức điều chỉnh
đều lộ.

Mọi phép kiểm còn lại chỉ soi **tính nhất quán nội tại** — mà hệ thống hoàn toàn có thể
*nhất quán sai*, đã xảy ra ba lần trong dự án này.

---

## Ba tầng

| Tầng | Khi nào chạy | Mã |
|---|---|---|
| **A** — lúc bắt dữ liệu | trong vòng lặp fetch | C-01 · C-02 · C-03 · C-04 |
| **B** — lúc ghi, Postgres cưỡng chế | mỗi lần INSERT | C-05 · C-06 · C-07 |
| **C** — sau khi chạy, hằng ngày | cuối job | C-08 · C-09 · C-10 · C-11 · C-12 |
| **D** — đối chiếu nguồn độc lập | định kỳ | C-13 · C-14 · C-15 · C-16 · C-17 · C-18 |
