# Đặc tả lấy dữ liệu — tra cứu từng dataset

> **Lập 2026-07-30 (S16).** Phụ lục tra cứu của bộ redesign DB.
> (1) `DATA_REQUIREMENTS.md` cần gì · (2) `DATA_SOURCE_MAP.md` nguồn nào cho được ·
> (3) `DB_DESIGN.md` thiết kế bảng · **(4) file này — lấy CỤ THỂ thế nào**
>
> **Mọi tên cột và giá trị mẫu dưới đây lấy từ lời gọi API THẬT ngày 2026-07-30** (E0024).
> Không dòng nào chép từ tài liệu nhà cung cấp hay viết theo trí nhớ.
>
> Cột **Dùng**: ✅ đang đọc · ⬜ nguồn trả về nhưng pipeline bỏ · 🆕 nên bắt đầu đọc

---

## Mục lục

| # | Dataset | Nguồn | Tần suất |
|---|---|---|---|
| [D1](#d1) | OHLCV ngày + khối ngoại | SSI `DailyStockPrice` | hằng ngày 16:15 |
| [D2](#d2) | OHLCV ngày đã điều chỉnh (lịch sử) | DNSE `chart-api` | 1 lần + kiểm định xoay vòng |
| [D3](#d3) | Nến 1 phút | SSI `IntradayOhlc` | hằng ngày EOD |
| [D4](#d4) | Chỉ số + độ rộng thị trường | SSI `DailyIndex` | hằng ngày 16:15 |
| [D5](#d5) | Danh mục mã | SSI `Securities` | tuần |
| [D6](#d6) | Chi tiết chứng khoán (KLNY, bước giá) | SSI `SecuritiesDetails` | tuần |
| [D7](#d7) | Sự kiện doanh nghiệp | VCI IQ `/v1/events` | hằng ngày |
| [D8](#d8) | Giao dịch nội bộ / cổ đông lớn | VCI IQ `insider-transaction` | hằng ngày |
| [D9](#d9) | Cổ đông lớn | VCI IQ `shareholder` | tháng |
| [D10](#d10) | Hồ sơ công ty (ngày NY, free float) | VCI IQ `/v1/company` | tuần |
| [D11](#d11) | Chỉ số tài chính | VCI IQ `statistics-financial` | quý |
| [D12](#d12) | BCTC | VCI IQ `financial-statement` | quý |
| [D13](#d13) | Ngành ICB | VCI IQ `sectors/icb-codes` | tháng |
| [D14](#d14) | Tin tức / CBTT | VCI IQ `news-events-for-chart` | hằng ngày |
| [D15](#d15) | Lãi suất liên NH — lịch sử | trolyluat | 1 lần |
| [D16](#d16) | Lãi suất SBV — hằng ngày | sbv.gov.vn | hằng ngày |
| [D17](#d17) | Hàng hoá | Yahoo Finance | hằng ngày |

**Universe áp dụng cho D1–D3, D7–D12: 204 mã** (`DATA_REQUIREMENTS.md` §9).

---

<a id="d1"></a>
## D1 — OHLCV ngày + khối ngoại · SSI `DailyStockPrice`

| | |
|---|---|
| **Endpoint** | `GET https://fc-data.ssi.com.vn/api/v2/Market/DailyStockPrice` |
| **Auth** | Bearer token (`AccessToken`, sống ~8h) — `.env` |
| **Tham số** | `symbol` · `fromDate` `toDate` (**dd/mm/yyyy**) · `pageIndex` · `pageSize` |
| **Giới hạn cứng** | **30 ngày/lời gọi** · từ chối `toDate ≥ hôm nay` · **`pageSize` chỉ nhận `10/20/50/100/1000`** |
| **Tần suất** | hằng ngày, 16:15 ICT · 204 lời gọi ≈ 4,5 phút (~1 lời gọi/giây) |
| **Phạm vi** | **2020-02-26** → nay. Mốc **cố định**, không cuộn theo ngày (E0012) |
| **Nạp lần đầu** | 204 mã × ~78 chunk (30 ngày) ≈ **5,6 giờ** — cần duyệt riêng |
| **Đơn vị giá** | **VND nguyên** |
| **Ngày không giao dịch** | **CÓ trả dòng**, `volume = 0` — khác DNSE (bỏ hẳn) |
| **Timeout đo được** | 0 / 261 mã |

**31 trường** (mẫu: ACB, 28/07/2026):

| Trường | Mẫu | Ý nghĩa | Dùng |
|---|---|---|:--:|
| `Symbol` | `ACB` | mã chứng khoán | ✅ |
| `TradingDate` | `28/07/2026` | ngày giao dịch, **dd/mm/yyyy** | ✅ |
| `Time` | `None` | luôn rỗng với dữ liệu ngày | ⬜ |
| `Market` | `HOSE` | sàn **tại thời điểm gọi** — ⚠️ không phải sàn của ngày quá khứ | ⬜ |
| `OpenPrice` | `22150` | giá mở cửa — **RAW** | ✅ |
| `HighestPrice` | `22650` | giá cao nhất — RAW | ✅ |
| `LowestPrice` | `22100` | giá thấp nhất — RAW | ✅ |
| `ClosePrice` | `22500` | **giá đóng cửa THÔ** — nguồn raw duy nhất trong 3 API | ✅ |
| `ClosePriceAdjusted` | `22500` | giá đóng cửa **đã điều chỉnh** cho mọi sự kiện sau ngày đó | ⬜ 🆕 |
| `AveragePrice` | `22650` | giá bình quân phiên | ⬜ 🆕 |
| `RefPrice` | `22300` | giá tham chiếu (đóng cửa phiên trước) | ⬜ 🆕 |
| `CeilingPrice` | `23850` | **giá trần** (+7% HOSE) | ⬜ 🆕 |
| `FloorPrice` | `20750` | **giá sàn** (−7%) | ⬜ 🆕 |
| `PriceChange` | `200` | thay đổi tuyệt đối so với tham chiếu (VND) | ⬜ |
| `PerPriceChange` | `0.90` | thay đổi (%) | ⬜ |
| `TotalMatchVol` | `15653800` | **KL khớp lệnh** (cổ phiếu) | ✅ |
| `TotalMatchVal` | `350277960000` | **GT khớp lệnh** (VND) — cần cho ADTV tính bằng tiền | ⬜ 🆕 |
| `TotalDealVol` | `0` | KL **thoả thuận** | ⬜ 🆕 |
| `TotalDealVal` | `0` | GT thoả thuận (VND) | ⬜ 🆕 |
| `TotalTradedVol` | `15653800` | tổng KL = khớp lệnh + thoả thuận | ⬜ |
| `TotalTradedValue` | `350277960000` | tổng GT | ⬜ |
| `ForeignBuyVolTotal` | `1959400` | KL nước ngoài **mua** | ✅ |
| `ForeignSellVolTotal` | `1188400` | KL nước ngoài **bán** | ✅ |
| `ForeignBuyValTotal` | `43753445000` | GT nước ngoài mua (VND) | ✅ |
| `ForeignSellValTotal` | `26532175500` | GT nước ngoài bán (VND) | ✅ |
| `ForeignCurrentRoom` | `323269297` | **room ngoại còn lại** (số cổ phiếu) | ✅ |
| `NetBuySellVol` | `771000` | KL mua ròng NN (mua − bán) | ✅ |
| `NetBuySellVal` | `17221269500` | GT mua ròng NN | ✅ |
| `TotalBuyTrade` | `0` | số **lệnh** đặt mua | ⬜ ❓ |
| `TotalBuyTradeVol` | `0` | KL đặt mua | ⬜ ❓ |
| `TotalSellTrade` | `0` | số lệnh đặt bán | ⬜ ❓ |
| `TotalSellTradeVol` | `0` | KL đặt bán | ⬜ ❓ |

❓ Bốn trường `TotalBuy/SellTrade*` = 0 với ACB — **chưa rõ là universal hay riêng mã**.

> **Bẫy 1 — `pageSize` sai trả `status=Error, data=None`**, rất dễ nhầm là lỗi mạng.
> **Bẫy 2 — `Market` là sàn HIỆN TẠI**, dùng nó cho ngày quá khứ sẽ sai (đã từng gây
> 141 báo động giả "vượt biên độ HOSE" cho mã khi đó còn ở UPCOM).
> **Bẫy 3 — `volume=0` không có nghĩa thiếu dữ liệu**: SSI trả dòng cho cả ngày không
> giao dịch. Phải đối chiếu lịch giao dịch (D4) mới phân biệt được.

---

<a id="d2"></a>
## D2 — OHLCV ngày đã điều chỉnh (lịch sử sâu) · DNSE

| | |
|---|---|
| **Endpoint** | `GET https://services.entrade.com.vn/chart-api/v2/ohlcs/{stock\|index\|derivative}` |
| **Auth** | **không cần** |
| **Tham số** | `symbol` · `resolution` · `from` `to` (**unix giây**) |
| **Đường theo loại** | `/stock` cổ phiếu + ETF · `/index` VNINDEX, VN30 · `/derivative` VN30F1M |
| **Tần suất** | **1 lần** nạp lịch sử; sau đó chỉ append bar hôm nay. Kéo lại full-history là **thao tác kiểm định xoay vòng**, không phải ingest |
| **Phạm vi** | `1D` từ **2012-03-20** (trần nền tảng) · `1W` 2021-02 · `1H` 2023-07 · `1m` ~3 tháng |
| **Đơn vị giá** | 🔴 **NGHÌN VND** (`c=22.5` = 22.500đ) — phải khai `price_unit='kVND'` |
| **Cơ sở giá** | **ADJUSTED**, và **không có tham số bật raw** — đã thử `adjusted=false` `adjust=0` `raw=true` `unadjusted=true`, cả 4 bị bỏ qua (E0017) |
| **Ngày không giao dịch** | **BỎ HẲN**, không trả dòng — ngược với SSI |
| **Chất lượng** | 0 null · 0 vi phạm logic OHLC trên 2.971 dòng · 0 timeout / 261 mã |

**7 khoá — mảng song song, không phải danh sách bản ghi** (mẫu ACB):

| Khoá | Mẫu | Ý nghĩa |
|---|---|---|
| `t` | `[1784772000, 1784858400, …]` | unix giây, đầu mỗi bar |
| `o` `h` `l` `c` | `[22.4, 22.1, 22.5]` … | OHLC, **nghìn VND**, đã điều chỉnh |
| `v` | `[27165000, 13447000, …]` | khối lượng (cổ phiếu) |
| `nextTime` | `0` | con trỏ phân trang; `0` = hết |

> **Đối chứng chéo đã chạy:** `v` của ACB ngày 24/07 = `13.447.000` **khớp đúng**
> `TotalMatchVol` của SSI. Hai nguồn độc lập cho cùng khối lượng ⇒ tin được.
>
> **Từ chối `from == to`** — phải hỏi theo cửa sổ rồi lọc.

---

<a id="d3"></a>
## D3 — Nến 1 phút · SSI `IntradayOhlc`

| | |
|---|---|
| **Endpoint** | `GET .../api/v2/Market/IntradayOhlc` · `resolution=1` |
| **Tần suất** | hằng ngày sau giờ đóng cửa |
| **Phạm vi** | **2020-02-26** → nay (trùng mốc daily) |
| **Cơ sở giá** | **RAW** — xác minh: PVD 27/05/2026 nến cuối `30.150` = `ClosePrice` raw, **không** phải `ClosePriceAdjusted` (`18.066`). **Không có cột adjusted** |
| **Số nến/phiên** | 193–228 |
| **Nạp lại toàn bộ** | ~**128 giờ** (vòng lặp theo ngày, không batch thật) — cần duyệt riêng |

| Trường | Mẫu | Ý nghĩa |
|---|---|---|
| `symbol` | `ACB` | mã |
| `time` | `2026-07-28 09:15:59` | **ICT, tz-naive**; lưu DB dạng UTC |
| `open` `high` `low` `close` | `22150` `22200` `22150` `22150` | VND nguyên, **RAW** |
| `volume` | `176800` | KL khớp trong phút |

> **Luật bất di dịch:** intraday **chỉ append hôm nay**, **cấm refetch quá khứ từ nguồn
> adjusted** — nếu không sẽ tạo "seam điều chỉnh" giữa chuỗi. Bar quá khứ đổi giá = cờ đỏ.

---

<a id="d4"></a>
## D4 — Chỉ số + độ rộng thị trường · SSI `DailyIndex`

| | |
|---|---|
| **Endpoint** | `GET .../api/v2/Market/DailyIndex` · `indexId` `fromDate` `toDate` `pageSize∈{10,20,50,100,1000}` |
| **Tần suất** | hằng ngày 16:15 |
| **Chỉ số khả dụng** | 27 (qua `IndexList`): VNINDEX · VN30 · VN100 · VNMIDCAP · VNSMALLCAP · VNDIAMOND · VNFINLEAD · VNX50 · 19 chỉ số ngành |
| **Vai trò kép** | vừa là dữ liệu chỉ số, vừa là **mỏ neo lịch giao dịch**: ngày có dòng VNINDEX với `TotalMatchVol>0` = phiên thật |

**21 trường** (mẫu VNINDEX 28/07/2026):

| Trường | Mẫu | Ý nghĩa | Dùng |
|---|---|---|:--:|
| `IndexId` · `IndexName` | `VNINDEX` | mã chỉ số | ✅ |
| `TradingDate` | `28/07/2026` | dd/mm/yyyy | ✅ |
| `Time` | `None` | luôn rỗng | ⬜ |
| `IndexValue` | `1680.62` | **điểm số**, không phải VND | ✅ |
| `Change` | `0.1160999…` | thay đổi tuyệt đối (điểm) | ⬜ |
| `RatioChange` | `0.70` | thay đổi (%) | ⬜ |
| **`Advances`** | `189` | **số mã TĂNG giá** | ⬜ 🆕 |
| **`Declines`** | `122` | **số mã GIẢM giá** | ⬜ 🆕 |
| **`NoChanges`** | `53` | số mã đứng giá | ⬜ 🆕 |
| **`Ceilings`** | `6` | **số mã kịch TRẦN** | ⬜ 🆕 |
| **`Floors`** | `6` | **số mã kịch SÀN** | ⬜ 🆕 |
| `TotalMatchVol` | `634234923` | KL khớp toàn thị trường | ⬜ 🆕 |
| `TotalMatchVal` | `13613399268440` | GT khớp toàn thị trường (VND) | ⬜ 🆕 |
| `TotalDealVol` `TotalDealVal` | `144704773` · `2379861079850` | thoả thuận toàn thị trường | ⬜ |
| `TotalVol` `TotalVal` | `778939696` · `15993260348290` | tổng = khớp + thoả thuận | ⬜ |
| `TotalTrade` | `0` | số lệnh — trả 0 | ⬜ ❓ |
| `TypeIndex` | `None` | rỗng | ⬜ |
| `TradingSession` | `C` | phiên: `C` = đã đóng cửa | ⬜ |

> 5 trường `Advances/Declines/NoChanges/Ceilings/Floors` là **toàn bộ mục 4.2 độ rộng thị
> trường** — feature `breadth` của Regime Gate đã khai trong C:/Users/OS/Lucius/Projects/PRJ quant/MASTER_PLAN.md nhưng chưa từng
> có dữ liệu. **Không nguồn nào khác trong 3 API có.**

---

<a id="d5"></a>
## D5 — Danh mục mã · SSI `Securities`

| | |
|---|---|
| **Tham số** | `market ∈ {HOSE, HNX, UPCOM, DER}` — **đủ 4 sàn** · `pageIndex` `pageSize` |
| **Tần suất** | tuần |
| **Quy mô** | 736 mã HOSE |

| Trường | Mẫu | Ý nghĩa |
|---|---|---|
| `Market` | `HOSE` | sàn |
| `Symbol` | `AAA` | mã |
| `StockName` | `CTCP NHUA AN PHAT XANH` | tên tiếng Việt (không dấu) |
| `StockEnName` | `An Phat Bioplastics Joint Stock Company` | tên tiếng Anh |

---

<a id="d6"></a>
## D6 — Chi tiết chứng khoán · SSI `SecuritiesDetails`

| | |
|---|---|
| **Tham số** | `market` · `symbol` · `pageIndex` `pageSize` |
| **Tần suất** | tuần |
| **Cấu trúc** | 3 trường ngoài (`RType` `ReportDate` `TotalNoSym`) + mảng **`RepeatedInfo`** lồng bên trong |

**`RepeatedInfo` — 29 trường** (mẫu ACB):

| Trường | Mẫu | Ý nghĩa | Dùng |
|---|---|---|:--:|
| `Symbol` | `ACB` | mã | 🆕 |
| `SymbolName` · `SymbolEngName` | `NGAN HANG TMCP A CHAU` | tên | 🆕 |
| `SecType` | `S` | loại: `S`=cổ phiếu | 🆕 |
| `MarketId` · `Exchange` | `HOSE` | sàn | 🆕 |
| **`ListedShare`** | `5136656599.0` | **số cổ phiếu niêm yết** | ⬜ 🆕 |
| **`LotSize`** | `100` | **lô giao dịch** | ⬜ 🆕 |
| **`TickPrice1`** / **`TickIncrement1`** | `1` / `10` | bước giá **10đ** cho giá từ 1đ | ⬜ 🆕 |
| **`TickPrice2`** / **`TickIncrement2`** | `10000` / `50` | bước **50đ** cho giá từ 10.000đ | ⬜ 🆕 |
| **`TickPrice3`** / **`TickIncrement3`** | `50000` / `100` | bước **100đ** cho giá từ 50.000đ | ⬜ 🆕 |
| `TickPrice4` / `TickIncrement4` | `None` | bậc 4 (không dùng với cổ phiếu) | ⬜ |
| `Isin` | `None` | 🔴 **rỗng với ACB** — độ phủ chưa rõ | ⬜ |
| `IssueDate` `MaturityDate` **`FirstTradingDate`** `LastTradingDate` | *(rỗng)* | 🔴 **rỗng với cổ phiếu** ⇒ **không dùng được cho ngày niêm yết** — phải lấy từ D10 | ⬜ |
| `Issuer` `ContractMultiplier` `SettlMethod` `Underlying` `PutOrCall` `ExercisePrice` `ExerciseStyle` `ExcerciseRatio` | `None`/`0`/`C` | dùng cho **phái sinh & chứng quyền**, rỗng với cổ phiếu | ⬜ |

> Bảng bước giá 3 bậc là đầu vào trực tiếp cho mô hình **chi phí giao dịch / trượt giá**
> trong backtest — hiện backtest đang giả định giá liên tục.

---

<a id="d7"></a>
## D7 — Sự kiện doanh nghiệp · VCI IQ `/v1/events` 🔑

Bảng quan trọng nhất sau giá: vừa là **đầu vào tính giá điều chỉnh**, vừa là **thước kiểm
chính giá đó**.

| | |
|---|---|
| **Endpoint** | `GET https://iq.vietcap.com.vn/api/iq-insight-service/v1/events` |
| **Auth** | **không cần** — chỉ `User-Agent` + `Referer` |
| **Tham số** | `ticker` · `fromDate` `toDate` (**YYYYMMDD**) · `eventCode` · `page` (0-based) · `size` |
| **Tần suất** | hằng ngày (delta) |
| **Phạm vi** | **từ 2010**, liên tục mỗi năm, 8/8 mã kiểm (E0013) |
| **Quy mô** | FPT: **79** sự kiện `DIV,ISS` · 382 sự kiện mọi loại |

### ⚠️ Cách gọi BẮT BUỘC

```
eventCode=DIV,ISS  +  fromDate=20100101  +  phân trang tới hết
```

**Gọi mặc định là sai và đang sai trong `build_corp_actions.py`** (E0014): mặc định là
`page=0, size=50`, **cả 15 event code**, cửa sổ 10 năm ⇒ giao dịch nội bộ (`DDINS`/`DDIND`)
chiếm hết 50 chỗ. **ACB lấy được 6 sự kiện trong khi thực tế có 30.**

**Bảng mã sự kiện:**

| `eventCode` | Nghĩa | Cần cho |
|---|---|---|
| `DIV` | trả cổ tức | hệ số điều chỉnh |
| `ISS` | phát hành thêm (thưởng, ESOP, quyền mua) | hệ số điều chỉnh |
| `DDIND` `DDINS` `DDRP` | giao dịch cổ đông lớn / nội bộ | mục 2.4 |
| `AGME` `AGMR` `EGME` | ĐHCĐ thường niên / bất thường | mục 3.9 |
| `NLIS` `SUSP` `RETU` `MOVE` | **niêm yết mới · đình chỉ · trở lại · chuyển sàn** | mục 3.8 + **ngày niêm yết** |
| `AIS` `MA` `OTHE` | khác | — |

**17 trường** (mẫu FPT, ESOP 2026):

| Trường | Mẫu | Ý nghĩa | Dùng |
|---|---|---|:--:|
| `id` | `6a309747e56b40f0…` | khoá của nhà cung cấp | ✅ |
| `ticker` · `organCode` | `FPT` | mã | ✅ |
| `eventCode` | `ISS` | mã loại sự kiện (bảng trên) | ✅ |
| `category` | `DIVIDEND` | nhóm lớn | ✅ |
| `eventNameVi` / `En` | `Phát hành cổ phiếu` | tên loại | ✅ |
| **`eventTitleVi`** | `Phát hành cổ phiếu - Phát hành cho CBCNV tỉ lệ 0.1%` | 🔑 **tiêu đề — nguồn duy nhất để phân loại** STOCK_DIV / RIGHTS / SPLIT / ESOP | ✅ |
| `eventTitleEn` | `Share Issue - ESOP ratio 0.1%` | nt, tiếng Anh | ⬜ |
| **`exrightDate`** | `2026-06-24` | 🔑 **ngày GDKHQ** — mốc nhảy của hệ số | ✅ |
| `recordDate` | `2026-06-24` | ngày chốt danh sách | ⬜ |
| `publicDate` | `2026-06-29` | ngày công bố | ⬜ |
| **`exerciseRatio`** | `0.00135` | 🔑 **tỷ lệ thực hiện** | ✅ |
| `displayDate1` `displayDate2` | `2026-06-24` · `2026-06-29` | ngày hiển thị | ⬜ |
| `organNameVi` / `En` | `Công ty Cổ phần FPT` | tên công ty | ⬜ |

> **`value_per_share`** (cổ tức tiền, VND/cp) xuất hiện với sự kiện `DIV`, **`NaN` với
> `ISS`** — phân loại phải theo `eventTitleVi`, không theo sự có mặt của trường.
>
> **KHÔNG có trường giá phát hành của RIGHTS** ⇒ mục 3.6 vẫn không nguồn nào cấp.
>
> **Đối chứng đã chạy:** hệ số suy từ `exerciseRatio` **khớp** hệ số đo từ
> `ClosePriceAdjusted/ClosePrice` ở 4/4 mã (PVD `0.5992` · VTP `0.8521` · HPG `0.8927`
> và `0.7439` · MBB `0.9615`).

---

<a id="d8"></a>
## D8 — Giao dịch nội bộ & cổ đông lớn · VCI IQ `insider-transaction` 🆕

| | |
|---|---|
| **Endpoint** | `GET .../v1/company/{ticker}/insider-transaction` · `page` `size` |
| **Tần suất** | hằng ngày |
| **Quy mô** | FPT: **273** bản ghi |
| **Trạng thái** | 🆕 **chưa ai dùng** — đóng mục 2.4 |

**35 trường** — nhóm chính (mẫu FPT):

| Trường | Mẫu | Ý nghĩa |
|---|---|---|
| `ticker` · `organCode` | `FPT` | mã |
| `eventCode` | `DDIND` | loại: cá nhân / tổ chức / liên quan |
| `eventNameVi` | `Giao dịch nội bộ: Giao dịch cá nhân` | mô tả loại |
| **`traderNameVi`** / `En` | `Nguyễn Văn Khoa` | **người/tổ chức giao dịch** |
| **`traderPositionVi`** / `En` | `Tổng Giám đốc` | **chức vụ** |
| `traderPersonId` | `35135` | định danh người |
| **`actionTypeCode`** · `actionTypeVi` / `En` | `B` · `Mua` / `Buy` | **chiều giao dịch** |
| **`shareRegister`** | `428368` | **KL đăng ký** |
| **`shareAcquire`** | `428368` | **KL thực hiện được** (≠ đăng ký = tín hiệu) |
| `shareBeforeTrade` · `shareAfterTrade` | `5414610` · `5842978` | sở hữu trước / sau |
| `ownershipAfterTrade` | `0.0034` | tỷ lệ sở hữu sau (0,34%) |
| **`tradeStatusVi`** / `En` | `Đã thực hiện xong` / `Done` | **trạng thái** — lọc giao dịch mới đăng ký vs đã xong |
| `startDate` · `endDate` | `2026-06-24` | khoảng đăng ký giao dịch |
| `publicDate` | `2026-06-29` | ngày công bố |
| `sourceUrlVi` | `http://fiinpro.com/News/Detail/…` | nguồn gốc |
| `roleNameVi` · `relativeNameVi` | `Nguyễn Văn Khoa` | vai trò / người liên quan |
| `icbCodeLv1` | `9000` | ngành cấp 1 |

---

<a id="d9"></a>
## D9 — Cổ đông lớn · VCI IQ `shareholder` 🆕

| | |
|---|---|
| **Endpoint** | `GET .../v1/company/{ticker}/shareholder` |
| **Tần suất** | tháng |
| **Quy mô** | FPT: **73** bản ghi |
| **Công dụng thêm** | 🔑 **xác nhận bảng hệ sinh thái (7.8) bằng dữ liệu** thay vì bằng trí nhớ — dò sở hữu chéo giữa các mã |

**10 trường** (mẫu FPT):

| Trường | Mẫu | Ý nghĩa |
|---|---|---|
| `ownerName` / `ownerNameEn` | `Trương Gia Bình` | tên chủ sở hữu |
| `ownerCode` | `536` | định danh — **khoá để dò sở hữu chéo** |
| `positionName` / `En` | `Chủ tịch Hội đồng Quản trị` | chức vụ (nếu là nội bộ) |
| `quantity` | `117347966` | số cổ phiếu nắm giữ |
| `percentage` | `0.0689` | tỷ lệ sở hữu (6,89%) |
| `ownerType` | `INDIVIDUAL` | `INDIVIDUAL` \| tổ chức |
| `updateDate` | `2026-02-03T10:36:48` | lần cập nhật gần nhất |
| `publicDate` | `2025-12-31` | ngày dữ liệu có hiệu lực |

---

<a id="d10"></a>
## D10 — Hồ sơ công ty · VCI IQ `/v1/company` 🔑🆕

**Đóng hai mục trước đó tưởng không nguồn nào có: `listingDate` (7.3) và `freeFloat` (5.5)** — E0023.

| | |
|---|---|
| **Endpoint** | `GET .../v1/company/{ticker}` |
| **Tần suất** | tuần (phần tĩnh) — các trường giá/định giá đổi hằng ngày |

**41 trường** — nhóm chính (mẫu FPT):

| Trường | Mẫu | Ý nghĩa | Đóng mục |
|---|---|---|---|
| **`listingDate`** | `2006-12-13` | 🔑 **NGÀY NIÊM YẾT** | **7.3** |
| **`freeFloat`** | `1457177458` | 🔑 số cổ phiếu tự do chuyển nhượng | **5.5** |
| **`freeFloatPercentage`** | `0.85` | 🔑 tỷ lệ free float | **5.5** |
| `numberOfSharesMktCap` | `1714326422` | số CP tính vốn hoá | 5.4 |
| `marketCap` | `108002564586000` | vốn hoá (VND) | 5.6 |
| `currentPrice` | `63000` | giá hiện tại | — |
| `foreignerPercentage` | `0.269` | tỷ lệ sở hữu NN hiện tại | — |
| `maximumForeignPercentage` | `0.49` | **trần room ngoại** | — |
| `statePercentage` | `0.0567` | tỷ lệ sở hữu nhà nước | — |
| `sector` / `sectorVn` | `Technology` / `Công nghệ Thông tin` | ngành | 7.2 |
| `icbCodeLv2` · `icbCodeLv4` | `9500` · `9537` | mã ngành ICB | 7.2 |
| `comGroupCode` | `VNINDEX` | nhóm sàn | — |
| `comTypeCode` | `CT` | loại hình DN | — |
| `averageMatchValue1Month` | `535059680718` | **GTGD bình quân 1 tháng** — dùng cho ADTV | — |
| `averageMatchVolume1Month` | `7755993` | KLGD bình quân 1 tháng | — |
| `highestPrice1Year` · `lowestPrice1Year` | `108948` · `61500` | đỉnh/đáy 52 tuần | — |
| `rating` · `ratingAsOf` · `analyst` | `BUY` · `08-Jun-26` · `Ngan Ly` | **khuyến nghị của Vietcap** | 8.1 |
| `targetPrice` · `upsideToTargetPercent` | `90300` · `0.433` | giá mục tiêu | 8.1 |
| `dividendPerShareTsr` · `projectedTSRPercentage` | `2300` · `0.470` | cổ tức/cp, TSR dự phóng | — |
| `prevInsight` | `{targetPrice: 116600, …}` | khuyến nghị **kỳ trước** | — |
| `profile` / `enProfile` | `<div>…` | giới thiệu DN (HTML) | — |
| `isBank` · `bank` · `listing` · `inCu` | `False` · `True` | cờ phân loại | — |

> ⚠️ Bảng này **trộn dữ liệu tĩnh (ngày niêm yết) với dữ liệu đổi hằng ngày (giá, vốn hoá,
> khuyến nghị)**. Trong `obs.security_ref`, `observed_at` giữ đúng vai trò tách hai loại đó.

---

<a id="d11"></a>
## D11 — Chỉ số tài chính · VCI IQ `statistics-financial`

| | |
|---|---|
| **Endpoint** | `GET .../v1/company/{ticker}/statistics-financial` |
| **Tần suất** | quý |
| **Phạm vi** | **từ 2018** · FPT: 41 bản ghi (theo quý) |
| **Quy mô** | **60 trường/bản ghi** |

Nhóm chính (mẫu FPT 2018Q1):

| Nhóm | Trường |
|---|---|
| Định danh kỳ | `year` `quarter` `yearReport` `organCode` `ratioTTMId` `ratioType`(`RATIO_TTM`) |
| Định giá | `pe` `pb` `ps` `priceToCashFlow` `evToEbitda` `marketCap` `numberOfSharesMktCap` `dividendYield` |
| Thanh khoản | `cashRatio` `quickRatio` `currentRatio` |
| Đòn bẩy | `debtPerEquity` `debtToEquity` `financialLeverage` `ownersEquity` |
| Sinh lời | `roe` `roa` `roic` `grossMargin` `ebitMargin` `preTaxProfitMargin` `afterTaxProfitMargin` |
| Hiệu quả | `assetTurnover` `fixedAssetTurnover` `daySaleOutstanding` `daysInventoryOutstanding` `daysPayableOutstanding` `cashCycle` |
| Tuyệt đối | `ebit` `ebitda` |
| **Riêng ngân hàng** | `netInterestMargin` `costToIncome` `cir` `npl` `loansGrowth` `depositGrowth` `ldrLoanDepositRatio` `car` `casaRatio` `loansLossReservesToNPLs` `provisionToOutstandingLoans` `equityToLoans` … |

> ⚠️ **Trường ngân hàng trả `0.0` (không phải `NULL`) với doanh nghiệp phi ngân hàng** —
> FPT có `npl = 0.0`, `netInterestMargin = 0.0`. Đưa thẳng vào feature sẽ tạo **0 giả**.
> Phải lọc theo `isBank` của D10.

---

<a id="d12"></a>
## D12 — Báo cáo tài chính · VCI IQ `financial-statement`

> **Đo lại và xác minh 2026-08-19** bằng lời gọi thật (FPT · VCB · CTG · BVH · HCM · VND ·
> HPG · VNM · REE). Mục này thay bản cũ — bản cũ mô tả thiếu cơ chế khuôn theo ngành và
> dẫn tới kết luận sai về độ phủ ngân hàng.

| | |
|---|---|
| **Endpoint số liệu** | `GET https://iq.vietcap.com.vn/api/iq-insight-service/v1/company/{ticker}/financial-statement` |
| **Endpoint định nghĩa** | `GET .../v1/company/{ticker}/financial-statement/metrics` |
| **Auth** | không — chỉ `User-Agent` + `Referer: https://iq.vietcap.com.vn/` |
| **Tham số bắt buộc** | `section` ∈ `BALANCE_SHEET` \| `INCOME_STATEMENT` \| `CASH_FLOW` \| `NOTE` |
| **Phạm vi đo được** | năm **2018–2025** (8 bản ghi) · quý **2018Q1–2026Q2** (34 bản ghi) |
| **Tần suất nạp** | quý |

Phản hồi: `data.years` và `data.quarters`, mỗi phần tử là **một bản ghi kỳ** (không phải
một chỉ tiêu). Chỉ tiêu nằm ngang thành cột.

| Trường kỳ | Ý nghĩa |
|---|---|
| `yearReport` | năm |
| `lengthReport` | **`5` = cả năm · `1`–`4` = quý** (đo: `years` luôn `5`, `quarters` luôn 1–4) |
| `publicDate` | 🔑 **ngày công bố** — cơ sở duy nhất để dựng point-in-time |
| `createDate` · `updateDate` | dấu thời gian của nhà cung cấp |
| `organCode` · `ticker` | mã |

`publicDate` đo được: FPT 2024 → `2025-03-15` · HPG 2024 → `2025-03-27` · VCB 2024 →
`2025-03-29`. **Không có trường này thì mọi feature BCTC trong backtest đều lookahead.**

### 🔑 `metrics` là THEO TỪNG MÃ — đây là điểm dễ sai nhất

`/metrics` **không phải từ điển dùng chung**. Nó trả về **khuôn của đúng loại hình doanh
nghiệp đó**. Dùng khuôn của mã này để đọc số của mã khác sẽ ra kết luận sai.

| Mã | Loại hình | IS | BS | CF | NOTE |
|---|---|---|---|---|---|
| FPT | thường | 25 | 122 | 41 | 157 |
| VCB | ngân hàng | 26 | 88 | 57 | 219 |
| BVH | bảo hiểm | 84 | 152 | 49 | 281 |
| HCM · VND | chứng khoán | 80 | 212 | 153 | 642 |

**Tiền tố trường mã hoá loại hình:**

| Hậu tố | Loại hình | Ví dụ |
|---|---|---|
| `*a` | doanh nghiệp thường | `bsa1` `isa3` `cfa1` `noc1` |
| `*b` | **ngân hàng** | `bsb…` `isb27` (Thu nhập lãi thuần) `cfb…` `nob…` |
| `*i` | **bảo hiểm** | `bsi…` `isi…` `cfi…` `noi…` |
| `*s` | **chứng khoán** | `bss…` `iss…` `cfs…` `nos…` |

Khuôn của một mã **trộn nhiều tiền tố**: VCB có `bsb×53 + bsa×34`; HCM có
`bsa×114 + bss×49 + nos×43`. Nên **không được lọc theo tiền tố** — phải lấy đúng danh sách
`field` từ `/metrics` của chính mã đó.

**Bản ghi trả về chứa nhiều trường hơn khuôn**: dòng của FPT có 331 khoá `bs*` trong khi
khuôn chỉ định nghĩa 122. Các khoá thừa là chỗ dành cho khuôn ngành khác, để trống. Đọc
trường không có trong `/metrics` của mã đó là đọc rác.

### 8 trường mô tả mỗi chỉ tiêu (`/metrics`)

| Trường | Mẫu | Ý nghĩa |
|---|---|---|
| `field` | `bsa1` | **mã chỉ tiêu** — khoá join sang bản ghi số liệu |
| `name` | `BSA1` | mã viết hoa |
| `titleVi` / `titleEn` | `TÀI SẢN NGẮN HẠN` / `CURRENT ASSETS` | tên chỉ tiêu |
| `fullTitleVi` / `fullTitleEn` | nt | tên đầy đủ |
| `level` | `1` | cấp trong cây (1 = mục lớn) |
| `parent` | `null` | mã chỉ tiêu cha |

`level` + `parent` cho phép tái tạo **cây phân cấp BCTC** — cộng con phải ra cha, tức có sẵn
một phép kiểm nhất quán nội tại, miễn phí.

⚠️ Một số bản ghi `/metrics` có **`field = null`** (VCB 6 · HCM/VND 9 · BVH 1) — dòng tiêu đề
phân nhóm, không phải chỉ tiêu. Phải bỏ qua, nếu không sẽ vỡ khi join.

### Bẫy: lỗi trả về HTTP 200

Thiếu `section` hoặc `section` sai → **HTTP 200**, nhưng thân phản hồi có `status: 400`,
`msg: "Error"`, `data: null`. Kiểm `resp.status_code` là không đủ — **phải kiểm
`body["status"]`**.

### Độ phủ chỉ tiêu — đo 2024, đếm chỉ tiêu có giá trị

Đối chiếu với `vnfinancialdata` 0.1.2 (dataset trên Hugging Face, revision `v1.0.0`):

| Mã | Ngành | IS · VCI/vnfd | BS · VCI/vnfd | CF · VCI/vnfd | NOTE · VCI |
|---|---|---|---|---|---|
| VCB | ngân hàng | 25 / 25 | **66 / 39** | 36 / 36 | **141** |
| CTG | ngân hàng | 25 / 25 | **65 / 39** | 35 / 35 | **149** |
| BVH | bảo hiểm | 51 / 56 | 84 / 86 | 33 / 33 | **126** |
| HCM | chứng khoán | 38 / 40 | **78 / 56** | **54 / 41** | **121** |
| VND | chứng khoán | 46 / 49 | **96 / 69** | **74 / 50** | **148** |
| FPT | công nghệ | 24 / 26 | 83 / 81 | 33 / 33 | **83** |
| HPG | thép | 22 / 24 | 74 / 72 | 34 / 34 | **91** |
| VNM | tiêu dùng | 23 / 25 | 76 / 74 | 32 / 32 | **84** |
| REE | tiện ích | 24 / 26 | 74 / 72 | 32 / 32 | **78** |

**Kết luận: VCI ngang hoặc hơn ở mọi ngành**, hơn rõ ở bảng cân đối của khối tài chính, và
**độc quyền toàn bộ thuyết minh** — `vnfinancialdata` không có `NOTE` nào.

Giá trị trùng khớp **tới từng đồng** ở mọi chỉ tiêu đối chiếu được (FPT · HPG · VCB, 2023 và
2024). Hai nguồn xuất xứ khác nhau mà khớp tuyệt đối ⇒ tin được cả hai về mặt số.

### Vai trò của `vnfinancialdata`

Chỉ còn **một** lý do dùng: **lịch sử 2011–2017** mà VCI không có (VCI bắt đầu 2018).
Kèm ba hạn chế phải khai rõ khi nạp:

- **Không có `publicDate`** ⇒ không dựng được point-in-time; phải gán độ trễ giả định và
  đánh dấu là giả định.
- **Chỉ theo năm**, không có quý.
- **Độ phủ ~50% HOSE** (386 mã HSX + 307 HNX; HOSE có ~736 mã niêm yết). Thiếu theo mã chứ
  không theo ngành — ví dụ **SSI hoàn toàn không có**, trong khi HCM/VND/VCI đều có.

Bộ mã chỉ tiêu của nó (`is_doanh_so_thuan`…) **khác hoàn toàn** bộ `isa*`/`isb*` của VCI —
trộn hai nguồn cần bảng ánh xạ, và với ngân hàng thì không ánh xạ 1-1 được.

---

<a id="d13"></a>
## D13 — Ngành ICB · VCI IQ `sectors/icb-codes`

| | |
|---|---|
| **Endpoint** | `GET .../v1/sectors/icb-codes` |
| **Tần suất** | tháng |
| **Quy mô** | **177 ngành** |

| Trường | Mẫu | Ý nghĩa |
|---|---|---|
| `name` | `1755` | **mã ICB** |
| `enSector` / `viSector` | `Nonferrous Metals` / `Kim Loại màu` | tên ngành |
| `icbLevel` | `4` | cấp phân ngành (1–4) |
| `marketCap` | `2379941667200` | tổng vốn hoá ngành |
| `isLevel1Custom` · `level1Custom` | `False` | cờ phân ngành tuỳ biến |

---

<a id="d14"></a>
## D14 — Tin tức / CBTT · VCI IQ `news-events-for-chart` 🆕

| | |
|---|---|
| **Endpoint** | `GET .../v1/news-events-for-chart` · `ticker` `fromDate` `toDate` `languageId=1` `eventCode` |
| **Tần suất** | hằng ngày |
| **Quy mô** | FPT 2025→nay: **189** bản ghi |

| Trường | Mẫu | Ý nghĩa |
|---|---|---|
| `id` | `677b1f2a7eee280b39cf01bc` | khoá |
| `newsTitle` | `FPT: Thông báo về việc niêm yết và giao dịch cổ phiếu thay đổi…` | tiêu đề |
| `newsShortContent` | *(như trên)* | tóm tắt |
| `publicDate` · `displayDate` | `2025-01-03T18:27:00` | thời điểm công bố |
| `isEvent` · `event` | `False` | tin thường hay sự kiện |

---

<a id="d15"></a>
## D15 — Lãi suất liên ngân hàng, LỊCH SỬ · trolyluat.vn

| | |
|---|---|
| **Endpoint** | `POST https://trolyluat.vn/api/sbv-interest-rates/chart-data` |
| **Body** | `{start_date, end_date, terms:[{value,column,…}], datetime:"YYYYMMDDHHmmss"}` |
| **Tần suất** | **1 lần** — đây là nguồn lịch sử, không phải nguồn hằng ngày |
| **Phạm vi** | **2011-01-04 → 2026-07-27** · **3.854 dòng** · qua đêm **0 dòng thiếu** |

**Giải mã phản hồi** (AES-256-GCM):
```
key        = b64decode("ux8n7q7mJd9a3YF9h+9J7C6Y5nqf" + datetime + "U=")     32 byte
iv         = b64decode(data.a)
ciphertext = b64decode(data.b) + b64decode(data.c)      # c = auth tag
```
Khoá suy từ **chính trường `datetime` mình gửi lên** ⇒ tự tính được, không có bí mật phía
server. Ghi chú *"không cào được bằng HTTP thuần"* trong `backfill_interbank_trolyluat.py`
là **sai** (E0022).

| Trường | Mẫu | Ý nghĩa |
|---|---|---|
| `id` | `2640` | khoá bản ghi |
| `date` | `04/01/2011` | ngày, **dd/mm/yyyy** |
| `rate_overnight` | `10.5241` | **lãi suất qua đêm (%)** — 0 dòng thiếu |
| `rate_1_week` `rate_2_week` | `10.9735` `11.2511` | kỳ hạn 1/2 tuần |
| `rate_1_month` `rate_3_month` | `12.7214` `14.2854` | kỳ hạn 1/3 tháng |
| `rate_6_month` `rate_9_month` | `None` | 🔴 **thiếu 1.341 dòng — thiếu TẠI NGUỒN**, không phải lỗi cào |

> **KHÔNG có doanh số (turnover)** — phần đó chỉ SBV có (D16).

---

<a id="d16"></a>
## D16 — Lãi suất SBV, HẰNG NGÀY · sbv.gov.vn

| | |
|---|---|
| **URL** | `https://sbv.gov.vn/vi/l%C3%A3i-su%E1%BA%A5t1` — **chính là URL đã có trong `macro_daily.py:78`** |
| **Cách lấy** | HTML server-rendered, parse bằng BeautifulSoup |
| **Tần suất** | hằng ngày |
| **Phạm vi** | 🔴 **chỉ ảnh hiện tại** — không bộ chọn ngày, không trang lịch sử ⇒ **không chạy được ngày nào là mất vĩnh viễn ngày đó** |

**Trang có HAI bảng:**

| | Cột | Mẫu | Trạng thái |
|---|---|---|---|
| **Bảng 1** — lãi suất **điều hành** | `Loại lãi suất` · `Giá trị` · `Văn bản quyết định` · `Ngày áp dụng` | `Lãi suất tái chiết khấu` · `3,000%` · `1123/QĐ-NHNN ngày 16/03/2023` · `19/03/2023` | 🔴 **scraper BỎ QUA** ⇒ toàn bộ nguyên nhân mục 6.3 rỗng |
| **Bảng 2** — liên ngân hàng | `Thời hạn` · `Lãi suất BQ liên NH` · `Doanh số (Tỷ đồng)` | `Qua đêm` · `2,17` · `815,007,0` | ✅ đang đọc |

> **Bẫy định dạng:** số dùng **dấu phẩy làm thập phân** (`3,000%` = 3,000% chứ không phải
> ba nghìn). Doanh số `815,007,0` định dạng lộn xộn — phải chuẩn hoá cẩn thận.
>
> Dòng gắn `(*)` tham chiếu ngày **CŨ hơn** ngày áp dụng ⇒ bỏ qua thay vì gán sai ngày.

---

<a id="d17"></a>
## D17 — Hàng hoá · Yahoo Finance

| | |
|---|---|
| **Endpoint** | `GET https://query2.finance.yahoo.com/v8/finance/chart/{ticker}` |
| **Tần suất** | hằng ngày |
| **Quy mô** | **15 mã futures** |

| Nhóm | Mã | Đơn vị |
|---|---|---|
| Kim loại | `GC=F` vàng · `SI=F` bạc · `PL=F` bạch kim · `HG=F` đồng · `TIO=F` quặng sắt | USD/oz, USD/lb, USD/tấn |
| Năng lượng | `CL=F` dầu WTI · `BZ=F` dầu Brent · `NG=F` khí tự nhiên | USD/thùng, USD/MMBtu |
| Nông sản | `ZC=F` ngô · `ZS=F` đậu tương · `ZW=F` lúa mì · `ZR=F` gạo · `SB=F` đường · `KC=F` cà phê · `CT=F` bông | USX/bushel, USD/cwt, USX/lb |

> ⚠️ **Đơn vị lẫn lộn giữa các mã** — `USX` (cent Mỹ) với ngô/đậu/lúa mì/đường/cà phê/bông,
> `USD` với vàng/dầu/gạo. Bắt buộc khai đơn vị theo từng mã, đúng luật L2 của `DB_DESIGN.md`.

---

## Phụ lục — lịch chạy tổng hợp

| Thời điểm | Việc | Số lời gọi |
|---|---|---|
| **16:15 ICT hằng ngày** | D1 giá+khối ngoại (204) · D4 chỉ số+breadth (~5) · D7 sự kiện delta (204) · D8 nội bộ (204) · D14 tin (204) · D16 SBV (1) · D17 hàng hoá (15) | ~840 |
| **sau giờ đóng cửa** | D3 nến 1 phút (204) | 204 |
| **hằng tuần** | D5 danh mục mã (4) · D6 chi tiết CK (204) · D10 hồ sơ DN (204) | ~410 |
| **hằng tháng** | D9 cổ đông lớn (204) · D13 ngành ICB (1) | 205 |
| **hằng quý** | D11 chỉ số TC (204) · D12 BCTC (204) | 408 |
| **một lần** | D2 lịch sử DNSE (204) · D15 lãi suất lịch sử (1) · D1 backfill SSI ~5,6h ⚠️ | — |

⚠️ = cần duyệt riêng trước khi chạy.

---

## Bảo trì

- Thêm dataset → thêm mục `D##` + dòng ở mục lục + dòng ở phụ lục lịch chạy.
- Đổi endpoint/trường → cập nhật **cùng commit** với code, kèm mã bằng chứng `E####`.
- Nghi nguồn đổi API → chạy lại `probe_sources.py` · `verify_sources_claims.py`.
- Mọi giá trị mẫu ở đây gắn với ngày đo **2026-07-30**; đo lại thì ghi ngày mới, đừng sửa đè.
