# Nguồn DNSE — trường khả dụng

> Lập lại 2026-08-19 **từ nguồn v2**: `INGEST_SPEC.md` (D2) + code thật trong
> `3. Build\code\` + schema. Cột **v2 ghi vào đâu** đọc từ code, không chép từ đặc tả.
>
> **Đã gọi xác minh trực tiếp 2026-08-19** (ACB, 90 ngày): 7 khoá đúng như mô tả · đơn vị
> **kVND** xác nhận (`c` cuối = `21.8`) · **0 ngày cuối tuần** trong chuỗi ⇒ bỏ ngày không
> giao dịch, xác nhận · ba biến thể tham số raw (`adjusted=false` · `raw=true` ·
> `adjust=0`) đều trả **mảng `c` giống hệt** bản gốc ⇒ E0017 vẫn đúng.

**Bản chất: chiều sâu lịch sử, không phải chiều rộng trường.** Nghèo trường nhất trong ba
nguồn API — 7 khoá — nhưng là nơi duy nhất với tới dải 2012–2020 mà SSI không có.
Không cần token.

## Phiên bản API

| Bề mặt | Host | Version trong đường dẫn | Auth |
|---|---|---|---|
| Chart | `services.entrade.com.vn` | **`/chart-api/v2/`** | không |
| Sự kiện | `api-bo.dnse.com.vn` | **không có đoạn version** (`/senses-api/`) | không |
| Auth | `api.dnse.com.vn/auth-service/login` | — | không cần dùng |

Bề mặt chart có `v2` nên đổi bản sẽ thấy được ở URL. Bề mặt `senses-api` **không đánh
version** — cùng loại rủi ro như iBoard của SSI: đổi hành vi mà không đổi đường dẫn.

---

# Cửa nào v2 thực sự gọi

| Cửa | Script v2 | Vai trò |
|---|---|---|
| `chart-api/v2/ohlcs/stock` | `daily_update.py` | **nguồn giá của job hằng ngày** |
| `chart-api/v2/ohlcs/stock` | `kiem_iboard.py` · `probe_adj_3nguon.py` · `sweep_dnse_vs_vci.py` | đối chứng / thăm dò |
| `senses-api/events` | `daily_events.py` | sự kiện DN, **một trong ba nguồn** gộp lại |

**Chưa dùng:** `ohlcs/index` · `ohlcs/derivative`.

> Nguồn CHÍNH của `daily_update.py` **không hardcode** — đọc từ
> `meta.source_contract.vai_tro`. Script sẽ dừng nếu chưa khai nguồn chính. Nên "DNSE là
> nguồn chính" là **trạng thái cấu hình trong DB**, không phải trong code.

---

# 1. `chart-api/v2/ohlcs/stock` — 7 khoá

| | |
|---|---|
| Endpoint | `GET https://services.entrade.com.vn/chart-api/v2/ohlcs/{stock\|index\|derivative}` |
| Tham số | `symbol` · `resolution` · `from` `to` (**unix giây**) |
| Đường theo loại | `/stock` cổ phiếu + ETF · `/index` VNINDEX, VN30 · `/derivative` VN30F1M |
| Auth | **không cần** |
| Đơn vị giá | 🔴 **NGHÌN VND** (`c=22.5` = 22.500đ) — phải khai `price_unit='kVND'` |
| Cơ sở giá | **ADJUSTED**, **không có tham số bật raw** |
| Ngày không giao dịch | **BỎ HẲN, không trả dòng** — ngược với SSI |
| Nhịp | **1 lần** nạp lịch sử, sau đó chỉ append bar hôm nay |
| Chất lượng đo được | 0 null · 0 vi phạm logic OHLC / 2.971 dòng · 0 timeout / 261 mã |

**Phạm vi theo khung — retention phân tầng:**

| Khung | Sâu từ |
|---|---|
| `1D` | **2012-03-20** (trần nền tảng) |
| `1W` | 2021-02 |
| `1H` | 2023-07 |
| `1m` | ~3 tháng |

Phản hồi là **dict các mảng song song**, không phải danh sách bản ghi (mẫu ACB):

| Khoá | Mẫu | Ý nghĩa | v2 ghi vào đâu |
|---|---|---|---|
| `t` | `[1784772000, 1784858400, …]` | unix giây, đầu mỗi bar | `trade_date` |
| `o` | `[22.4, …]` | mở cửa, **kVND**, đã điều chỉnh | cột `open` |
| `h` | `[22.6, …]` | cao nhất | cột `high` |
| `l` | `[22.1, …]` | thấp nhất | cột `low` |
| `c` | `[22.5, …]` | đóng cửa | cột `close` |
| `v` | `[27165000, 13447000, …]` | khối lượng (cổ phiếu) | cột `volume` |
| `nextTime` | `0` | con trỏ phân trang; `0` = hết | — |

**Dùng 6/7 khoá.** Đây là nguồn gần như không có gì thừa ở mức trường — thừa nằm ở mức
đường dẫn: `/index` và `/derivative` chưa ai gọi.

