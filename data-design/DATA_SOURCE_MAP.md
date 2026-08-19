# Nguồn nào cho được dữ liệu nào

> **Lập 2026-07-30 (S16).** Tài liệu **2/3** của bộ redesign DB.
> (1) cần lấy gì (`DATA_REQUIREMENTS.md`) · (2) nguồn nào cho được ← *file này* · (3) sơ đồ bảng.
>
> Số hiệu mục (1.1, 3.6…) khớp đúng `DATA_REQUIREMENTS.md`.
> Chi tiết từng trường của mỗi endpoint: `C:/Users/OS/Lucius/Projects/PRJ quant/06_document/DATA_SOURCES.md`.

## Luật: KHÔNG dùng vnstock

vnstock là **thư viện bọc, không có dữ liệu riêng** — nó gọi lại chính VCI/MSN/KBS rồi
cắt bớt trường (`_Price.history(source="VCI")` trả 6 trường trong khi nguồn có nhiều hơn;
với SSI pipeline đọc 6/31 trường). Thêm một tầng phái sinh giữa ta và nguồn nghĩa là:
thêm một chỗ có thể lặng lẽ đổi hành vi (31/08/2025 nó bỏ `Vnstock().stock()` → `save_financials`
ghi **0 dòng im lặng**), và `4.0.5` đã ra mà **không có changelog**.

⇒ **Mọi dòng dưới đây gọi thẳng nguồn.** Đã kiểm 2026-07-30: bỏ vnstock **không mất
khả năng nào** — VCI IQ chạy đầy đủ bằng `requests` thuần (E0016), kết quả trùng khớp
với đường qua vnstock (FPT: 79 sự kiện DIV+ISS ở cả hai đường).

---

## 1. Bốn bề mặt API trực tiếp + nguồn ngoài

| Ký hiệu | Bề mặt | Auth | Bản chất |
|---|---|---|---|
| **SSI** | `fc-data.ssi.com.vn/api/v2/Market/*` — 9 endpoint | token | Nguồn **RAW** duy nhất. 31 trường/dòng. 30 ngày/lời gọi, ~1 lời gọi/giây. Sâu tới **2020-02-26** (mốc cố định, E0012) |
| **DNSE** | `services.entrade.com.vn/chart-api/v2/ohlcs/{stock\|index\|derivative}` | **không** | Chỉ 7 trường (`t o h l c v nextTime`). Đơn vị **nghìn VND**. **ADJUSTED, không có tham số bật raw** (E0017). 1D sâu 2012-03-20, 1 lời gọi/mã |
| **VCI IQ** | `iq.vietcap.com.vn/api/iq-insight-service/*` | **không** (chỉ cần UA + Referer) | Sự kiện DN, BCTC, cổ đông, ngành. **Chưa đo thấy khuyết tật nào** |
| **VCI trading** | `trading.vietcap.com.vn/api/*` | không | Bề mặt **giá** — đã đo hỏng: 8/8 dòng `high=low=close`, 135 timeout/261 mã, lệch ngày khi hỏi 1 ngày |
| *ngoài* | Yahoo · Vietcombank · SBV · Vietstock | — | Hàng hoá, tỷ giá, lãi suất, báo cáo CTCK |

> **VCI có HAI bề mặt, chất lượng khác hẳn — không được gộp.** Bỏ bề mặt giá, giữ IQ.

---

## 2. Bảng map — 60 mục dữ liệu × nguồn

✅ có · ⚠️ có nhưng hạn chế · — không có · ❓ chưa xác minh

### Nhóm 1 — Giá & giao dịch

