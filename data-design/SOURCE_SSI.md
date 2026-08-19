# Nguồn SSI — trường khả dụng

> Lập lại 2026-08-19 **từ nguồn v2**: `INGEST_SPEC.md` (D1, D3–D6) + code thật trong
> `3. Build\code\` + schema. **Không dùng** `PRJ quant\06_document\DATA_SOURCES.md` — tài
> liệu đó mô tả pipeline v1 và cột "đang dùng" của nó không áp cho v2.
>
> Cột **v2 ghi vào đâu** đọc trực tiếp từ code, không chép từ đặc tả. Chỗ nào đặc tả và code
> lệch nhau đều ghi rõ ở §Chênh lệch.

**Bản chất: nguồn RAW duy nhất của hệ.** Không nguồn nào khác cho O/H/L chưa điều chỉnh.
Đổi lại chỉ sâu tới 2020-02-26 và cần token.

## Phiên bản API

Ghi lại để khi SSI ra bản mới còn biết mình đang ở đâu.

| Bề mặt | Host | Version trong đường dẫn | Auth |
|---|---|---|---|
| FastConnect | `fc-data.ssi.com.vn` | **`/api/v2/`** — hiện hành, không có v1 hay v3 | Bearer token, sống ~8h |
| iBoard | `iboard-api.ssi.com.vn` | **không có đoạn version** | không |

⚠️ iBoard không đánh version, nên **đổi hành vi sẽ không kèm đổi URL**. Đây là bề mặt cần
canh: dạng phản hồi có thể đổi im lặng. FastConnect ít nhất còn cho tín hiệu qua `/api/vN/`.

Token FastConnect và `.env` hiện lấy từ **project v1** — xem §Phụ thuộc v1.

---

# Cửa nào v2 thực sự gọi

| Cửa | Script v2 | Bảng đích |
|---|---|---|
| `api/v2/Market/DailyStockPrice` | `load_ssi_fc_price.py` | `obs.price_daily` (`source='SSI'`) |
| `api/v2/Market/IndexComponents` | `load_index_member.py` | thành phần rổ, chụp hằng ngày |
| iBoard `statistics/charts/history` | `load_iboard.py` | `obs.price_daily` (nguồn giá thứ hai) |
| iBoard `statistics/charts/history` | `load_index.py` | `obs.index_daily` → lịch giao dịch |
| iBoard `statistics/company/stock-price` | `load_ssi_sp.py` | `obs.price_daily` (close thô + phụ trợ) |

**Chưa cài trong v2:** `IntradayOhlc` · `DailyIndex` · `Securities` · `SecuritiesDetails`.

---

# 1. `DailyStockPrice` — FastConnect, 31 trường

| | |
|---|---|
| Endpoint | `GET https://fc-data.ssi.com.vn/api/v2/Market/DailyStockPrice` |
| Tham số | `symbol` · `fromDate` `toDate` (**dd/mm/yyyy**) · `pageIndex` · `pageSize` |
| Giới hạn cứng | **30 ngày/lời gọi** (code v2 dùng bước 28) · từ chối `toDate ≥ hôm nay` · `pageSize` **chỉ nhận** `10/20/50/100/1000` |
| Phạm vi | **2020-02-26** → nay, mốc **cố định không cuộn** (E0012) |
| Đơn vị giá | **VND nguyên** |
| Nhịp | hằng ngày 16:15 ICT · 204 mã ≈ 4,5 phút (~1 lời gọi/giây) |
| Nạp lần đầu | 204 mã × ~78 chunk ≈ **5,6 giờ** — cần duyệt riêng |
| Ngày không giao dịch | **có trả dòng**, `volume = 0` |
| Timeout đo được | 0 / 261 mã |

Mẫu ACB 28/07/2026. Cột **v2 ghi vào đâu** theo `load_ssi_fc_price.py`; `extra.*` là khoá
trong cột JSON `extra` của `obs.price_daily`.