`daily_update.py` không tự quy đổi đơn vị trong code — nó đọc `price_unit` và `price_basis`
từ `meta.source_contract` cho đúng `source` đang chạy. Đơn vị là **khai báo trong DB**, không
phải hằng số trong script.

### Đối chứng chéo đã chạy

`v` của ACB ngày 24/07 = `13.447.000` **khớp đúng** `TotalMatchVol` của SSI. Hai nguồn độc
lập cho cùng khối lượng ⇒ tin được con số khối lượng.

---

# 2. `senses-api/events` — sự kiện doanh nghiệp

| | |
|---|---|
| Endpoint | `GET https://api-bo.dnse.com.vn/senses-api/events` |
| Auth | không |
| Sâu từ | **2007-03-06** |
| v2 dùng | `daily_events.py`, hàm `lay_dnse()` |

**Không phải nguồn sự kiện duy nhất.** `daily_events.py` gộp **ba nguồn**: DNSE ·
VCI IQ `/v1/events` · Vietstock `EventsTypeData`. Phân loại và bóc tỷ lệ do `ca_parse.py`
làm (`classify`, `parse`, `classify_vietstock`, `parse_vietstock`, `c18_ngoai_dai`).

Cửa này **chưa được lập tài liệu ở mức trường** — đặc tả v2 không có bảng trường cho nó, và
mục "DNSE cho được loại sự kiện nào" vẫn đang treo.

---

# Cạm bẫy đã đo

- **Đơn vị kVND là bẫy nguy hiểm nhất của nguồn này.** Đo 2026-08-02: một truy vấn lấy neo
  giá **không lọc `source`** đã so neo `22.600 VND` (từ `SSI_SP`) với giá `22,6 kVND` (DNSE)
  ⇒ tỷ lệ `0,001` ⇒ **2.610 dòng ĐÚNG bị chặn oan**. Kho `obs` chứa nhiều nguồn với đơn vị
  khác nhau, nên **mọi truy vấn giá phải lọc `source`**.
- **Ngưỡng 0,2% khi so giá.** Dưới mức đó là sai số làm tròn 2 chữ số kVND, **không phải**
  điều chỉnh giá. Đặt ngưỡng chặt hơn sẽ báo động giả.
- **Không có đường bật raw.** Đã thử `adjusted=false` · `adjust=0` · `raw=true` ·
  `unadjusted=true` — cả 4 bị bỏ qua, trả kết quả giống hệt (E0017). Không dùng DNSE làm
  nguồn giá thô trong bất kỳ hoàn cảnh nào.
- **`from == to` trả về MẢNG RỖNG, không phải lỗi.** Đo 2026-08-19: HTTP `200`, thân
  `{"t":[],"o":[],"h":[],"l":[],"c":[],"v":[],"nextTime":…}`. Đặc tả cũ ghi "từ chối" là
  chưa chính xác — nó im lặng trả rỗng, dễ bị hiểu nhầm thành "mã không có dữ liệu". Phải
  hỏi theo cửa sổ rồi lọc.
- **Bỏ hẳn ngày không giao dịch.** Ngược với SSI (trả dòng `volume=0`). Đếm số dòng để suy
  ra số phiên sẽ cho hai kết quả khác nhau giữa hai nguồn.

---

# Ánh xạ sang schema

| Cửa | Bảng L0 | Khoá |
|---|---|---|
| `chart-api/v2/ohlcs/stock` | `obs.price_daily` (`source='DNSE'`) | `(source, symbol, trade_date, observed_at)` |
| `senses-api/events` | `obs.corp_action` | `(source, symbol, ex_date, event_code, observed_at)` |

`source_contract` phải khai `price_unit='kVND'` **và** `price_basis='adjusted'`. Thiếu khai
đơn vị là lỗi im lặng: số vẫn vào bảng, chỉ sai 1000 lần.

DNSE **không** được đưa vào `market.px_raw` — nó chỉ nuôi phần adjusted.

Mọi dòng phải đi qua đúng hai cổng của job hằng ngày (C-01, C-02). Nạp lịch sử mà bỏ cổng
thì kho có hai chuẩn, đúng bệnh của v1.

---

# Vai trò trong hệ

Phục vụ 6/60 mục. Độc quyền đúng hai thứ: **lịch sử adjusted 2012–2020** và **khung 1H sâu
3 năm**. Ngoài hai thứ đó, mọi mục DNSE cho được thì SSI hoặc VCI đều cho tốt hơn.

Đây là lý do kỹ thuật khiến **round-robin giữa ba nguồn sai từ gốc** — không có hai nguồn
nào đủ giống nhau để thay phiên nhau.

# Còn treo

- DNSE cho được **loại sự kiện nào**, và trường của `senses-api/events` gồm những gì.
- **Rate limit thật** — chưa đo.
- Độ sâu của `/derivative`.