| # | Dữ liệu | SSI | DNSE | VCI IQ | VCI trading | Ngoài | **Đề xuất** |
|---|---|:--:|:--:|:--:|:--:|:--:|---|
| 1.1 | OHLCV ngày — **giá THÔ** | ✅ `ClosePrice` | — | — | — | — | **SSI** (duy nhất) |
| 1.2 | OHLCV ngày — đã điều chỉnh | ✅ `ClosePriceAdjusted` | ✅ | — | ⚠️ | — | **DNSE** (0 timeout, 14 năm, 1 lời gọi) |
| 1.3 | Intraday 1 phút | ✅ RAW, 2020-02-26+ | ⚠️ ~3 tháng | — | ⚠️ ~2,5 năm | — | **SSI** |
| 1.4 | Khung 5m/15m/30m/1H | — | ✅ 1H sâu 3 năm | — | — | — | **dẫn xuất từ 1m**; DNSE cho 1H |
| 1.5 | Khung tuần / tháng | — | ✅ `1W` | — | — | — | **dẫn xuất** |
| 1.6 | Giá trị giao dịch (VND) | ✅ `TotalMatchVal` | — | — | ✅ `accumulatedValue` | — | **SSI** |
| 1.7 | Trần / sàn / tham chiếu | ✅ `Ceiling/Floor/RefPrice` | — | — | — | — | **SSI** (duy nhất) |
| 1.8 | Giá bình quân phiên | ✅ `AveragePrice` | — | — | — | — | **SSI** (duy nhất) |
| 1.9 | KL & giá trị thoả thuận | ✅ `TotalDealVol/Val` | — | — | — | — | **SSI** (duy nhất) |
| 1.10 | Số lệnh mua / bán | ⚠️ `TotalBuy/SellTrade` (=0 với ACB) | — | — | — | — | SSI — ❓ độ phủ |
| 1.11 | Tick + chiều Mua/Bán | ❓ stream | ❓ MQTT | — | — | — | ❓ chưa kiểm phiên này |
| 1.12 | Sổ lệnh L2 | ❓ stream | ❓ MQTT | — | — | — | ❓ nt |

### Nhóm 2 — Dòng tiền

| # | Dữ liệu | SSI | DNSE | VCI IQ | Ngoài | **Đề xuất** |
|---|---|:--:|:--:|:--:|:--:|---|
| 2.1 | Khối ngoại mua/bán (KL + giá trị) | ✅ **cùng lời gọi giá** | — | — | — | **SSI** — miễn phí, không cần lời gọi riêng |
| 2.2 | Room ngoại | ✅ `ForeignCurrentRoom` | — | — | — | **SSI** |
| 2.3 | Tự doanh CTCK | — | — | — | Vietstock (scrape) | ngoài — không API nào có |
| 2.4 | Giao dịch cổ đông lớn & nội bộ | — | — | ✅ `/v1/company/{sym}/insider-transaction` · 273 bản ghi FPT | — | **VCI IQ** ← *endpoint mới, chưa ai dùng* |

### Nhóm 3 — Sự kiện doanh nghiệp

**Cả SSI và DNSE đều KHÔNG có corporate action.** VCI IQ `/v1/events` là nguồn duy nhất
trong ba nguồn — sâu tới **2010**, liên tục mỗi năm (E0013).

| # | Dữ liệu | Nguồn | Cách lấy |
|---|---|---|---|
| 3.1 | Ex-date · ngày chốt | **VCI IQ** | `exrightDate` · `recordDate` |
| 3.2 | Cổ tức tiền mặt | **VCI IQ** | `valuePerShare`, `eventCode=DIV` |
| 3.3 | Cổ tức CP / CP thưởng | **VCI IQ** | `exerciseRatio` + phân loại theo `eventTitleVi` |
| 3.4 | Chia tách / gộp | **VCI IQ** | nt |
| 3.5 | Quyền mua — tỷ lệ | **VCI IQ** | nt, `eventCode=ISS` |
| 3.6 | **Giá phát hành RIGHTS** | 🔴 **KHÔNG nguồn nào** | Phải suy ngược từ thị trường, hoặc nhập tay từ CBTT/HOSE |
| 3.7 | Phát hành riêng lẻ / ESOP | ⚠️ VCI IQ một phần | `eventCode=ISS` |
| 3.8 | Niêm yết / huỷ / đình chỉ / chuyển sàn | **VCI IQ** | `eventCode=NLIS,SUSP,RETU,MOVE` — **chưa ai dùng** |
| 3.9 | ĐHCĐ | **VCI IQ** | `eventCode=AGME,AGMR,EGME` |
| 3.10 | Ngày công bố BCTC | ❓ VCI IQ | `eventCode=AIS` hoặc `publicDate` — chưa xác minh |