| Trường | Mẫu | Ý nghĩa | v2 ghi vào đâu |
|---|---|---|---|
| `Symbol` | `ACB` | mã | khoá |
| `TradingDate` | `28/07/2026` | ngày, **dd/mm/yyyy** | khoá |
| `Time` | `None` | luôn rỗng với dữ liệu ngày | — |
| `Market` | `HOSE` | sàn **tại thời điểm gọi** | — |
| `OpenPrice` | `22150` | mở cửa — RAW | cột `open` |
| `HighestPrice` | `22650` | cao nhất — RAW | cột `high` |
| `LowestPrice` | `22100` | thấp nhất — RAW | cột `low` |
| `ClosePrice` | `22500` | **đóng cửa THÔ** — nguồn raw duy nhất trong 3 API | cột `close` |
| `ClosePriceAdjusted` | `22500` | đóng cửa đã điều chỉnh cho mọi sự kiện sau ngày đó | `extra.close_adj` |
| `AveragePrice` | `22650` | giá bình quân phiên | `extra.avg_price` |
| `RefPrice` | `22300` | tham chiếu (đóng cửa phiên trước) | cột `ref_price` |
| `CeilingPrice` | `23850` | **trần** (+7% HOSE) | `extra.ceiling` |
| `FloorPrice` | `20750` | **sàn** (−7%) | `extra.floor` |
| `PriceChange` | `200` | thay đổi tuyệt đối (VND) | — (suy được) |
| `PerPriceChange` | `0.90` | thay đổi (%) | — (suy được) |
| `TotalMatchVol` | `15653800` | **KL khớp lệnh** (cổ phiếu) | cột `volume` |
| `TotalMatchVal` | `350277960000` | **GT khớp lệnh** (VND) — cần cho ADTV tính bằng tiền | cột `value` |
| `TotalDealVol` | `0` | KL **thoả thuận** | `extra.deal_vol` |
| `TotalDealVal` | `0` | GT thoả thuận (VND) | `extra.deal_val` |
| `TotalTradedVol` | `15653800` | tổng = khớp + thoả thuận | — (suy được) |
| `TotalTradedValue` | `350277960000` | tổng GT | — (suy được) |
| `ForeignBuyVolTotal` | `1959400` | KL nước ngoài **mua** | `extra.fbuy_vol` |
| `ForeignSellVolTotal` | `1188400` | KL nước ngoài **bán** | `extra.fsell_vol` |
| `ForeignBuyValTotal` | `43753445000` | GT ngoại mua (VND) | `extra.fbuy_val` |
| `ForeignSellValTotal` | `26532175500` | GT ngoại bán (VND) | `extra.fsell_val` |
| `ForeignCurrentRoom` | `323269297` | **room ngoại còn lại** (số cổ phiếu) | `extra.froom` |
| `NetBuySellVol` | `771000` | KL mua ròng NN | — (suy được từ mua − bán) |
| `NetBuySellVal` | `17221269500` | GT mua ròng NN | — (suy được) |
| `TotalBuyTrade` | `0` | số **lệnh** đặt mua | `extra.buy_trades` |
| `TotalBuyTradeVol` | `0` | KL đặt mua | — |
| `TotalSellTrade` | `0` | số lệnh đặt bán | `extra.sell_trades` |
| `TotalSellTradeVol` | `0` | KL đặt bán | — |

**v2 ghi 20 trường dữ liệu + 2 khoá** — 7 vào cột riêng, 13 vào `extra`.

Nhóm khối ngoại (mục 2.1 · 2.2 · 2.3), trần/sàn/tham chiếu (1.7), giá bình quân (1.8) và
thoả thuận (1.9) **đều đã được lấy**, dù `DATA_REQUIREMENTS.md` còn đánh 🔴 cho các mục này.

