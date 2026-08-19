# Nguồn VCI / Vietcap — trường khả dụng

> Lập lại 2026-08-19 **từ nguồn v2**: `INGEST_SPEC.md` (D7–D14) + code thật trong
> `3. Build\code\` + **lời gọi xác minh trực tiếp cùng ngày**. Không dùng
> `PRJ quant\06_document\DATA_SOURCES.md` — tài liệu đó mô tả pipeline v1.

**VCI có HAI bề mặt, chất lượng khác hẳn nhau — không được gộp thành "nguồn VCI".**

| Bề mặt | Dùng cho | Kết luận |
|---|---|---|
| `iq.vietcap.com.vn/api/iq-insight-service/*` | mặt doanh nghiệp | **giữ** — chưa đo thấy khuyết tật nào |
| `trading.vietcap.com.vn/api/*` | giá | **loại khỏi luồng** — đã đo hỏng |

## Phiên bản API

| Bề mặt | Host | Version trong đường dẫn | Auth |
|---|---|---|---|
| IQ Insight | `iq.vietcap.com.vn` | **`/api/iq-insight-service/v1/`** | không — chỉ `User-Agent` + `Referer` |
| Trading | `trading.vietcap.com.vn` | **không có đoạn version** (`/api/chart/…`) | không |

Bề mặt IQ có `v1` nên đổi bản sẽ thấy ở URL. Bề mặt trading không đánh version — nhưng nó
đã bị loại khỏi luồng nên rủi ro đó không còn quan trọng.

Header bắt buộc cho mọi lời gọi IQ:

```
User-Agent: Mozilla/5.0 … Chrome/126.0
Referer:    https://iq.vietcap.com.vn/
```

---

# Cửa nào v2 thực sự gọi

| Cửa | Script v2 | Vai trò |
|---|---|---|
| `iq…/v1/events` | `daily_events.py` (`lay_vci`) | sự kiện DN — **một trong ba nguồn gộp** |
| `iq…/v1/company/{ticker}` | `load_csv.py` | hồ sơ công ty |
| `trading…/chart/OHLCChart/gap` | `kiem_iboard.py` · `probe_adj_3nguon.py` · `sweep_dnse_vs_vci.py` | **chỉ đối chứng**, không nạp |

**Chưa cài:** `financial-statement` · `statistics-financial` · `insider-transaction` ·
`shareholder` · `sectors/icb-codes` · `news-events-for-chart`.

Sự kiện DN của v2 gộp **ba nguồn**: VCI IQ · DNSE `senses-api/events` · Vietstock
`EventsTypeData`. Phân loại và bóc tỷ lệ do `ca_parse.py` làm.

---

# 1. `/v1/events` — sự kiện doanh nghiệp

Bảng quan trọng nhất sau giá: vừa là **đầu vào tính giá điều chỉnh**, vừa là **thước kiểm
chính giá đó**.

| | |
|---|---|
| Endpoint | `GET .../v1/events` |
| Tham số | `ticker` · `fromDate` `toDate` (**YYYYMMDD**) · `eventCode` · `page` (0-based) · `size` |
| Phạm vi | **từ 2010**, liên tục mỗi năm, 8/8 mã kiểm (E0013) |
| Quy mô | FPT: **79** sự kiện `DIV,ISS` · 382 sự kiện mọi loại |

## ⚠️ Cách gọi BẮT BUỘC

```
eventCode=DIV,ISS  +  fromDate=20100101  +  phân trang tới hết
```

Gọi mặc định là **sai** (E0014): mặc định `page=0, size=50`, **cả 15 event code**, cửa sổ
10 năm ⇒ giao dịch nội bộ (`DDINS`/`DDIND`) chiếm hết 50 chỗ. **ACB lấy được 6 sự kiện
trong khi thực tế có 30.**

| `eventCode` | Nghĩa | Cần cho |
|---|---|---|
| `DIV` | trả cổ tức | hệ số điều chỉnh |
| `ISS` | phát hành thêm (thưởng, ESOP, quyền mua) | hệ số điều chỉnh |
| `DDIND` `DDINS` `DDRP` | giao dịch cổ đông lớn / nội bộ | 2.4 |
| `AGME` `AGMR` `EGME` | ĐHCĐ thường niên / bất thường | 3.9 |
| `NLIS` `SUSP` `RETU` `MOVE` | niêm yết mới · đình chỉ · trở lại · chuyển sàn | 3.8 + ngày niêm yết |
| `AIS` `MA` `OTHE` | khác | — |

**17 trường** (mẫu FPT, ESOP 2026):

| Trường | Mẫu | Ý nghĩa | Dùng |
|---|---|---|:--:|
| `id` | `6a309747e56b40f0…` | khoá của nhà cung cấp | ✅ |
| `ticker` · `organCode` | `FPT` | mã | ✅ |
| `eventCode` | `ISS` | mã loại sự kiện | ✅ |
| `category` | `DIVIDEND` | nhóm lớn | ✅ |
| `eventNameVi` / `En` | `Phát hành cổ phiếu` | tên loại | ✅ |
| **`eventTitleVi`** | `Phát hành cổ phiếu - Phát hành cho CBCNV tỉ lệ 0.1%` | 🔑 **nguồn DUY NHẤT để phân loại** STOCK_DIV / RIGHTS / SPLIT / ESOP | ✅ |
| `eventTitleEn` | `Share Issue - ESOP ratio 0.1%` | nt, tiếng Anh | ⬜ |
| **`exrightDate`** | `2026-06-24` | 🔑 **ngày GDKHQ** — mốc nhảy của hệ số | ✅ |
| `recordDate` | `2026-06-24` | ngày chốt danh sách | ⬜ |
| `publicDate` | `2026-06-29` | ngày công bố | ⬜ |
| **`exerciseRatio`** | `0.00135` | 🔑 **tỷ lệ thực hiện** | ✅ |
| `displayDate1` `displayDate2` | `2026-06-24` · `2026-06-29` | ngày hiển thị | ⬜ |
| `organNameVi` / `En` | `Công ty Cổ phần FPT` | tên công ty | ⬜ |

- **`value_per_share`** (cổ tức tiền, VND/cp) chỉ có với `DIV`, **`NaN` với `ISS`** — phân
  loại phải theo `eventTitleVi`, **không** theo sự có mặt của trường.
- **Không có giá phát hành của RIGHTS** ⇒ mục 3.6 vẫn không nguồn nào cấp.
- **Đối chứng đã chạy:** hệ số suy từ `exerciseRatio` **khớp** hệ số đo từ
  `ClosePriceAdjusted/ClosePrice` ở 4/4 mã (PVD `0.5992` · VTP `0.8521` · HPG `0.8927` và
  `0.7439` · MBB `0.9615`).

---

# 2. `/v1/company/{ticker}` — hồ sơ công ty

Đóng hai mục trước đó tưởng không nguồn nào có: **`listingDate` (7.3)** và
**`freeFloat` (5.5)** — E0023.

**41 trường**, nhóm chính (mẫu FPT):

| Trường | Mẫu | Ý nghĩa | Đóng mục |
|---|---|---|---|
| **`listingDate`** | `2006-12-13` | 🔑 **ngày niêm yết** | **7.3** |
| **`freeFloat`** | `1457177458` | số CP tự do chuyển nhượng | **5.5** |
| **`freeFloatPercentage`** | `0.85` | tỷ lệ free float | **5.5** |
| `numberOfSharesMktCap` | `1714326422` | số CP tính vốn hoá | 5.4 |
| `marketCap` | `108002564586000` | vốn hoá (VND) | 5.6 |
| `currentPrice` | `63000` | giá hiện tại | — |
| `foreignerPercentage` | `0.269` | tỷ lệ sở hữu NN hiện tại | — |
| `maximumForeignPercentage` | `0.49` | **trần room ngoại** | — |
| `statePercentage` | `0.0567` | tỷ lệ sở hữu nhà nước | — |
| `sector` / `sectorVn` | `Technology` / `Công nghệ Thông tin` | ngành | 7.2 |
| `icbCodeLv2` · `icbCodeLv4` | `9500` · `9537` | mã ngành ICB | 7.2 |
| `comGroupCode` · `comTypeCode` | `VNINDEX` · `CT` | nhóm sàn, loại hình DN | — |
| `averageMatchValue1Month` | `535059680718` | GTGD bình quân 1 tháng — ADTV | — |
| `averageMatchVolume1Month` | `7755993` | KLGD bình quân 1 tháng | — |
| `highestPrice1Year` · `lowestPrice1Year` | `108948` · `61500` | đỉnh/đáy 52 tuần | — |
| `rating` · `ratingAsOf` · `analyst` | `BUY` · `08-Jun-26` · `Ngan Ly` | khuyến nghị Vietcap | 8.1 |
| `targetPrice` · `upsideToTargetPercent` | `90300` · `0.433` | giá mục tiêu | 8.1 |
| `dividendPerShareTsr` · `projectedTSRPercentage` | `2300` · `0.470` | cổ tức/cp, TSR dự phóng | — |
| `prevInsight` | `{targetPrice: 116600, …}` | khuyến nghị **kỳ trước** | — |
| `profile` / `enProfile` | `<div>…` | giới thiệu DN (HTML) | — |
| **`isBank`** | `False` | 🔑 cờ phân loại — **bắt buộc để lọc chỉ số ngân hàng** | — |
| `bank` · `listing` · `inCu` | `True` | cờ khác | — |

> ⚠️ Bảng này **trộn dữ liệu tĩnh (ngày niêm yết) với dữ liệu đổi hằng ngày (giá, vốn hoá,
> khuyến nghị)**. Trong `obs.security_ref`, `observed_at` giữ đúng vai trò tách hai loại.

---

# 3. `financial-statement` — BCTC

Chi tiết đầy đủ ở **`INGEST_SPEC.md` §D12** (đã đo lại và viết lại 2026-08-19). Tóm tắt
những gì phải nhớ:

| | |
|---|---|
| Tham số bắt buộc | `section` ∈ `BALANCE_SHEET` \| `INCOME_STATEMENT` \| `CASH_FLOW` \| `NOTE` |
| Phạm vi | năm **2018–2025** · quý **2018Q1–2026Q2** (34 kỳ) |
| `lengthReport` | **`5` = năm · `1`–`4` = quý** |
| `publicDate` | **có** — cơ sở duy nhất để dựng point-in-time |

**`/metrics` là theo từng mã, không phải từ điển dùng chung.** Nó trả khuôn của đúng loại
hình DN đó. Tiền tố trường mã hoá loại hình: `*a` thường · `*b` ngân hàng · `*i` bảo hiểm ·
`*s` chứng khoán. Khuôn một mã **trộn nhiều tiền tố**, nên **không được lọc theo tiền tố** —
phải lấy đúng danh sách `field` từ `/metrics` của chính mã đó.

Bản ghi trả về chứa nhiều trường hơn khuôn (FPT: 331 khoá `bs*` trong khi khuôn có 122);
trường ngoài khuôn là chỗ của ngành khác, để trống.

---

# 4. `statistics-financial` — chỉ số tài chính

| | |
|---|---|
| Endpoint | `GET .../v1/company/{ticker}/statistics-financial` |
| Phạm vi | **từ 2018**, theo quý — FPT 41 bản ghi |
| Quy mô | **60 trường/bản ghi** |

| Nhóm | Trường |
|---|---|
| Định danh kỳ | `year` `quarter` `yearReport` `organCode` `ratioTTMId` `ratioType` |
| Định giá | `pe` `pb` `ps` `priceToCashFlow` `evToEbitda` `marketCap` `numberOfSharesMktCap` `dividendYield` |
| Thanh khoản | `cashRatio` `quickRatio` `currentRatio` |
| Đòn bẩy | `debtPerEquity` `debtToEquity` `financialLeverage` `ownersEquity` |
| Sinh lời | `roe` `roa` `roic` `grossMargin` `ebitMargin` `preTaxProfitMargin` `afterTaxProfitMargin` |
| Hiệu quả | `assetTurnover` `fixedAssetTurnover` `daySaleOutstanding` `daysInventoryOutstanding` `daysPayableOutstanding` `cashCycle` |
| Tuyệt đối | `ebit` `ebitda` |
| **Riêng ngân hàng** | `netInterestMargin` `costToIncome` `cir` `npl` `loansGrowth` `depositGrowth` `ldrLoanDepositRatio` `car` `casaRatio` `loansLossReservesToNPLs` `provisionToOutstandingLoans` `equityToLoans` |

> ⚠️ **Trường ngân hàng trả `0.0` chứ không phải `NULL`** với DN phi ngân hàng — FPT có
> `npl = 0.0`, `netInterestMargin = 0.0`. Đưa thẳng vào feature sẽ tạo **0 giả**. Phải lọc
> theo `isBank` của `/v1/company`.

---

# 5. `insider-transaction` — giao dịch nội bộ

| | |
|---|---|
| Endpoint | `GET .../v1/company/{ticker}/insider-transaction` · `page` `size` |
| Quy mô | FPT **273** bản ghi · **35 trường** |
| Trạng thái | 🆕 chưa ai dùng — đóng mục 2.4 |

| Trường | Mẫu | Ý nghĩa |
|---|---|---|
| `eventCode` · `eventNameVi` | `DDIND` · `Giao dịch nội bộ: Giao dịch cá nhân` | loại |
| **`traderNameVi`** / `En` | `Nguyễn Văn Khoa` | người/tổ chức giao dịch |
| **`traderPositionVi`** / `En` | `Tổng Giám đốc` | chức vụ |
| `traderPersonId` | `35135` | định danh người |
| **`actionTypeCode`** · `actionTypeVi` | `B` · `Mua` | chiều giao dịch |
| **`shareRegister`** | `428368` | KL đăng ký |
| **`shareAcquire`** | `428368` | **KL thực hiện được** — khác đăng ký là tín hiệu |
| `shareBeforeTrade` · `shareAfterTrade` | `5414610` · `5842978` | sở hữu trước / sau |
| `ownershipAfterTrade` | `0.0034` | tỷ lệ sở hữu sau |
| **`tradeStatusVi`** / `En` | `Đã thực hiện xong` / `Done` | trạng thái — lọc mới đăng ký vs đã xong |
| `startDate` · `endDate` · `publicDate` | `2026-06-24` … | khoảng đăng ký, ngày công bố |
| `sourceUrlVi` | `http://fiinpro.com/News/Detail/…` | nguồn gốc |
| `roleNameVi` · `relativeNameVi` · `icbCodeLv1` | — | vai trò, người liên quan, ngành |

---

# 6. `shareholder` — cổ đông lớn

| | |
|---|---|
| Endpoint | `GET .../v1/company/{ticker}/shareholder` |
| Quy mô | FPT **73** bản ghi · **10 trường** |
| Công dụng thêm | 🔑 **xác nhận bảng hệ sinh thái (7.8) bằng dữ liệu** — dò sở hữu chéo giữa các mã |

| Trường | Mẫu | Ý nghĩa |
|---|---|---|
| `ownerName` / `En` | `Trương Gia Bình` | tên chủ sở hữu |
| **`ownerCode`** | `536` | định danh — **khoá để dò sở hữu chéo** |
| `positionName` / `En` | `Chủ tịch Hội đồng Quản trị` | chức vụ nếu là nội bộ |
| `quantity` | `117347966` | số cổ phiếu nắm giữ |
| `percentage` | `0.0689` | tỷ lệ sở hữu |
| `ownerType` | `INDIVIDUAL` | cá nhân \| tổ chức |
| `updateDate` · `publicDate` | `2026-02-03` · `2025-12-31` | cập nhật / hiệu lực |

---

# 7. `sectors/icb-codes` — ngành ICB

`GET .../v1/sectors/icb-codes` — **177 ngành**, tần suất tháng.

`name` (mã ICB, vd `1755`) · `enSector` / `viSector` · `icbLevel` (1–4) · `marketCap` ·
`isLevel1Custom` · `level1Custom`.

# 8. `news-events-for-chart` — tin tức / CBTT

`GET .../v1/news-events-for-chart` · `ticker` `fromDate` `toDate` `languageId=1` `eventCode`
FPT 2025→nay: **189** bản ghi.

`id` · `newsTitle` · `newsShortContent` · `publicDate` · `displayDate` · `isEvent` · `event`.

---

# Bề mặt giá — vì sao loại

Đã đo, không phải phỏng đoán:

| Cửa | Kết quả đo |
|---|---|
| `trading…/chart/OHLCChart/gap` | **8/8 dòng có `high = low = close`** · **135 timeout / 261 mã** · lệch ngày khi hỏi đúng một ngày |
| `trading…/gap-chart` | **rỗng** |

Bề mặt IQ không dính lỗi nào trong số này. Đây là lý do phải tách hai bề mặt trong tài liệu
và trong `meta.vendor_door`, thay vì ghi một dòng "VCI".

Cửa `OHLCChart/gap` **vẫn được giữ** trong v2 — nhưng chỉ trong script đối chứng
(`probe_adj_3nguon.py`, `sweep_dnse_vs_vci.py`), không nạp vào `obs`.

---

# Cạm bẫy

- **Lỗi trả về HTTP 200.** Thiếu hoặc sai `section` của `financial-statement` → HTTP 200
  nhưng thân có `status: 400`, `data: null`. Phải kiểm `body["status"]`, không chỉ
  `resp.status_code`.
- **`/metrics` theo từng mã** — dùng khuôn của mã này đọc số của mã khác sẽ ra kết luận sai.
- **`field = null` trong `/metrics`** (VCB 6 · HCM/VND 9 · BVH 1) — dòng tiêu đề phân nhóm,
  phải bỏ qua trước khi join.
- **Chỉ số ngân hàng trả `0.0` chứ không phải `NULL`** cho DN phi ngân hàng.
- **Gọi `/v1/events` mặc định là sai** — phải truyền `eventCode` và phân trang tới hết.
- **`value_per_share` là `NaN` với `ISS`** — phân loại theo `eventTitleVi`.

---

# Ánh xạ sang schema

| Cửa | Bảng L0 | Khoá |
|---|---|---|
| `/v1/events` | `obs.corp_action` | `(source, symbol, ex_date, event_code, observed_at)` |
| `/v1/company` | `obs.security_ref` | `(source, symbol, observed_at)` |
| `financial-statement` · `statistics-financial` | `obs.financial` | `(source, symbol, period, metric, observed_at)` |
| `shareholder` · `insider-transaction` | `obs.ownership` | `(source, symbol, owner_code, observed_at)` |
| `sectors/icb-codes` | `obs.security_ref` | như trên |
| `news-events-for-chart` | `obs.document` | `(source, doc_id)` |

VCI **không** vào `market.px_raw` và không vào bảng giá nào — bề mặt giá đã bị loại. Nó nuôi
`market.ca`, `features.financials`, `meta.securities`.

---

# Vai trò trong hệ

Phục vụ 15/60 mục, **độc quyền toàn bộ mặt doanh nghiệp**: sự kiện DN, BCTC, chỉ số tài
chính, cổ đông, ngành, tin tức. Không nguồn nào trong hệ thay được phần này.

Đo 2026-08-19: VCI **ngang hoặc hơn** `vnfinancialdata` ở mọi ngành về số chỉ tiêu, hơn rõ ở
bảng cân đối khối tài chính, và **độc quyền toàn bộ thuyết minh** (78–149 chỉ tiêu tuỳ mã;
`vnfinancialdata` không có `NOTE`). Xem bảng đầy đủ ở `INGEST_SPEC.md` §D12.

# Còn treo

- **5.7b ban lãnh đạo / công ty con**: 404 trên IQ. Đường duy nhất thấy được là
  `trading.vietcap.com.vn/data-mt/graphql` — **vẫn là endpoint của VCI**, chỉ chưa dựng được
  truy vấn GraphQL đúng.
- Sáu cửa IQ đã có đặc tả nhưng **chưa có loader** trong v2: `financial-statement` ·
  `statistics-financial` · `insider-transaction` · `shareholder` · `sectors/icb-codes` ·
  `news-events-for-chart`.