> ⚠️ **Cách gọi bắt buộc:** `eventCode=DIV,ISS` + `fromDate` + **phân trang**. Gọi mặc định
> (`page=0, size=50`, cả 15 loại sự kiện) thì giao dịch nội bộ chiếm hết chỗ — ACB lấy được
> 6 sự kiện thay vì 30 (E0014). Đây là lỗi đang sống trong `build_corp_actions.py`.

### Nhóm 4 — Chỉ số & thị trường

| # | Dữ liệu | SSI | DNSE | VCI IQ | Ngoài | **Đề xuất** |
|---|---|:--:|:--:|:--:|:--:|---|
| 4.1 | OHLC chỉ số | ✅ `DailyIndex` (21 trường) | ✅ `/index` | ⚠️ chỉ danh sách (37) | — | **SSI** |
| 4.2 | **Độ rộng thị trường** | ✅ `Advances/Declines/NoChanges/Ceilings/Floors` | — | — | — | **SSI** (duy nhất) |
| 4.3 | KL & giá trị toàn thị trường | ✅ `TotalMatchVol/Val` | — | — | — | **SSI** |
| 4.5 | Phái sinh VN30F1M/F2M | ✅ `Securities market=DER` | ✅ `/derivative` | — | — | **DNSE** giá · **SSI** danh mục |
| 4.6 | Chứng quyền (CW) | ✅ có trong `Securities` | ❓ HTTP 400 với mã thử | — | — | ❓ cần thử mã CW còn hiệu lực |
| 4.7 | ETF nội | ✅ | ✅ `/stock` | — | — | như cổ phiếu |
| 4.8 | Realized vol VNINDEX | — | — | — | — | **dẫn xuất** từ 4.1 — không cần nguồn |

> **4.4 đã bỏ** (quyết định 2026-07-30).
>
> **Đính chính "VIX":** hệ thống chưa bao giờ lấy chỉ số biến động CBOE. `fetch_vix.py`
> lấy **mã cổ phiếu VIX trên HOSE** (CTCP Chứng khoán VIX) + chỉ số `VNFINLEAD`, qua VCI
> chart API. Cũng không có chỉ số quốc tế nào. Xem ghi chú ở tài liệu (1) §4.

### Nhóm 5 — Cơ bản

| # | Dữ liệu | SSI | VCI IQ | **Đề xuất** |
|---|---|:--:|:--:|---|
| 5.1 | BCTC quý (CĐKT/KQKD/LCTT/TM) | — | ✅ `/financial-statement/metrics` — 4 nhóm | **VCI IQ** |
| 5.2 | BCTC năm | — | ✅ cùng endpoint | **VCI IQ** |
| 5.3 | Chỉ số tài chính (PE/PB/ROE…) | — | ✅ `/statistics-financial` — 41 bản ghi, từ 2018 | **VCI IQ** |
| 5.4 | Số cổ phiếu lưu hành | ✅ `SecuritiesDetails.ListedShare` | ✅ `numberOfSharesMktCap` | **SSI** (đối chứng bằng VCI) |
| 5.5 | Free float | — | — | 🔴 chưa có nguồn xác minh |
| 5.6 | Vốn hoá | — | ✅ `marketCap` | **VCI IQ** |
| 5.7 | Cổ đông lớn | — | ✅ `/shareholder` — 73 bản ghi FPT | **VCI IQ** ← *chưa ai dùng* |
| 5.7b | Ban lãnh đạo · công ty con | — | ❓ 404 trên IQ | ❓ có đường `trading.vietcap.com.vn/data-mt/graphql` — chưa xác minh |
| 5.8 | Kế hoạch KD / nghị quyết HĐQT | — | ⚠️ `eventCode=OTHE` một phần | VCI IQ, không đầy đủ |