Mười trường không ghi chia hai loại: **suy được** (`PriceChange` `PerPriceChange`
`TotalTradedVol` `TotalTradedValue` `NetBuySellVol` `NetBuySellVal`) và **thật sự bỏ**
(`Market` `Time` `TotalBuyTradeVol` `TotalSellTradeVol`).

### Cặp `ClosePrice` / `ClosePriceAdjusted`

Tỷ lệ `adj/raw` là **hàm bậc thang nhảy tại mỗi ex-date**, bằng tích hệ số mọi sự kiện sau
ngày đó; ngày gần nhất luôn `1.0`. Quét 18 tháng, 102 điểm mỗi mã:

```
PVD   {0.5992, 1.0}              1 sự kiện
VTP   {0.8521, 1.0}              1 sự kiện
HPG   {0.7439, 0.8927, 1.0}      2 sự kiện
MBB   {0.9615, 1.0}              1 sự kiện
```

Cả hai chuỗi nhất quán nội tại ⇒ cặp này **đối chứng được hệ số `market.ca`** mà không cần
nguồn CA thứ hai. `load_ssi_sp.py` đã dựng sẵn chuỗi tỷ lệ này (biến `chuoi_F`).

---

# 2. iBoard `statistics/charts/history`

| | |
|---|---|
| Endpoint | `GET https://iboard-api.ssi.com.vn/statistics/charts/history` |
| Tham số | `resolution=1D` · `symbol` · `from` `to` (**unix timestamp**) |
| Header bắt buộc | `User-Agent` trình duyệt + `Referer: https://iboard.ssi.com.vn/` |
| Auth | không |
| Phạm vi | **2000-07-28** — đúng ngày HOSE mở cửa (E0053) |
| Cơ sở giá | **ADJUSTED** |

Phản hồi dạng TradingView — `data` là **dict các mảng song song**, không phải danh sách bản ghi:

| Khoá | Ý nghĩa | v2 dùng |
|---|---|---|
| `t` | mảng unix timestamp | ✅ |
| `o` `h` `l` `c` | mảng giá — **đã điều chỉnh** | ✅ |
| `v` | mảng khối lượng | ✅ |

Hai script dùng chung cửa này với hai mục đích khác nhau:

- `load_iboard.py` → `obs.price_daily`, **nguồn giá thứ hai**. Không thay DNSE, không xoá gì.
  Mọi dòng phải đi qua đúng hai cổng của job hằng ngày (C-01, C-02) — nạp lịch sử mà bỏ cổng
  thì kho có hai chuẩn, đúng bệnh của v1.
- `load_index.py` → `obs.index_daily`, và **lịch giao dịch dẫn xuất từ đây**. VNINDEX có từ
  2000-07-28 nên lịch không có danh sách ngày nghỉ nào để hết hạn.

---

# 3. iBoard `statistics/company/stock-price`

| | |
|---|---|
| Endpoint | `GET https://iboard-api.ssi.com.vn/statistics/company/stock-price` |
| Auth | không |
| Cơ sở giá | **TRỘN THANG trong cùng một dòng** — xem cạm bẫy |

**Đang dùng trong v2** (`load_ssi_sp.py`), nhưng **không** làm nguồn OHLC. Trường ở đây là
**camelCase**, khác hẳn FastConnect.

| Trường | Ý nghĩa | v2 ghi vào đâu |
|---|---|---|
| `closePrice` | đóng cửa **THÔ** | cột `close` |
| `openPrice` `highestPrice` `lowestPrice` | O/H/L **ĐÃ ĐIỀU CHỈNH** | `extra.open_adj` `high_adj` `low_adj` |
| `closePriceAdjusted` | đóng cửa đã điều chỉnh | `extra.close_adj` |
| `ceilingPrice` `floorPrice` | trần / sàn | `extra.ceiling` `extra.floor` |
| `refPrice` | tham chiếu | cột `ref_price` |
| `totalMatchVol` `totalMatchVal` | KL / GT khớp | cột `volume` · `value` |
| `totalDealVol` `totalDealVal` | thoả thuận | `extra.deal_vol` `extra.deal_val` |
| `foreignBuyVolTotal` `foreignSellVolTotal` | KL ngoại mua / bán | `extra.fbuy_vol` `extra.fsell_vol` |
| `foreignBuyValTotal` `foreignSellValTotal` | GT ngoại mua / bán | `extra.fbuy_val` `extra.fsell_val` |
| `foreignCurrentRoom` | room ngoại | `extra.froom` |
| `totalBuyTrade` `totalSellTrade` | số lệnh mua / bán | `extra.buy_trades` `extra.sell_trades` |

