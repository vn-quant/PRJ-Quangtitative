# OHLCV ngày — đặc tả vận hành

Phạm vi: **giá ngày của cổ phiếu thường**. Intraday, chỉ số, khối ngoại, BCTC ở tài liệu riêng.

> Đây là tài liệu **tra cứu để làm**. Không ghi tiến độ, không ghi lịch sử sửa đổi.
> Muốn biết *vì sao* một quy tắc lại như vậy: `..\archive\OHLCV_SPEC.md`.
> Muốn biết *phiên bản nào đổi gì*: `..\archive\CHANGELOG.md`.
> Trạng thái cài đặt của từng phép kiểm: chạy `3. Build\test_spec_vs_code.py`.

| § | Nội dung |
|---|---|
| [1](#1) | Ba đại lượng không được lẫn |
| [2](#2) | Nguồn |
| [3](#3) | Bảng và cột |
| [4](#4) | Công thức |
| [5](#5) | Tầng đọc — hỏi giá thế nào |
| [6](#6) | Luồng chạy hằng ngày |
| [7](#7) | Luồng nạp lần đầu |
| [8](#8) | 18 phép kiểm |
| [9](#9) | Xử lý lỗi |
| [10](#10) | Việc người phải làm |

---

<a id="1"></a>
## 1. Ba đại lượng không được lẫn

| Ký hiệu | Là gì | Tính chất |
|---|---|---|
| **`raw(d)`** | Giá khớp lệnh thật ngày `d`, đúng như sàn công bố hôm đó | **Bất biến.** Không sự kiện nào sau `d` làm nó đổi |
| **`factor(e)`** | Hệ số của sự kiện `e` = `P_ref / P_prev` | `≤ 1`. Bằng **1** với ESOP và phát hành riêng lẻ |
| **`adj(d, as_of)`** | Giá `d` quy về mặt bằng ngày `as_of` | **Đổi theo `as_of`** — không phải một con số duy nhất |

> `adj` không tồn tại độc lập. *"Giá điều chỉnh của FPT ngày 2019-05-02"* là câu thiếu nghĩa:
> phải hỏi **quy về mặt bằng ngày nào**. Mọi hàm trả `adj` đều nhận `as_of`.

---

<a id="2"></a>
## 2. Nguồn

| | SSI `DailyStockPrice` | SSI iBoard | DNSE `chart-api` | DNSE `senses` | VCI IQ |
|---|---|---|---|---|---|
| Nhãn `source` | `SSI_SP` | `SSI_IBOARD` | `DNSE` | — | `VND` (sự kiện) |
| `raw` | ✅ từ **2020-02-26** | ❌ | ❌ | — | ❌ |
| `adj` | ❌ | ✅ từ **2000-07-28** | ✅ từ **2012-03-20** | — | ❌ |
| Giá tham chiếu SÀN | ✅ qua `Ceiling`/`Floor` | ❌ | ❌ | — | ❌ |
| Sự kiện — `ex_date` | ❌ | ❌ | ❌ | ✅ gần như đủ | ⚠️ thiếu 115/2.931 |
| Sự kiện — tỷ lệ | ❌ | ❌ | ❌ | ⚠️ nằm trong text | ✅ có cấu trúc |
| Sự kiện — `issue_price` | ❌ | ❌ | ❌ | ✅ 92% RIGHTS | ❌ không bao giờ có |
| Độ sâu sự kiện | — | — | — | **2007** | 2010 |
| Đơn vị giá | VND | **kVND** (×1000) | **kVND** (×1000) | — | — |
| Volume có điều chỉnh | — | ❌ không | ❌ không | — | — |
| Ngày không GD | có dòng `vol=0` | — | bỏ hẳn | — | — |

**Đã loại, không gọi lại** (E0033): SSI không có corporate action (thử 9/9 endpoint) ·
Vietstock `/data/corporateaz` trả JSON rỗng · CafeF `LichSuKien.ashx` trả 15 byte rỗng.

### 2.1 Quy tắc chọn nguồn

> 🔑 **Một chuỗi giá = MỘT nguồn, khoá theo mã, cho toàn bộ lịch sử.**
> Hai nguồn có **mặt bằng điều chỉnh khác nhau** — rơi nguồn giữa chừng tạo bậc giả
> (SHP lệch 12 lần ở 2012, E0045). Cấm fallback theo từng ngày, cấm round-robin.

| Trường | Lấy từ | Vì |
|---|---|---|
| Giá **adj** | `SSI_IBOARD` nếu nó phủ ≥ `DNSE`, ngược lại `DNSE` — chốt một lần cho cả chuỗi mỗi mã | iBoard sâu tới 2000 và được VCI đối chứng; DNSE lệch >5% ở 16/133 mã trong 2–3 năm đầu chuỗi (E0045, E0047) |
| Giá **raw** | `SSI_SP` | nguồn duy nhất có giá thô |
| `ex_date` | DNSE senses | thiếu 3/1.824 vs VCI thiếu 115/2.931 |
| `issue_price` | DNSE senses | 92% RIGHTS có; VCI không bao giờ có |
| `ratio` · `cash_vnd` | **cả hai, đối chiếu** | không nguồn nào đúng hơn |
| Phân xử khi hai nguồn lệch | `Ceiling`/`Floor` của SSI | sàn công bố, không phải suy diễn |

**Thứ tự phân xử:**

```
1. ex_date ≥ 2020-02-26  →  so với factor_exchange (SÀN CÔNG BỐ)
                            nguồn nào khớp trong 1 bước giá  →  lấy nguồn đó
                            không nguồn nào khớp             →  UNKNOWN
2. ex_date <  2020-02-26  →  DNSE và VCI khớp trong 1%       →  lấy
                            lệch                             →  UNKNOWN, KHÔNG chọn bừa
3. chỉ một nguồn có       →  lấy, nhưng phải qua C-18 (dải hợp lý)
```

⚠️ **Gặp `429`/`5xx` thì chờ và thử lại CÙNG nguồn đó.** Nhảy sang nguồn khác là đổi
ngữ nghĩa dữ liệu giữa chừng.

### 2.2 Lấy sự kiện từ DNSE senses

```
GET https://api-bo.dnse.com.vn/senses-api/events?symbol=X&page=N
    không cần token · chỉ User-Agent + Referer
```

| # | Bẫy | Xử |
|---|---|---|
| 1 | `size`/`limit`/`pageSize`/`per_page` **đều bị bỏ qua** — trần cứng 20/trang | phân trang tới khi rỗng |
| 2 | Phân trang **trả trùng** — FPT 96 bản ghi thô → 70 duy nhất | khử trùng theo nội dung |
| 3 | Bản tin CBTT trộn lẫn — tiền tố `MÃ: MÃ: Thông báo…`, ngày là ĐKCC = ex_date+1, không có số | loại `^[A-Z0-9]{3}:\s` và `Niêm yết và giao dịch`. 320/3.896 bản ghi |
| 4 | `ex_date = 0001-01-01` — NULL trá hình | kiểm cả chuỗi rỗng lẫn sentinel |
| 5 | `ex_date` **tương lai** — đã công bố, chưa xảy ra | lọc `ex_date ≤ as_of`; cấm tính `factor` mà không kèm `as_of` |
| 6 | **`id` đổi giữa hai lần gọi** — cùng sự kiện ACB 2026-06-15 trả hai `id` khác nhau | khoá khử trùng phải **bền theo nội dung**, không dùng `id` |

**Trường VCI** (dùng sai thì tiêu đề rỗng và cổ tức luôn NULL):
`eventTitleVi` · `exrightDate` · `exerciseRatio` · `valuePerShare`.

**Quy tắc số** (E0035): nguồn này **không bao giờ** dùng dấu phân cách nghìn. Giá tiền luôn là
số nguyên trần (`10000`); dấu `.` xuất hiện với 1–13 chữ số sau nó (`66.6666666666667`).
⇒ `.` và `,` **luôn** là dấu thập phân: `float(s.replace(",", "."))`.

**Phân loại từ `title`:** `ESOP` và `phát hành riêng lẻ` cũng khớp *"Phát hành thêm"* nhưng
**`factor = 1`**. Xếp chúng thành `RIGHTS` là sai (E0027).

**Ca nguồn ghi sai, phải tra tay:**

| Ca | Nguồn ghi | Đúng | Xác minh bằng |
|---|---|---|---|
| `VCG 2010-08-17` | `100:6116` | 0,6116 | VnEconomy `100:61,1603` + VCI |
| `BID 2013-06-21` | `100:1764` | 0,1764 | VCI `0,1763655` |
| `SHB 2012-08-17` | `1:021` | 0,21 | VCI ghi `0.0` — VCI hỏng ở ca này |
| `MWG 2014-04-15` | `1000:3720` | ❓ | VCI không có ⇒ tra CBTT |
| `PVT 2009-12-14` | `1:055` | ❓ | nt |

---

<a id="3"></a>
## 3. Bảng và cột

### 3.1 `obs.price_daily` — sổ quan sát, chỉ ghi thêm

```
PRIMARY KEY (source, security_id, trade_date, source_ts, observed_at)
UNIQUE      (source, security_id, trade_date, source_ts, payload_hash)
REVOKE UPDATE, DELETE
```

| Cột | Kiểu | Ý nghĩa |
|---|---|---|
| `source` | text | `SSI_SP` · `SSI_IBOARD` · `DNSE` · `VND` · `LEGACY_BRONZE` |
| **`security_id`** | bigint | 🔑 khoá bền, **không phải ticker** |
| `symbol` | text | nhãn tại thời điểm quan sát |
| `trade_date` | date | **valid time** — ngày giao dịch theo lịch ICT |
| `observed_at` | timestamptz | **transaction time** — lúc ta nhìn thấy |
| `source_ts` | timestamptz | dấu thời gian thô của nguồn, **nằm trong PK**. Cần vì DNSE có ngày trả 2 bar cùng `trade_date` |
| `open` `high` `low` `close` | numeric | giá trị nguyên bản của nguồn, **chưa quy đổi đơn vị** |
| `volume` | bigint | KL khớp lệnh |
| `total_value` | numeric | GT khớp lệnh (VND) — cột chính vì ADTV là bộ lọc tần suất cao nhất |
| `ref_price` | numeric | `RefPrice` của nguồn |
| `price_unit` | text | `VND` \| `kVND` — `CHECK` |
| `price_basis` | text | `RAW` \| `ADJUSTED` \| `UNKNOWN` — `CHECK` |
| `extra` | jsonb | trần/sàn, thoả thuận, khối ngoại, giá BQ… |
| `run_id` | uuid | → `ops.run_ledger` |
| `payload_hash` | bytea | `GENERATED` sha256 — **không** gồm `observed_at` |

> **Vì sao `security_id` chứ không phải ticker:** mã huỷ niêm yết rồi ký tự được cấp lại cho
> công ty khác ⇒ hai công ty trộn vào một chuỗi giá. `meta.securities` cấp `security_id`
> theo cặp `(ticker, listing_date)`; `symbol` chỉ là nhãn.
>
> **Lọc loại tài sản:** `meta.universe` chỉ nhận `security_id` có `sec_type = 'S'`. ETF, CW,
> trái phiếu có cơ chế điều chỉnh khác hẳn — lọt vào đây là hỏng lặng lẽ. Ràng buộc đặt ở
> `meta.universe`, không phải lọc lúc nạp.
>
> ⚠️ `RefPrice` của SSI vào ex_date là **close phiên trước, chưa hiệu chỉnh** (E0025).
> Giá tham chiếu thật của sàn nằm ở `Ceiling`/`Floor`.

### 3.2 `obs.corp_action` → `market.ca`

| Cột | Ý nghĩa |
|---|---|
| `security_id` `ex_date` | khoá gộp — **một dòng cho MỌI sự kiện cùng ngày** |
| `components` | jsonb — mỗi phần tử `{type, ratio, cash_vnd, issue_price}` |
| `r_bonus` `r_stock_div` `r_rights` `cash_vnd` `issue_price` | tổng hợp từ `components`, đầu vào công thức §4.2 |
| `factor` | hệ số cuối |
| `factor_method` | `DECLARED` \| `EXCHANGE_IMPLIED` \| `NO_ADJUST` \| `UNKNOWN` |
| `factor_exchange` | hệ số suy từ `Ceiling`/`Floor` của sàn — đầu vào C-15 |
| `evidence_id` | mã `E####` khi không phải `DECLARED` |

**Loại sự kiện:**

| Loại | Vào công thức qua | Ghi chú |
|---|---|---|
| `CASH_DIV` | `cash_vnd` | **Gross, trước thuế** — sàn điều chỉnh theo Gross |
| `STOCK_DIV` | `r_stock_div` | cổ tức bằng cổ phiếu |
| `BONUS_SHARES` | `r_stock_div` | cổ phiếu thưởng — bản chất pháp lý khác nhưng cùng công thức |
| `SPLIT` | `r_stock_div = n − 1` | tách 1→n |
| `RIGHTS` | `r_rights` + `issue_price` | |
| `ESOP` | ❌ không vào công thức | `factor = 1`, `factor_method='NO_ADJUST'` |
| `PRIVATE_PLACEMENT` | ❌ | nt |

⚠️ `obs.corp_action` phải có **chỉ mục khử trùng không chứa `observed_at`** — nếu không
`ON CONFLICT` không bao giờ bắt và mỗi lần chạy sinh một bộ dòng mới.

---

<a id="4"></a>
## 4. Công thức

### 4.1 Chiều

```
factor = P_ref / P_prev     (≤ 1)

adj(d, as_of) = raw(d) × Π factor(e)      với  d < e.ex_date ≤ as_of
raw(d)        = adj(d, T) ÷ Π factor(e)   với  d < e.ex_date ≤ T
```

Kiểm nhanh: PVD `raw = 31.050`, `adj = 18.600` ⇒ raw **lớn hơn** adj ⇒ **chia**.

### 4.2 Giá tham chiếu tại `ex_date` — HỢP NHẤT, không phải TÍCH

```
              P_prev − D + r_rights × P_ph
P_ref  =  ───────────────────────────────────────
           1 + r_bonus + r_stock_div + r_rights

factor =  P_ref / P_prev
```

| Ký hiệu | Nghĩa |
|---|---|
| `P_prev` | giá **raw** đóng cửa phiên liền trước `ex_date` |
| `D` | **tổng** cổ tức tiền mặt Gross (VND/cp), 0 nếu không có |
| `r_bonus` `r_stock_div` | tỷ lệ cổ phiếu thưởng / cổ tức CP — **gộp chung mẫu số** |
| `r_rights` `P_ph` | tỷ lệ quyền mua và giá phát hành |
| ESOP · riêng lẻ | **không xuất hiện trong công thức** |

**Một công thức, không phải bốn.** Bốn "loại sự kiện" chỉ là các trường hợp một số hạng bằng 0:

| Chỉ có | Rút gọn thành |
|---|---|
| `CASH_DIV` | `(P − D) / P` |
| `STOCK_DIV` / `BONUS` | `1 / (1 + r)` |
| `SPLIT` 1→n | `1 / n` |
| `RIGHTS` | `(P + r·P_ph) / ((1 + r)·P)` — TERP |

> 🔴 **Nhiều sự kiện cùng `ex_date` phải HỢP NHẤT trên mẫu số chung.** Nhân hai hệ số độc
> lập cho kết quả sai lớn: VND `2022-03-10` (BONUS 80% + RIGHTS 100%) — tích cho `23.197`,
> hợp nhất cho `29.821`, sàn công bố `29.803` (E0028).

**Đối chiếu với giá sàn công bố:**

| Mã | ex_date | Sự kiện | Tính | Sàn công bố |
|---|---|---|---|---|
| PVD | 2026-07-14 | STOCK_DIV 0,669 | `31.050/1,669` = 18.604 | 18.600 |
| TCB | 2024-06-20 | BONUS 1,0 | `48.300/2` = 24.150 | 24.153 |
| VCI | 2021-06-18 | BONUS 1,0 | `98.300/2` = 49.150 | 49.129 |
| VPB | 2022-09-28 | BONUS 0,5 | `27.400/1,5` = 18.267 | 18.252 |
| VND | 2022-03-10 | BONUS 0,8 + RIGHTS 1,0 | `(73.500+10.000)/2,8` = 29.821 | 29.803 |

### 4.3 Hệ số của sàn — đáp án, không phải suy diễn

```
ref_sàn         = (Ceiling + Floor) / 2
factor_exchange = ref_sàn / close(phiên trước)
```

> 🔑 Không cần biết biên độ. `C = R(1+b)` và `F = R(1−b)` ⇒ `C + F = 2R`.
> **Đừng gán cứng 7%/10%/15%** — biên độ đổi theo thời kỳ (HOSE 5% tới 2012 rồi 7% từ 2013,
> riêng 2008 co còn ~3,8%; HNX 7% tới 2012 rồi 10% từ 2013). Gán cứng làm 38–82% ex-date
> trước 2013 báo lệch oan.
>
> **Đừng dùng cột `RefPrice` của nguồn** để thay: lúc nó là ref đã điều chỉnh, lúc là giá
> đóng cửa phiên trước — không nhất quán (E0051).

`factor` **chính** dùng bản lý thuyết ở §4.2, **không làm tròn bước giá**, để tránh drift tích
luỹ qua nhiều sự kiện. `factor_exchange` chỉ dùng làm phép kiểm C-15.

**RIGHTS thiếu `issue_price`:** giải ngược từ `factor_exchange` ⇒ `factor_method =
'EXCHANGE_IMPLIED'`. Chỉ làm được từ 2020-02-26 (độ sâu `Ceiling`/`Floor`).
⚠️ Cấm giả định `issue_price = 10.000` — đã đo 4/8 ca RIGHTS 2026 khác hẳn mệnh giá.

---

<a id="5"></a>
## 5. Tầng đọc — hỏi giá thế nào

Mọi thứ ngoài kho đọc qua `market.*`. **Không truy vấn thẳng `obs.*`** — `obs` là nhật ký
quan sát, chưa khoanh nguồn, chưa quy đổi đơn vị, còn lẫn số suy ra.

| Gọi cái này | Trả về | Đơn vị |
|---|---|---|
| `market.px_raw` | giá **thô**, nguồn `SSI_SP` | VND |
| `market.px_adj` | giá **điều chỉnh, mặt bằng hôm nay** | VND |
| `market.px_adj_as_of(ngày)` | giá điều chỉnh, **mặt bằng ngày chỉ định** | VND |
| `market.ca` | sự kiện điều chỉnh giá | — |
| `market.factor_exchange` | hệ số sàn công bố ở mỗi phiên | — |

> **Backtest phải dùng `px_adj_as_of(ngày_đó)`**, không dùng `px_adj`. `px_adj` là mặt bằng
> hôm nay — dùng nó cho quá khứ là dùng thông tin từ tương lai.
>
> Mốc neo: `px_adj_as_of(d)` tại chính ngày `d` phải bằng `px_raw(d)`. Đó là phép kiểm C-11.

**`px_raw` trả `NULL` cho O/H/L ngoài HOSE.** `close` luôn là số **đo**; nhưng O/H/L ở
`SSI_SP` được suy ra bằng `raw_x = adj_x / (adj_close / raw_close)`, và phép suy đó chỉ đúng
ở HOSE (145/147 mã), hỏng ở HNX (14/26) và UPCoM (0/29 — sàn dùng giá bình quân) (E0061, E0063).

**Có bất kỳ sự kiện nào `factor_method='UNKNOWN'`** trong khoảng ⇒ giá trả `NULL` kèm cờ
`ca_unknown`, không trả số gần đúng.

**Không có dạng scalar.** Không tồn tại hàm trả một con số cho `(mã, ngày)` — engine backtest
gọi theo từng dòng trên hàng triệu điểm sẽ chết. Chỉ có dạng bảng, tính một lượt bằng window
function, nạp thẳng vào Polars.

---

<a id="6"></a>
## 6. Luồng chạy hằng ngày

**16:15 ICT, T2–T6.** Universe theo `meta.universe WHERE active` (`sec_type='S'`).

| Bước | Việc | Chi tiết |
|---|---|---|
| 0 | Mở `run_id` | `n_expected := count(meta.universe WHERE active)` |
| 1 | Đọc `meta.source_contract` | `price_unit`/`price_basis` theo nguồn — **không hardcode** |
| 2 | SSI `DailyStockPrice` · cửa sổ **30 ngày** | 30 ngày = trần một lời gọi ⇒ hỏi 30 ngày tốn đúng bằng hỏi 1 ngày |
| 3 | DNSE `1D` · cửa sổ 30 ngày | đối chứng độc lập |
| 4 | Sự kiện: DNSE senses + VCI `/v1/events` · cửa sổ 90 ngày | phân trang **tới hết** |
| 5 | Gate lúc bắt (C-01→C-04) | cột hỏng → **NULL riêng cột**, dòng hỏng → quarantine, **sang dòng kế, không bỏ cả mã** |
| 6 | `INSERT ... ON CONFLICT DO NOTHING` | nguồn nói y hệt ⇒ 0 dòng mới |
| 7 | Tính lại `market.ca` cho `ex_date` mới | gộp theo `(security_id, ex_date)` **trước** khi tính |
| 8 | Đóng `run_ledger` · chạy C-05→C-17 | ~30 giây |
| 9 | Ghi `ops.STATUS` + dòng vào `logs\INDEX.csv` | đỏ thì lên đầu file |

**Cửa sổ & watermark:** chuẩn 30 ngày · dữ liệu trễ ≤ 30 ngày hấp thụ tự động · trễ hơn phải
backfill tường minh. `ON CONFLICT DO NOTHING` khiến hỏi lại ngày cũ vô hại; nguồn **sửa** ⇒
sinh quan sát mới, đúng thứ cần bắt.

**Nguồn chia lại lịch sử:** job so **chính dữ liệu** (không dựa bảng sự kiện) để phát hiện
nguồn đã điều chỉnh lại chuỗi, rồi nạp lại mã đó. Phân biệt được 4 ca: y hệt → bỏ qua ·
lệch 0,1% do làm tròn → bỏ qua · cổ tức 3% → nạp lại · thưởng 1:1 → nạp lại (E0062).

### 6.1 Chính sách gọi nguồn (E0029)

| Nguồn | Độ trễ p50 | Giãn cách áp dụng | 204 mã |
|---|---|---|---|
| SSI | — | **1,0 s** | ~4,5 phút |
| DNSE | 0,55 s | **0,2 s** | ~2,6 phút |
| VCI IQ | 0,82 s | **0,3 s** | ~3,8 phút |

Đo 30 lời gọi/nguồn ở 3 mức giãn cách: tất cả 200, không 429. Vẫn giữ giãn cách vì mẫu nhỏ
không chứng minh được là không có quota theo giờ/ngày, và chi phí bảo hiểm là ~40 giây/ngày.
SSI giữ 1,0 s vì có token — bị chặn thì tốn hơn nhiều so với hai nguồn công khai.

**Bắt buộc:** backoff luỹ thừa khi `429`/`5xx`, và **ghi mọi lần 429 vào `ops.run_ledger`**
để giới hạn thật tự lộ ra theo thời gian thay vì phải đoán.

---

<a id="7"></a>
## 7. Luồng nạp lần đầu

| # | Việc | Nguồn | Thời gian | Duyệt |
|---|---|---|---|---|
| 1 | Sự kiện DN 2007 → nay | DNSE senses + VCI IQ | ~15 phút | — |
| 2 | Adjusted toàn lịch sử | SSI iBoard (chính) + DNSE (đối chứng) | ~25 phút | — |
| 3 | Legacy `basis='UNKNOWN'` | ver 1 qua `postgres_fdw` | ~30 phút | — |
| 4 | Raw 2020-02-26 → nay | SSI `DailyStockPrice` | **~5,6 giờ** | ⚠️ **CẦN** |

Sự kiện phải nạp **trước** giá. Trước bước 4 bắt buộc diễn tập một mã đủ đường đi và đối
chiếu bộ dữ liệu vàng (~2 phút).

---

<a id="8"></a>
## 8. 18 phép kiểm

Mỗi phép kiểm khai **nó bắt lỗi nào đã từng xảy ra thật**.
Trạng thái cài đặt hiện tại: chạy `3. Build\test_spec_vs_code.py`.
Giải nghĩa từng mã trong một dòng: `BANG_TRA_MA_KIEM.md`.

### Tầng A — lúc bắt

| Mã | Kiểm | Bắt lỗi thật nào |
|---|---|---|
| **C-01** | `low ≤ close ≤ high` từng dòng. `open` ngoài biên ⇒ **NULL riêng `open`, GIỮ dòng**; chỉ bỏ cả dòng khi `close` ngoài biên hoặc `high < low` | nguồn trả `high=low=close` còn `open` đúng — 1.085 dòng. Và iBoard: 10.795/12.215 dòng bị chặn chỉ sai mỗi `open`, vứt cả dòng làm 8 mã rơi sang DNSE (2026-08-12) |
| **C-02** | Neo thang giá vs `close` phiên trước; nới biên nếu là `ex_date` | CTG 2013 `close=6` · VIX 2014 `close>200.000` |
| **C-03** | `EMPTY = FAIL` khi kỳ vọng có data | scraper nuốt lỗi, 0 dòng nhiều tháng |
| **C-04** | Phân trang: `totalElements` vs số dòng nhận | lời gọi mặc định lấy 6/30 sự kiện |

**Ca hợp lệ không được bắt nhầm:** `o=h=l=c` khi khoá trần/sàn · `vol=0` ngày không giao dịch.

### Tầng B — lúc ghi, Postgres cưỡng chế

| Mã | Kiểm | Bắt lỗi thật nào |
|---|---|---|
| **C-05** | `REVOKE UPDATE, DELETE` + `ALTER DEFAULT PRIVILEGES` | job đêm xoá backfill 5 đêm liền |
| **C-06** | `CHECK price_unit` · `CHECK price_basis` | lẫn thang ×1000 · trộn raw/adj |
| **C-07** | `UNIQUE (source, security_id, trade_date, source_ts, payload_hash)` | ghi trùng khi chạy lại |

### Tầng C — sau khi chạy, hằng ngày

| Mã | Kiểm | Ngưỡng | Bắt lỗi thật nào |
|---|---|---|---|
| **C-08** | Đủ mã: `n_expected == n_seen` | lệch bất kỳ | rổ tụt 100→30 mà chỉ `log.warning` |
| **C-09** | Đủ ngày: `universe × trading_calendar` LEFT JOIN `obs` | có NULL | máy tắt một hôm, thủng âm thầm |
| **C-10** | `observed_at ≥ trade_date` | vi phạm bất kỳ | lệch múi giờ ICT/UTC |
| **C-11** | `px_adj_as_of(d)` tại `d` == `px_raw(d)` | > 0,01% | sai chiều công thức lộ ngay |
| **C-12** | Nhảy giá > 50%/phiên không có sự kiện | bất kỳ | **trộn nguồn trong một chuỗi** — 219 bậc giả (E0045) |

### Tầng D — đối chiếu độc lập

| Mã | Kiểm | Tần suất | Bắt lỗi thật nào |
|---|---|---|---|
| **C-13** | `SSI_raw × Π factor` vs giá adj ngoài ex-date | 1/7 universe/ngày | trộn raw/adj · thiếu sự kiện |
| **C-14** | Hai quan sát khác `observed_at` phải dựng ra **cùng** `raw` | tuần | thiếu sự kiện trong khoảng giữa |
| **C-15** | `factor` tính ra vs `factor_exchange` từ `Ceiling`/`Floor` | mọi `ex_date` | sai loại sự kiện · sai `ratio` · thiếu sự kiện cùng ngày · điều chỉnh nhầm ESOP |
| **C-16** | > 1 bar cho cùng `(source, symbol, trade_date)` | mỗi lần nạp | feed DNSE 2022-12-27: 93/100 mã có bar thừa ở `00:00 UTC` thay vì `02:00 UTC` |
| **C-17** | `ratio` DNSE vs `exerciseRatio` VCI — so **TẬP với TẬP**, **cấm `.first()`** | mỗi lần nạp sự kiện | 12/575 ca lệch thật. Lấy `.first()` cho 116 báo động giả |
| **C-18** | Dải hợp lý mọi số parse ra: `cash_vnd ∈ [10; 50.000]` · `ratio ∈ [0; 3]` · `issue_price ∈ [1.000; 200.000]` | mỗi lần nạp sự kiện | số sai **chạy êm và trông hợp lý** — không phép kiểm cấu trúc nào thấy |

**Ngưỡng C-15:** `|factor − factor_exchange| × P_prev ≤ 1 bước giá`. Lệch ở ca đúng: 3–21đ,
đều dưới nửa bước giá; ca sai quy tắc lệch 6.600đ ⇒ phân biệt được rõ ràng.

> C-15 là phép kiểm **mạnh nhất**: sàn công bố đáp án ở mỗi ex-date, và nó bắt cả bốn lớp lỗi
> hệ số cùng lúc. Hạn chế: chỉ có từ 2020-02-26.
>
> C-12 là phép kiểm **rẻ nhất mà bắt được lỗi kiến trúc** — nó chính là thứ phát hiện trộn nguồn.

### Bộ dữ liệu vàng — chạy trước mọi commit chạm luồng giá

Không phụ thuộc DB, không phụ thuộc mạng. **Thứ duy nhất biết trước đáp án** — mọi phép kiểm
khác chỉ kiểm tính nhất quán nội tại, mà hệ thống có thể nhất quán sai.

| # | Ca | Kỳ vọng | Bảo vệ điều gì |
|---|---|---|---|
| G1 | PVD `2026-07-14` STOCK_DIV 0,669 | `factor = 0,5992` · `ref = 18.604` | công thức cơ bản |
| G2 | TCB `2024-06-20` BONUS 1,0 | `ref = 24.150` | BONUS = STOCK_DIV |
| G3 | VPB `2022-09-28` BONUS 0,5 | `ref = 18.267` | nt |
| G4 | **VND `2022-03-10` BONUS 0,8 + RIGHTS 1,0** | **`ref = 29.821`, không phải 23.197** | quy tắc HỢP NHẤT |
| G5 | **PNJ `2024-09-26` ESOP 1,0%** | **`factor = 1`** | ESOP không điều chỉnh |
| G6 | MBB CASH_DIV 1.000đ | `factor = 0,9615` | cash div Gross |
| G7 | MBS `2026-04-02` RIGHTS r=0,5, `P_prev = 24.500`, ref sàn 19.700 | `issue_price` ngụ ý 10.100 | TERP · cấm giả định mệnh giá |
| G8 | PVS `2025-05-19` `o=26260 h=l=c=26080` | chỉ `open` lệch ⇒ C-01 phải **CỨU dòng, NULL open** | gate OHLC |
| G9 | ACB khoá trần `o=h=l=c` `vol=500k` | C-01 phải **THA** | không bắt nhầm |
| G9a | `close` ngoài `[low, high]` | C-01 phải **BỎ CẢ DÒNG** | close là cột chịu lực |
| G9b | `high < low` | C-01 phải **BỎ CẢ DÒNG** | hỏng thật |
| G10 | PVD 01–24/07/2026, 18 phiên | `DNSE_v / SSI_vol = 1.0000` | volume không un-adjust |
| G11 | PVD `raw=31.050` `adj=18.600` | `raw = adj ÷ factor` | chiều công thức |
| G12 | CTG 2013 `close=6` | phải bị loại | known-bad |
| G13 | `MBB 2025-08-13` tỷ lệ `100:32` | DNSE `0,32` đúng · VCI `0,03` sai | C-17 |
| G14 | `BID 2013-06-21` tỷ lệ `100:1764` | VCI `0,1764` đúng · DNSE `17,64` sai | C-17 chiều ngược lại |
| G15 | `BVH 2022-11-25` `3026.1 đồng/CP` | `3.026,1đ`, không phải `30.261đ` | `.` là thập phân |
| G16 | `HSG 2012-06-27` `bằng tiền mặt 500đồng/CP` | parse ra `500` | biến thể chữ |
| G17 | DNSE `ACB 2022-12-27` | 2 bar `00:00 UTC` v=418k và `02:00 UTC` v=2.040k | C-16 — phải giữ bar `02:00` |
| G18 | DNSE `MBB` bản ghi `MBB: MBB: Thông báo…` | phân loại `BAN_TIN` | bẫy #3 §2.2 |
| G19 | `CTG 2023-11-30` tỷ lệ `100:11.7415` | `r = 0,117415` | `.` là thập phân kể cả 4+ chữ số |
| G20 | `GMD 2006-03-02` tỷ lệ `100:66.6666666666667` | `r = 0,6667` | 13 chữ số thập phân |
| G21 | `giá 10000 đồng/CP` | `10.000` | số nguyên trần, không phân cách nghìn |
| G22 | `VCG 2010-08-17` tỷ lệ `100:6116` | C-18 phải **BẮT** (`r=61,16 > 3`) | lỗi nguồn ⇒ `UNKNOWN`, không đoán |
| G23 | SHP 2012-04 | không có bậc `×0,08` rồi `×12,17` | một chuỗi = một nguồn |

---

<a id="9"></a>
## 9. Xử lý lỗi

### 9.1 Ma trận triệu chứng → hành động

| Triệu chứng | Chẩn đoán | Hành động | Chặn |
|---|---|---|---|
| **C-01** đỏ, ít dòng | nguồn trả dòng hỏng lẻ | quarantine dòng, chạy tiếp | không |
| **C-01o** tăng vọt | nguồn đổi cách trả `open` | so `ops.quarantine` mã `C-01o`; giá trị vẫn dùng được thì xét nới cổng | không |
| **C-01** đỏ, > 5% dòng một nguồn | nguồn hỏng có hệ thống | dừng nguồn đó, chạy tiếp nguồn khác | nguồn đó |
| **C-02** đỏ | sai thang giá hoặc thiếu sự kiện | so hệ số ngụ ý với `market.ca`; khớp ⇒ tha | dòng |
| **C-03** đỏ | nguồn đổi API / hết token | dò lại endpoint; cập nhật `source_contract` + `INGEST_SPEC` **cùng commit** | dataset |
| **C-04** đỏ | lời gọi thiếu tham số phạm vi | sửa, **nạp lại toàn bộ dataset đó** | dataset |
| **C-05/06/07** đỏ | schema bị sửa sai | 🔴 **DỪNG TOÀN BỘ** | tất cả |
| **C-08** đỏ | co rổ trong im lặng | không fallback, **FAIL to** | lần chạy |
| **C-09** đỏ | thủng ngày | tự backfill nếu ≤ 30 ngày; ngoài ⇒ backfill tường minh | không |
| **C-10** đỏ | lệch múi giờ | sửa chuẩn hoá tz, nạp lại phần ảnh hưởng | dataset |
| **C-11** đỏ | sai chiều hoặc sai công thức hệ số | 🔴 **DỪNG mọi thứ đọc `px_adj`** | downstream |
| **C-12** đỏ, trùng `ex_date` | bình thường | không làm gì | không |
| **C-12** đỏ, không có sự kiện, **nhảy qua lại** | **trộn nguồn trong một chuỗi** | kiểm `nguon_cua_ma` — mỗi mã phải khoá đúng một nguồn | vùng đó |
| **C-12** đỏ, không có sự kiện, nhảy một chiều | thiếu sự kiện | tra VCI/DNSE events; không có ⇒ `UNKNOWN` ⇒ giá NULL + cờ | vùng ngày |
| **C-13** đỏ | trộn raw/adj hoặc thiếu sự kiện | khoanh mã/khoảng ngày, xử như C-12 | vùng đó |
| **C-14** đỏ | thiếu sự kiện giữa hai quan sát | tra events trong `[T1, T2]` | vùng đó |
| **C-15** đỏ, `factor_exchange ≈ 1` | ta điều chỉnh mà **sàn không** | kiểm có phải ESOP/riêng lẻ ⇒ `NO_ADJUST` | sự kiện đó |
| **C-15** đỏ, lệch lớn có quy luật | thiếu sự kiện cùng `ex_date` | tra lại events cùng ngày, gộp lại | sự kiện đó |
| **C-15** đỏ, RIGHTS thiếu `issue_price` | không tính được TERP | giải ngược ⇒ `EXCHANGE_IMPLIED` + `evidence_id` | không |
| **C-16** đỏ | nguồn trả > 1 bar/ngày | giữ **cả hai** trong `obs` (khác `source_ts`); tầng đọc chọn bar ở timestamp-of-day phổ biến nhất của nguồn đó — **tự xác định, không hardcode** | không |
| **C-17** đỏ, `ex_date ≥ 2020-02-26` | một trong hai nguồn sai | **hỏi sàn**: nguồn nào khớp `factor_exchange` trong 1 bước giá thì lấy | không |
| **C-17** đỏ, `ex_date < 2020-02-26` | không có trọng tài | `UNKNOWN` — cấm chọn bừa một nguồn | vùng ngày đó |
| **C-18** đỏ | parse sai hoặc nguồn ghi thiếu dấu thập phân | `UNKNOWN`, ghi title gốc vào evidence, **không đoán** | sự kiện đó |

### 9.2 Ba nguyên tắc

> **N1 — Thà thiếu còn hơn sai mà trông như đúng.** Không chắc ⇒ `NULL` + cờ.
>
> **N2 — Không bao giờ sửa dòng cũ.** Sửa = ghi một quan sát mới.
>
> **N3 — Lỗi một dòng không giết cả mã; lỗi một mã không giết cả lần chạy.** Nhưng lỗi nền
> tảng (C-05→C-07, C-11) thì dừng tất cả.

### 9.3 Sự kiện bị huỷ hoặc thay đổi sau khi công bố

| Tình huống | Xử lý |
|---|---|
| Phương án đổi **trước** `ex_date` | ghi `components` mới với `observed_at` mới; `factor` tính lại; dòng cũ giữ nguyên |
| Đợt phát hành **huỷ sau** `ex_date` | 🔴 chưa xác minh HOSE xử thế nào. Giả định làm việc: điều chỉnh **vẫn có hiệu lực** vì thị trường đã giao dịch ở mặt bằng mới; sàn ra thông báo điều chỉnh lại ⇒ **sự kiện mới với `ex_date` mới** |
| Tỷ lệ nộp tiền thấp / phát hành không đủ | không ảnh hưởng `factor` — sàn điều chỉnh theo **phương án công bố**, không theo kết quả thực hiện |

Ô 🔴 cần một ca thật để kiểm. C-15 sẽ tự phát hiện nếu giả định sai.

### 9.4 Leo thang

| Mức | Điều kiện | Hành động |
|---|---|---|
| 1 · ghi nhận | 1 lần đỏ, đã tự xử | `run_ledger` |
| 2 · cảnh báo | đỏ ≥ 3 ngày liên tiếp | lên đầu `ops.STATUS` |
| 3 · chặn | C-05→C-07 hoặc C-11 đỏ | dừng chain |
| 4 · dead-man | không có run > 36h | cờ cao nhất |

---

<a id="10"></a>
## 10. Việc người phải làm

| Khi nào | Việc |
|---|---|
| `factor_method='UNKNOWN'` xuất hiện | tra CBTT/HOSE, điền `issue_price` hoặc `ratio` |
| C-13/C-15 đỏ mà không phải thiếu sự kiện | quyết nguồn nào đúng |
| Nguồn đổi API | cập nhật `source_contract` + `INGEST_SPEC` + `evidence_id` **cùng commit** |
| Trước mỗi backfill nặng | duyệt go/no-go |
| Hằng tuần | đọc `ops.STATUS`, xử mục mức 2 |
| Hằng quý | đo lại mốc sâu nhất của mỗi nguồn (cuộn hay cố định) |

### 10.1 Quy ước nền

| | |
|---|---|
| Múi giờ | `timestamptz` lưu **UTC** cho mọi mốc thời gian thật. `trade_date` là `DATE` theo **lịch ICT**, không quy đổi. Nguồn trả ICT tz-naive ⇒ quy đổi ở **một chỗ duy nhất**, trong adapter của nguồn |
| Ngày không giao dịch (`vol=0`) | **lưu hết**. `obs` là nhật ký quan sát — *"nguồn nói ngày này volume=0"* là một sự thật. Lọc là phán xét, để ở tầng đọc. Giữ thì còn phân biệt được "không có phiên" với "có phiên nhưng không khớp lệnh" |
| `px_adj` có matview không | **không**. Thuần hàm set-returning; 550k dòng + window function ≈ 1–2 s. Thêm matview chỉ khi **đo được** là chậm |

### 10.2 Biên độ ±% của một ngày quá khứ

Chỉ cần cho C-02. `factor_exchange` (§4.3) **không cần** biên độ.

`Securities.Market` của nguồn là **sàn hiện tại**, không phải sàn của ngày quá khứ — dùng nó
đã gây 141 báo động giả. Cách đúng: suy từ chính dòng dữ liệu ngày đó, trên ngày **không phải
ex-date**:

```
band(d) = round_to_nearest( Ceiling(d) / RefPrice(d) − 1 , {0.038, 0.05, 0.07, 0.10, 0.15} )
```

Ngày ex-date thì **kế thừa band của phiên liền trước** — vào ex-date `RefPrice` là close phiên
trước chưa hiệu chỉnh nên tỷ số vô nghĩa.

---

## Tham chiếu

| Cần gì | Mở |
|---|---|
| Thiết kế tầng và bảng | `DB_DESIGN.md` |
| Endpoint và từng trường | `INGEST_SPEC.md` |
| Giải nghĩa mã C-01…C-18 | `BANG_TRA_MA_KIEM.md` |
| Vì sao quy tắc lại như vậy · lịch sử sai | `..\archive\OHLCV_SPEC.md` |
| Phiên bản nào đổi gì | `..\archive\CHANGELOG.md` |
| Một con số đã đo — tra lại thay vì đo lại | `C:\Users\OS\Lucius\Projects\PRJ quant\09_other\evidence\INDEX.md` |