### Nhóm 6 — Vĩ mô

Không nguồn nào trong ba API phục vụ nhóm này.

| # | Dữ liệu | Nguồn | Trạng thái |
|---|---|---|---|
| 6.1 | USD/VND | Vietcombank (web) · Yahoo · SBV (trung tâm) | 🟡 hai bảng trùng, phải hợp nhất |
| 6.2 | Tỷ giá khác | Vietcombank | ✅ |
| 6.3 | **Lãi suất điều hành SBV** | **SBV — bảng 1 của trang `sbv.gov.vn/vi/lãi-suất1`** | 🔴 0 dòng — **nguồn có, scraper không đọc bảng đó** |
| 6.4 | **Lãi suất liên NH qua đêm + kỳ hạn** | **SBV — bảng 2 của cùng trang** | 🟡 148 dòng, sâu 1 năm |
| 6.4c | Doanh số liên ngân hàng | SBV — cùng bảng 2 | 🔴 5 dòng |
| 6.5 | Lợi suất TPCP | — | 🔴 chưa có nguồn |
| 6.6 | CPI/GDP/FDI/cung tiền | WorldBank · GSO | 🟠 WorldBank năm, đã cũ |
| 6.7 | Hàng hoá | Yahoo (15 mã futures) | ✅ |

**Chi tiết nguồn SBV** (đo trực tiếp 2026-07-30, E0019). URL anh đưa **chính là URL đã nằm
trong code** — `macro_daily.py:78 SBV_INTERBANK_URL`. Trang trả về **hai bảng**:

| | Nội dung | Trạng thái |
|---|---|---|
| **Bảng 1** | `Loại lãi suất \| Giá trị \| Văn bản quyết định \| Ngày áp dụng` — tái chiết khấu **3,000%**, tái cấp vốn **4,500%**, áp dụng **19/03/2023** | 🔴 **scraper bỏ qua hoàn toàn** ⇒ đây là toàn bộ nguyên nhân mục 6.3 rỗng |
| **Bảng 2** | `Thời hạn \| Lãi suất BQ liên NH \| Doanh số` — Qua đêm **2,17** · 1 Tuần **4,53** · 2 Tuần **5,23** | ✅ đang đọc |

**Trang SBV chỉ có ảnh hiện tại** — không bộ chọn ngày, không trang lịch sử. Nhưng lịch sử
lấy được ở nơi khác:

### trolyluat.vn — CÀO ĐƯỢC, sâu 15,5 năm (E0022)

Ghi chú trong `backfill_interbank_trolyluat.py` nói API *"mã hoá, không cào được bằng HTTP
thuần"* nên lịch sử phải đọc tay ra khỏi Highcharts. **Sai.** Cào được bình thường:

```
POST https://trolyluat.vn/api/sbv-interest-rates/chart-data
body {start_date, end_date, terms:[{value,column,…}], datetime:"YYYYMMDDHHmmss"}
→ {code:0, data:{a,b,c}}
   key        = b64decode("ux8n7q7mJd9a3YF9h+9J7C6Y5nqf" + datetime + "U=")   (32 byte)
   iv         = b64decode(a)
   ciphertext = b64decode(b) + b64decode(c)          # c = auth tag
   AES-256-GCM
```

**Khoá suy từ chính trường `datetime` mình gửi lên** ⇒ tự tính được, không có bí mật nào
phía server. Đây là che khuất, không phải bảo mật.

| | Hiện có | trolyluat |
|---|---|---|
| Số dòng | 148 | **3.854** |
| Từ | 2025-07-22 | **2011-01-04** |
| Đến | 2026-07-27 | 2026-07-27 |
| Kỳ hạn | ON · 1W · 2W · 1M · 3M · 6M · 9M | y hệt |
| Dòng thiếu (qua đêm) | — | **0** |
| Doanh số (turnover) | có (5 dòng) | ❌ **không có** |