Code đặt tên `open_adj` / `high_adj` / `low_adj` chính là cách xử lý việc trộn thang: giữ
đúng bản chất từng trường thay vì gộp thành một bộ OHLC giả.

---

# 4. `IndexComponents` — FastConnect

| | |
|---|---|
| Endpoint | `GET .../api/v2/Market/IndexComponents` |
| Trả về | `IndexCode` `IndexName` `Exchange` `TotalSymbolNo` `IndexComponent[]` (`Isin` + `StockSymbol`) |
| v2 dùng | `load_index_member.py` — **chụp ảnh rổ hằng ngày** |

> ⚠️ **Không có API nào cho thành phần rổ theo NGÀY.** Đo 2026-08-07: `IndexComponents` bỏ
> qua `fromDate` / `toDate` / `effectiveDate`, **luôn trả rổ hiện tại**. Mỗi ngày không chụp
> là mất vĩnh viễn một ảnh — HOSE đảo rổ 2 lần/năm và không công bố lại lịch sử.

---

# 5. Cửa chưa cài trong v2

Bốn cửa dưới đây có trong đặc tả nhưng **chưa có script nào gọi**. Ghi lại để biết mỗi cửa
đang khoá mục nào.

## `IntradayOhlc` — 7 trường, RAW

`GET .../api/v2/Market/IntradayOhlc` · `resolution=1`

| Trường | Mẫu | Ý nghĩa |
|---|---|---|
| `symbol` | `ACB` | mã |
| `time` | `2026-07-28 09:15:59` | **ICT, tz-naive** — lưu DB dạng UTC |
| `open` `high` `low` `close` | `22150` … | VND nguyên, **RAW** |
| `volume` | `176800` | KL khớp trong phút |

Xác minh raw: PVD 27/05/2026 nến cuối `30.150` = `ClosePrice` thô, không phải
`ClosePriceAdjusted` (`18.066`). Không có cột adjusted. Số nến/phiên **193–228**.
Nạp lại toàn bộ ≈ **128 giờ** (vòng lặp theo ngày, không batch thật) — cần duyệt riêng.

> **Luật bất di dịch:** intraday **chỉ append hôm nay**, cấm refetch quá khứ từ nguồn
> adjusted — nếu không sẽ tạo "seam điều chỉnh" giữa chuỗi. Bar quá khứ đổi giá = cờ đỏ.

## `DailyIndex` — 21 trường

`GET .../api/v2/Market/DailyIndex` · `indexId` `fromDate` `toDate` `pageSize∈{10,20,50,100,1000}`

27 chỉ số khả dụng (qua `IndexList`): VNINDEX · VN30 · VN100 · VNMIDCAP · VNSMALLCAP ·
VNDIAMOND · VNFINLEAD · VNX50 + 19 chỉ số ngành.

| Nhóm | Trường | Ghi chú |
|---|---|---|
| Định danh | `IndexId` `IndexName` `TradingDate` `Time` `TypeIndex` | `Time`, `TypeIndex` luôn rỗng |
| Giá trị | `IndexValue` `Change` `RatioChange` | `IndexValue` là **điểm số**, không phải VND |
| **Độ rộng** | **`Advances` `Declines` `NoChanges` `Ceilings` `Floors`** | mẫu VNINDEX 28/07: `189` `122` `53` `6` `6` |
| Thanh khoản | `TotalMatchVol` `TotalMatchVal` `TotalDealVol` `TotalDealVal` `TotalVol` `TotalVal` | toàn thị trường |
| Khác | `TotalTrade` `TradingSession` | `TotalTrade` trả 0 · `C` = đã đóng cửa |

**Năm trường độ rộng là toàn bộ mục 4.2**, và **không nguồn nào khác trong 3 API có**.
Feature `breadth` của Regime Gate đã khai trong `MASTER_PLAN.md` nhưng chưa từng có dữ liệu.

v2 hiện lấy chỉ số từ **iBoard `charts/history`** (`load_index.py`) — cửa đó cho O/H/L/C/V
của chỉ số nhưng **không cho độ rộng**. Nên độ rộng vẫn trống dù `DailyIndex` sẵn sàng.

## `Securities` — 4 trường

`market ∈ {HOSE, HNX, UPCOM, DER}` — đủ 4 sàn. 736 mã HOSE.
`Market` · `Symbol` · `StockName` (tiếng Việt không dấu) · `StockEnName`. Cấp mục 7.1.

## `SecuritiesDetails` — `RepeatedInfo`, 29 trường

3 trường ngoài (`RType` `ReportDate` `TotalNoSym`) + mảng `RepeatedInfo`:

| Trường | Mẫu (ACB) | Ý nghĩa |
|---|---|---|
| `Symbol` `SymbolName` `SymbolEngName` | `ACB` · `NGAN HANG TMCP A CHAU` | định danh |
| `SecType` `MarketId` `Exchange` | `S` = cổ phiếu · `HOSE` | phân loại |
| **`ListedShare`** | `5136656599.0` | **số cổ phiếu niêm yết** — mục 5.4 |
| **`LotSize`** | `100` | **lô giao dịch** — mục 7.4 |
| **`TickPrice1/2/3`** + **`TickIncrement1/2/3`** | `1`/`10` · `10000`/`50` · `50000`/`100` | **bảng bước giá 3 bậc** |
| `TickPrice4` / `TickIncrement4` | `None` | bậc 4, không dùng với cổ phiếu |
| `Isin` | `None` | 🔴 rỗng với ACB — độ phủ chưa rõ |
| `IssueDate` `MaturityDate` `FirstTradingDate` `LastTradingDate` | rỗng | 🔴 **rỗng với cổ phiếu** ⇒ ngày niêm yết phải lấy từ VCI `/v1/company` |
| `Issuer` `ContractMultiplier` `SettlMethod` `Underlying` `PutOrCall` `ExercisePrice` `ExerciseStyle` `ExcerciseRatio` | rỗng | phái sinh & chứng quyền |

Bảng bước giá 3 bậc là đầu vào trực tiếp cho mô hình **chi phí giao dịch / trượt giá** —
backtest hiện đang giả định giá liên tục.

---

# Chênh lệch giữa đặc tả và code

`INGEST_SPEC.md` D1 đánh dấu "Dùng" **không khớp** `load_ssi_fc_price.py`. Bảng ở §1 trên
theo **code**, vì code là cái đang chạy.

| Trường | Đặc tả D1 | Code v2 |
|---|---|---|
| `ClosePriceAdjusted` `AveragePrice` `RefPrice` `CeilingPrice` `FloorPrice` `TotalMatchVal` `TotalDealVol` `TotalDealVal` `TotalBuyTrade` `TotalSellTrade` | ⬜ chưa dùng | **có ghi** |
| `NetBuySellVol` `NetBuySellVal` | ✅ dùng | **không ghi** (suy được từ mua − bán) |

Mười hai trường lệch. Đây là việc của `test_spec_vs_code.py` — chưa rõ nó có phủ tới mức
từng trường không.

---

# Cạm bẫy đã đo