Cột: `id · date (dd/mm/yyyy) · rate_overnight · rate_1_week · rate_2_week · rate_1_month ·
rate_3_month · rate_6_month · rate_9_month`. `rate_6_month`/`rate_9_month` thiếu 1.341 dòng
— **thiếu tại nguồn**, không phải lỗi cào.

**Kết luận phân vai:**

| Nguồn | Vai trò |
|---|---|
| **trolyluat** | **Lịch sử** 2011 → nay, một lần cào là xong. Vượt xa yêu cầu 2015+ |
| **SBV bảng 2** | Cập nhật hằng ngày + **doanh số** (trolyluat không có) |
| **SBV bảng 1** | **Lãi suất điều hành** (6.3) — chưa ai đọc |

Dump thủ công `interbank_history_trolyluat.json` (2025-07 → 2026-03) giờ **thừa** — thay
bằng lần cào lặp lại được. Đồng thời nên đối chiếu dump cũ với bản cào để xem lần đọc tay
trước có sai chỗ nào không.

### Nhóm 7 — Tham chiếu / master

| # | Dữ liệu | SSI | VCI IQ | **Đề xuất** |
|---|---|:--:|:--:|---|
| 7.1 | Danh mục mã theo sàn | ✅ `Securities` — **HOSE/HNX/UPCOM/DER đủ 4** | ✅ `search-bar` | **SSI** |
| 7.2 | Ngành (ICB) | — | ✅ `/sectors/icb-codes` (177) + `company.sector` | **VCI IQ** |
| 7.3 | **Ngày niêm yết** | ⚠️ `FirstTradingDate` **rỗng với cổ phiếu** | — | 🔴 **chưa có nguồn** — nguyên nhân VHM có 629 dòng trước ngày niêm yết |
| 7.4 | Bảng bước giá · lô | ✅ `TickPrice1-4` / `TickIncrement1-4` / `LotSize` | — | **SSI** (duy nhất) |
| 7.5 | ISIN | ⚠️ có trường, `None` với ACB | — | ❓ độ phủ chưa rõ |
| 7.6 | Lịch giao dịch | ✅ **dẫn xuất**: ngày có dòng `DailyIndex` cho VNINDEX = phiên thật | — | **dẫn xuất từ SSI** — hết cảnh bảng phải nhớ nạp |
| 7.7 | Universe theo dõi | — | — | nội bộ (khai báo) |
| 7.8 | **Hệ sinh thái / nhóm sở hữu** | — | ⚠️ `shareholder` cho **manh mối** (ai sở hữu ai), không cho tên nhóm | **báo chí tài chính** (bảng khai tay) + `shareholder` để xác nhận — xem tài liệu (1) §9.1 |

### Nhóm 8 — Văn bản

| # | Dữ liệu | Nguồn | Ghi chú |
|---|---|---|---|
| 8.1 | Báo cáo phân tích CTCK | Vietstock (scrape) | ✅ 9.456 báo cáo |
| 8.2 | Tin tức / CBTT | **VCI IQ** `/v1/news-events-for-chart` — 189 bản ghi FPT | ← *chưa ai dùng* |
| 8.3 | Bản cáo bạch · nghị quyết | — | 🔴 chưa có nguồn |

---

## 3. Tổng kết theo nguồn — ai gánh gì

| Nguồn | Số mục phục vụ | Độc quyền (không nguồn nào thay được) |
|---|---|---|
| **SSI** | 17 | Giá **thô** · trần/sàn/tham chiếu · giá BQ · thoả thuận · độ rộng thị trường · bước giá/lô · khối ngoại+room · intraday raw sâu |
| **VCI IQ** | 15 | **Toàn bộ sự kiện doanh nghiệp** · BCTC · chỉ số tài chính · cổ đông · ngành · tin tức |
| **DNSE** | 6 | Lịch sử adjusted 2012–2020 (SSI không với tới) · khung 1H sâu 3 năm |
| VCI trading | 0 | — **loại khỏi luồng** |
| Ngoài (Yahoo/VCB/SBV/Vietstock) | 10 | Hàng hoá · tỷ giá · lãi suất · VIX · báo cáo CTCK · tự doanh |