- **Bẫy `pageSize`.** Giá trị ngoài `{10,20,50,100,1000}` trả `status=Error, data=None`,
  rất dễ nhầm là lỗi mạng.
- **`Market` là sàn HIỆN TẠI, không phải sàn của ngày quá khứ.** Đã từng gây **141 báo động
  giả** "vượt biên độ HOSE" cho mã khi đó còn ở UPCOM.
- **`volume=0` không có nghĩa thiếu dữ liệu** — SSI trả dòng cho cả ngày không giao dịch
  (khác DNSE, bỏ hẳn). Phải đối chiếu lịch giao dịch mới phân biệt được.
- **`company/stock-price` trộn thang trong cùng một dòng**: `open/high/low` đã điều chỉnh,
  `closePrice` thô. `L ≤ closeAdj ≤ H` đúng **250/250**; `L ≤ close ≤ H` đúng **0/250**
  (E0056). Không dùng làm OHLC — v2 xử lý bằng cách đặt tên `*_adj`.
- **`FirstTradingDate` rỗng với cổ phiếu** — không dùng được cho ngày niêm yết (E0023).
- **`IndexComponents` không có chiều thời gian** — luôn trả rổ hiện tại.
- **`TotalBuyTrade` = 0 với ACB** — chưa rõ universal hay riêng mã. Đừng dựng feature trên
  nhóm trường lệnh trước khi đo.
- **Không cửa nào của SSI cho O/H/L thô trước 2020-02-26** (E0057).
- **"SSI = 9 endpoint" chỉ đúng trong phạm vi FastConnect.** iBoard nằm ngoài danh sách đó;
  hiểu nhầm chỗ này từng dẫn tới kết luận sai "SSI chỉ sâu tới 2020" (E0058).

---

# Ánh xạ sang schema

| Cửa | Bảng L0 | Khoá |
|---|---|---|
| `DailyStockPrice` | `obs.price_daily` (`source='SSI'`) | `(source, symbol, trade_date, observed_at)` |
| iBoard `charts/history` | `obs.price_daily` | như trên |
| iBoard `company/stock-price` | `obs.price_daily` | như trên |
| iBoard `charts/history` (chỉ số) | `obs.index_daily` | `(source, index_code, trade_date, observed_at)` |
| `IndexComponents` | thành phần rổ | ảnh chụp theo ngày |
| *(chưa cài)* `IntradayOhlc` | `obs.price_intraday` | `(source, symbol, ts, observed_at)` |
| *(chưa cài)* `Securities` · `SecuritiesDetails` | `obs.security_ref` | `(source, symbol, observed_at)` |

`market.px_raw` phân giải giá thô **chỉ từ FastConnect `DailyStockPrice`** và `closePrice`
của `company/stock-price`. Mọi thứ từ `charts/history` là adjusted, không được dùng làm thô.

`source_contract` bắt buộc khai `price_basis`. `meta.vendor_door` phải có dòng của cửa
trước khi khai nguồn (`050_vendor_door.sql`).

---

# Phụ thuộc v1

`load_ssi_fc_price.py` **không tự gọi HTTP** — nó nạp client của v1:

```python
load_dotenv(Path(r"C:\Users\OS\Lucius\Projects\PRJ quant") / ".env")
sys.path.insert(0, str(Path(r"C:\Users\OS\Lucius\Projects\PRJ quant") / "01_fetch"))
import ssi_fc
```

Hệ quả: token SSI, logic retry và cách phân trang của v2 **là của v1**, qua đường dẫn tuyệt
đối cứng. Đổi hoặc xoá `PRJ quant\01_fetch\ssi_fc.py` sẽ làm gãy job hằng ngày của v2.

Cộng thêm: v2 chưa có venv riêng, phải dùng interpreter
`C:\Users\OS\Lucius\Projects\PRJ quant\.venv\Scripts\python.exe` (ghi trong `README.md` cùng
thư mục này).