**Ba nguồn API không chồng lấn nhiều như tưởng.** Chúng gần như **bổ sung** chứ không
thay thế nhau: SSI giữ vi cấu trúc + giá thô, VCI IQ giữ toàn bộ mặt doanh nghiệp, DNSE
giữ chiều sâu lịch sử. Đây là lý do kỹ thuật khiến round-robin sai từ gốc — **không có
hai nguồn nào đủ giống nhau để thay phiên**.

---

## 4. Mục KHÔNG nguồn nào phục vụ (6)

| # | Dữ liệu | Hệ quả | Đường ra |
|---|---|---|---|
| 3.6 | **Giá phát hành RIGHTS** | 8 sự kiện RIGHTS 2026 không tính được TERP | Suy ngược từ thị trường (gắn nhãn) hoặc nhập tay từ CBTT |
| 7.8 | **Hệ sinh thái / nhóm sở hữu** | Không đo được rủi ro tập trung; thiếu feature return nhóm | Bảng khai báo tay; `shareholder` của VCI IQ cho manh mối |
| 6.5 | Lợi suất TPCP | Thiếu đường cong lãi suất | — |
| 8.3 | Bản cáo bạch | — | — |

> ✅ **Hai mục ra khỏi danh sách này ngày 2026-07-30** (E0023) — dump đủ 41 trường của
> `VCI IQ /v1/company` cho thấy nguồn đã có sẵn:
> **7.3 ngày niêm yết** = `listingDate` (FPT `2006-12-13`) ·
> **5.5 free float** = `freeFloat` + `freeFloatPercentage`.
> Trước đó tôi kết luận "không nguồn nào có" vì `SecuritiesDetails.FirstTradingDate` của
> SSI **rỗng với cổ phiếu** — đúng với SSI, nhưng chưa hỏi hết VCI.

**Mục 7.3 và lãi suất liên ngân hàng có tính cấp thời**: chỉ tích luỹ được từ lúc bắt đầu
chụp hằng ngày. Càng để lâu càng mất, không mua lại được.

---

## 5. vnstock có phải đường duy nhất ở chỗ nào không?

**Không.** Đã kiểm 16 đường VCI IQ bằng `requests` thuần (E0016): 11 chạy, và tất cả dữ
liệu vnstock đang cung cấp đều lấy được trực tiếp. Chỗ duy nhất còn nghi là **5.7b ban
lãnh đạo / công ty con** — 404 trên IQ, vnstock lấy qua `trading.vietcap.com.vn/data-mt/graphql`.
Đó vẫn là **endpoint của VCI**, chỉ là chưa dựng được truy vấn GraphQL đúng. Không có mục
nào phụ thuộc vnstock về bản chất.

⇒ **Gỡ vnstock khỏi luồng dữ liệu được.** Giữ lại (nếu muốn) làm công cụ đối chứng thủ công.

---

## 5a. Kiểm kê CỬA theo nhà cung cấp — tra trước mọi câu "nguồn X có Y không?"

> ⚠️ Bảng dưới là **bản sao để đọc**. Bản gốc là `meta.vendor_door` trong `quant_v2`,
> và `meta.source_contract` có khoá ngoại tới nó — **không khai được một nguồn mà
> cửa của nó chưa nằm trong sổ cửa** (`050_vendor_door.sql`, đã thử và bị chặn thật).

| Nhà | Cửa | Token | Sâu từ | Thang | Tình trạng |
|---|---|---|---|---|---|
| DNSE | `services.entrade.com.vn/chart-api/v2/ohlcs/stock` | không | 2012-03-20 | ADJUSTED | đang dùng |
| DNSE | `…/ohlcs/derivative` | không | ? | ADJUSTED | chưa dùng |
| DNSE | `api-bo.dnse.com.vn/senses-api/events` | không | 2007-03-06 | — | đang dùng |
| DNSE | `api.dnse.com.vn/auth-service/login` | — | — | — | không cần |
| **SSI** | `fc-data.ssi.com.vn/api/v2/Market/DailyStockPrice` | **có** | **2020-02-26** | **RAW** | chưa dùng |
| SSI | `…/IntradayOhlc` · `…/DailyIndex` | có | ? | — | chưa dùng |
| **SSI** | `iboard-api.ssi.com.vn/statistics/charts/history` | không | **2000-07-28** | ADJUSTED | **đang dùng** |
| SSI | `…/company/stock-price` | không | ≈ trên | **MIXED** ⚠️ | chưa dùng |
| SSI | `…/charts/history/{foreign,proprietary}-trading-history` | không | — | — | 🔴 **404** |
| VCI | `trading.vietcap.com.vn/api/chart/OHLCChart/gap` | không | 2006-01-19 | ADJUSTED | đối chứng |
| VCI | `…/gap-chart` | không | — | — | 🔴 rỗng |
| VCI | `iq.vietcap.com.vn/api/iq-insight-service/v1/events` · `/company` | không | 2010-01-01 | — | đang dùng |

**Tổng: DNSE 4 cửa · SSI 7 · VCI 4.** Chi tiết từng cửa: `SELECT * FROM meta.vendor_door`.

Hai kết luận đã đo, đừng đo lại:
- **Không cửa nào cho O/H/L THÔ trước 2020-02-26.** Đã thử VCI `gap` + `gap-chart`,
  DNSE với `adjusted=false`/`type=raw`, iBoard `charts/history` 6 biến thể — tất cả
  trả adjusted (`E0057`).
- `company/stock-price` **trộn thang trong cùng một dòng**: `open/high/low` đã điều
  chỉnh, `closePrice` thô (`L ≤ closeAdj ≤ H` đúng 250/250; `L ≤ close ≤ H` đúng 0/250).
  Không dùng làm OHLC (`E0056`).

### ↩ Vì sao mục 5b dưới đây từng dẫn đến kết luận sai (`E0058`)

5b khai *"SSI — 9 endpoint"* rồi kết luận SSI chỉ sâu tới 2020-02-26. **Cả 9 đều là
FastConnect.** iBoard — sâu tới 2000 và đã nằm trong repo từ trước
(`ssi_foreign_flow.py`, `ssi_prop_flow.py`) — không có trong danh sách.

Phương pháp 5b tự khai là *"grep toàn repo"*, và grep đó không sai. **Sai ở từ khoá:**
danh sách ứng viên lấy từ *tài liệu FastConnect* rồi grep xem cái nào được dùng. Câu
được trả lời là *"endpoint nào của API này chưa dùng?"* — hẹp hơn câu cần hỏi là
*"nhà cung cấp này có mấy đường vào?"*. Grep theo `https?://` mất 30 giây và ra ngay.

**Quy tắc rút ra:** đơn vị kiểm kê là **host**, danh sách lấy **từ repo**, không lấy
từ tài liệu vendor.

---

## 5b. Nguồn ĐANG TRẢ VỀ mà pipeline KHÔNG ĐỌC

> Mục này **chỉ đúng trong phạm vi FastConnect** — xem §5a để biết toàn bộ cửa.

Câu hỏi "SSI/DNSE có thông tin gì không dùng?" — đo bằng cách grep toàn repo.

### SSI FastConnect — 9 endpoint, code sản xuất dùng **2** (E0020)

| Endpoint | Dùng? | Cấp được mục |
|---|---|---|
| `DailyStockPrice` | ✅ nhưng chỉ đọc **7/31 trường** | — |
| `IntradayOhlc` | ✅ | 1.3 |
| `SecuritiesDetails` | ❌ **0 tham chiếu** | 5.4 KLNY · 7.4 bước giá/lô · 7.5 ISIN |
| `DailyIndex` | ❌ **0 tham chiếu** | 4.1 · **4.2 độ rộng thị trường** · 4.3 · 7.6 lịch giao dịch |
| `IndexList` | ❌ **0 tham chiếu** | danh sách 27 chỉ số |
| `IndexComponents` | ❌ | 4.4 rổ chỉ số |
| `Securities` | ❌ | 7.1 danh mục mã 4 sàn |
| `DailyOhlc` | ❌ | (bản gọn của DailyStockPrice) |
| `AccessToken` | ✅ | auth |

**18/31 trường của `DailyStockPrice` không đọc ở đâu:**
`Market` · `Time` · `AveragePrice` · `PriceChange` · `PerPriceChange` · **`CeilingPrice`** ·
**`FloorPrice`** · **`RefPrice`** · **`TotalMatchVal`** · `TotalDealVol` · `TotalDealVal` ·
`TotalTradedVol` · `TotalTradedValue` · `TotalBuyTrade` · `TotalBuyTradeVol` ·
`TotalSellTrade` · `TotalSellTradeVol` · `ClosePriceAdjusted` *(chỉ dùng trong script so sánh)*

> **Sáu mục đánh 🔴 "chưa lấy" ở tài liệu (1) thực ra đã nằm sẵn trong phản hồi mà pipeline
> đang nhận mỗi ngày rồi vứt đi:** 1.6 giá trị GD · 1.7 trần/sàn/tham chiếu · 1.8 giá BQ ·
> 1.9 thoả thuận · 4.2 độ rộng · 7.4 bước giá. Không tốn thêm một lời gọi API nào.

### DNSE — gần như không thừa gì

7 khoá `t o h l c v nextTime`; dùng 6, chỉ `nextTime` (con trỏ phân trang) không dùng.
**Thừa ở mức đường dẫn**: `/index` và `/derivative` chưa dùng (dự án lấy chỉ số từ VCI).
DNSE là nguồn **nghèo trường nhất** — đây là lý do nó chỉ nên giữ vai trò chiều sâu lịch sử.

### VCI IQ — 5 endpoint chạy được mà chưa ai gọi

`insider-transaction` (2.4) · `shareholder` (5.7) · `sectors/icb-codes` (7.2) ·
`news-events-for-chart` (8.2) · `market-indices`

---

## 6. Đã giải quyết các mục 🔴 "chưa xác minh" của `C:/Users/OS/Lucius/Projects/PRJ quant/06_document/DATA_SOURCES.md`

| Mục cũ | Kết quả hôm nay |
|---|---|
| #1 Mốc SSI `2020-02-26` cố định hay cuộn | **Cố định** — không đổi sau 2 ngày, 5/5 mã (E0012) |
| #3 DNSE có tham số bật raw không | **KHÔNG** — 4 biến thể tham số đều bị bỏ qua, trả kết quả giống hệt (E0017) |
| #8 VCI IQ còn endpoint nào khác | Thêm **5 đường chạy được**: `insider-transaction` · `shareholder` · `sectors/icb-codes` · `news-events-for-chart` · `market-indices` (E0016) |

Còn treo: #2 DNSE có corporate action không · #4 DNSE rate limit · #6 vnstock 4.0.5
breaking change (**hết quan trọng** nếu gỡ vnstock) · #7 `TotalBuyTrade`=0 universal hay riêng mã.

---

## Bảo trì

- Thêm endpoint mới → thêm dòng, kèm mã bằng chứng `E####`.
- Đổi nguồn cho một mục → sửa cột **Đề xuất** và ghi lý do vào `CHANGELOG.md`.
- Nghi nguồn đổi API → chạy lại `probe_sources.py` + `verify_sources_claims.py`.
- File này **không** chứa schema bảng — xem tài liệu (3).
