# Thiết kế bảng — DB PRJ quant

> Tài liệu **3/3** của bộ thiết kế dữ liệu.
> (1) cần lấy gì (`DATA_REQUIREMENTS.md`) · (2) nguồn nào cho được (`DATA_SOURCE_MAP.md`) ·
> (3) thiết kế bảng ← *file này*
>
> Hai phần: **§A bảng tra cứu** (bảng nào giữ gì) · **§B sơ đồ tầng** (ai ghi, ai đọc).
> Migration SQL thật: `3. Build\schema\`. Trạng thái thi công: chạy `3. Build\status.py`.

---

## 0. Một nguyên tắc, mọi thứ khác suy ra từ đó

> **Lưu QUAN SÁT, không lưu GIÁ.**
>
> *"Lúc 19:42 ngày 29/07, SSI nói close của ACB ngày 24/07 là 22.500, trường `ClosePrice`,
> đơn vị VND, cơ sở RAW."* — câu đó không bao giờ trở thành sai.
> *"Close của ACB ngày 24/07 là 22.500"* — câu này trở thành sai ngay lần chia tách kế tiếp.

Bệnh của DB hiện tại không phải một job hỏng, mà là: **bảng khả biến, một-giá-trị-một-ô,
nhiều writer khác ngữ nghĩa cùng ghi đè**. Trong cấu trúc đó "job đêm thắng" không phải sự
cố — nó là hành vi được đảm bảo. Mọi bản vá thọ tối đa một đêm.

Ba luật cưỡng chế bằng **máy**, không bằng văn bản:

| Luật | Cưỡng chế bằng |
|---|---|
| **L1.** Tầng `obs` chỉ INSERT | `REVOKE UPDATE, DELETE ON SCHEMA obs` — Postgres từ chối, không phải reviewer từ chối |
| **L2.** Giá trị lưu **nguyên bản của nguồn** + cột khai `đơn vị` và `cơ sở giá` | `CHECK (price_unit IN ('VND','kVND'))` · `CHECK (price_basis IN ('RAW','ADJUSTED','UNKNOWN'))` |
| **L3.** Không cột nào là **vết sẹo** | Không có `backfilled`. Xuất xứ = `observed_at` + `run_id`; nhãn `raw_origin` **tính lại mỗi lần đọc** |

---

# §A — BẢNG TRA CỨU

## A1. Toàn bộ bảng

`W` = ai ghi · `R` = ai đọc

| Tầng | Bảng | Hạt | Khoá chính | Giữ mục nào (theo tài liệu 1) |
|---|---|---|---|---|
| **L0 obs** | `obs.price_daily` | nguồn × mã × ngày × lần quan sát | `(source, symbol, trade_date, observed_at)` | 1.1 1.2 1.6 1.7 1.8 1.9 |
| | `obs.price_intraday` | nguồn × mã × phút × lần qs | `(source, symbol, ts, observed_at)` | 1.3 |
| | `obs.foreign_flow` | nguồn × mã × ngày × lần qs | `(source, symbol, trade_date, observed_at)` | 2.1 2.2 |
| | `obs.corp_action` | nguồn × mã × ex_date × loại × lần qs | `(source, symbol, ex_date, event_code, observed_at)` | 3.1–3.10 |
| | `obs.index_daily` | nguồn × chỉ số × ngày × lần qs | `(source, index_code, trade_date, observed_at)` | 4.1 4.2 4.3 4.5 4.7 |
| | `obs.security_ref` | nguồn × mã × lần qs | `(source, symbol, observed_at)` | 5.4 7.1 7.3 7.4 7.5 |
| | `obs.financial` | nguồn × mã × kỳ × chỉ tiêu × lần qs | `(source, symbol, period, metric, observed_at)` | 5.1 5.2 5.3 5.6 |
| | `obs.ownership` | nguồn × mã × chủ sở hữu × lần qs | `(source, symbol, owner_code, observed_at)` | 5.7 2.4 |
| | `obs.macro` | nguồn × chuỗi × ngày × lần qs | `(source, series_id, date, observed_at)` | 6.1–6.7 |
| | `obs.document` | nguồn × tài liệu | `(source, doc_id)` | 8.1 8.2 |
| **L1 market** | `market.px_raw` 👁 | mã × ngày | `(symbol, date)` | giá thô đã phân giải |
| | `market.px_adj(as_of)` ƒ | mã × ngày × mốc quy chiếu | — | giá điều chỉnh, PIT |
| | `market.ca` 👁 | mã × ex_date | `(symbol, ex_date)` | hệ số + cách có hệ số |
| | `market.index_daily` 👁 | chỉ số × ngày | `(index_code, date)` | 4.1 |
| | `market.breadth` 👁 | ngày | `(date)` | 4.2 |
| | `market.foreign_flow` 👁 | mã × ngày | `(symbol, date)` | 2.1 2.2 |
| | `market.px_intraday` 👁 | mã × phút | `(symbol, ts)` | 1.3 |
| **L2 meta** | `meta.securities` 👁 | mã | `(symbol)` | 7.1 7.2 7.3 7.4 7.5 |
| | `meta.universe` ✍ | mã | `(symbol)` | danh sách 204 mã |
| | `meta.ecosystem` ✍ | nhóm | `(eco_id)` | 7.8 |
| | `meta.ecosystem_member` ✍ | nhóm × mã | `(eco_id, symbol)` | 7.8 |
| | `meta.trading_calendar` 👁 | ngày | `(date)` | 7.6 |
| | `meta.data_catalog` ✍ | bảng | `(table_name)` | — |
| | `meta.source_contract` ✍ | nguồn × dataset | `(source, dataset)` | khai đơn vị/cơ sở giá |
| **L3 features** | `features.daily_signals` | mã × ngày × feature | `(symbol, date, feature)` | — |
| | `features.financials` 👁 | mã × kỳ × chỉ tiêu | — | 5.1–5.3 |
| | `features.analyst_reports` · `report_theses` | báo cáo | — | 8.1 |

👁 = VIEW / matview (không ai ghi trực tiếp) · ƒ = hàm có tham số · ✍ = **người** khai báo

## A2. Cột của ba bảng lõi

### `obs.price_daily`

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `source` | text | `SSI` · `DNSE` · `VCI_IQ` · `LEGACY_BRONZE` |
| `symbol` `trade_date` | text · date | |
| `observed_at` | timestamptz | **cột quan trọng nhất bảng** — quyết định ngữ nghĩa dòng |
| `open` `high` `low` `close` | numeric | **giá trị nguyên bản của nguồn**, chưa quy đổi |
| `volume` | bigint | |
| `price_unit` | text | `VND` \| `kVND` — DNSE trả nghìn đồng, **không quy đổi lúc ghi** |
| `price_basis` | text | `RAW` \| `ADJUSTED` \| `UNKNOWN` |
| `extra` | jsonb | 24 trường SSI còn lại: trần/sàn/tham chiếu, giá trị GD, thoả thuận, giá BQ… |
| `run_id` | uuid | trỏ về `ops.run_ledger` |
| `payload_hash` | bytea | dedupe: chỉ INSERT khi khác quan sát gần nhất cùng khoá |

> **Vì sao `extra jsonb`:** pipeline hiện đọc 7/31 trường SSI và **vứt 24 trường còn lại**,
> khiến 6 mục bị đánh 🔴 "chưa lấy" trong khi dữ liệu đã nằm trong phản hồi. `extra` giữ
> nguyên phản hồi ⇒ thêm feature sau này không phải fetch lại lịch sử.

### `market.ca` — hệ số điều chỉnh phải khai **cách có được nó**

| Cột | Ghi chú |
|---|---|
| `symbol` `ex_date` | |
| `event_type` | `CASH_DIV` `STOCK_DIV` `SPLIT` `RIGHTS` `OTHER` |
| `ratio` `cash_vnd` `issue_price` | `issue_price` **bắt buộc** cho RIGHTS — thiếu thì TERP không tính được |
| `factor` | hệ số cuối. ⚠️ Nhiều sự kiện cùng `ex_date` ⇒ **HỢP NHẤT trên mẫu số chung**, **KHÔNG phải TÍCH** — quy tắc TÍCH đã bị **E0028** bác. Công thức: `OHLCV.md` §4.2 |
| `factor_method` | `DECLARED` \| `IMPLIED_FROM_MARKET` \| **`UNKNOWN`** |
| `evidence_id` | mã `E####` khi `IMPLIED_FROM_MARKET` |

> **`UNKNOWN` phải LAN TRUYỀN.** `px_adj` bắc qua một sự kiện `UNKNOWN` trả **NULL + cờ**,
> tuyệt đối không trả một số sai lặng lẽ. Đây là luật chống-fail-thầm-lặng đặt ở tầng mô
> hình dữ liệu — chỗ mà quên là không thể.
>
> Đã đo: 4/8 sự kiện RIGHTS 2026 có giá phát hành **khác hẳn mệnh giá** ⇒ giả định
> "luôn 10.000" bị **cấm trong code**, không chỉ ghi trong tài liệu.

### `market.px_raw` — hàm phân giải, không phải bảng

```
cho mỗi (symbol, date):
  2020-02-26+       → quan sát SSI mới nhất, price_basis='RAW'   raw_origin='SSI_OBSERVED'
  trước 2020-02-26  → NULL                                       raw_origin='NONE'
  price_basis='UNKNOWN' (LEGACY_BRONZE) → KHÔNG BAO GIỜ được chọn

❌ Nhánh DỰNG LẠI 2012–2020 đã BỎ (2026-08-01) — lý do đầy đủ: `..\archive\OHLCV_SPEC.md` §3.4b.
   Trước 2020-02-26 dùng thẳng chuỗi ĐÃ ĐIỀU CHỈNH của DNSE (không cần bảng sự kiện).
```

`raw_origin` là **cột dẫn xuất**, tính lại mỗi lần đọc — nên không mục được như `backfilled`.

---

# §B — SƠ ĐỒ TẦNG

## B1. Luồng ghi và luồng đọc

```
┌─ NGUỒN ────────────────────────────────────────────────────────────────────┐
│  SSI (raw, 31 trường)   DNSE (adjusted, 14 năm)   VCI IQ (sự kiện, BCTC)   │
│  SBV · trolyluat · Yahoo · Vietstock                                       │
└────────────────────────────────┬───────────────────────────────────────────┘
                                 │  CHỈ INSERT ─ mỗi nguồn một đường riêng
                                 │  KHÔNG fallback xuyên ngữ nghĩa
                                 ▼
┌─ L0  obs.*  NHẬT KÝ QUAN SÁT ──────────────────────────────────────────────┐
│  bất biến · append-only · REVOKE UPDATE,DELETE · dedupe theo payload_hash  │
│  price_daily · price_intraday · foreign_flow · corp_action · index_daily   │
│  security_ref · financial · ownership · macro · document                   │
└────────────────────────────────┬───────────────────────────────────────────┘
                                 │  hàm phân giải XÁC ĐỊNH (code, review được)
                                 ▼
┌─ L1  market.*  CHUẨN HOÁ (VIEW) ───────────────────────────────────────────┐
│  px_raw ─────────────┐                                                     │
│  ca (factor+method) ─┼──► px_adj(as_of)   ◄── truyền as_of ⇒ backtest PIT   │
│  index_daily · breadth · foreign_flow · px_intraday                        │
└────────────────────────────────┬───────────────────────────────────────────┘
                                 ▼
┌─ L2  meta.*  KHAI BÁO (người viết) ────────────────────────────────────────┐
│  universe(204) · ecosystem · securities · trading_calendar 👁               │
│  data_catalog · source_contract                                            │
└────────────────────────────────┬───────────────────────────────────────────┘
                                 ▼
┌─ L3  features.*  →  model · backtest · phân tích ngày · web viz            │
└────────────────────────────────────────────────────────────────────────────┘

         ┌─ ops.*  VẬN HÀNH (cắt ngang mọi tầng) ─┐
         │  run_ledger · check_result · STATUS    │
         └────────────────────────────────────────┘
```

## B2. Ai được ghi vào đâu

| Tầng | Ai ghi | Cách | Ai đọc |
|---|---|---|---|
| `obs.*` | **chỉ job fetch** | `INSERT` — không có đường nào khác | L1 |
| `market.*` | **không ai** | VIEW; sửa = sửa hàm phân giải, có commit | L2 L3, người |
| `meta.*` | **người** | khai báo tường minh, có `reason` + ngày | mọi tầng |
| `features.*` | job tính feature | rebuild được từ L1 | model, app |
| `ops.*` | mọi job | append | người, check |

## B3. Vì sao ba lớp lỗi cũ biến mất

| Lỗi cũ | Cấu trúc mới xử thế nào |
|---|---|
| Job đêm xoá backfill SSI, 5 đêm liền | **Bất khả** — không role nào có quyền `UPDATE`/`DELETE` trên `obs`. Không phải "được khuyến cáo", mà là Postgres từ chối |
| "Adjusted một phần" không sửa được bằng bất kỳ hệ số nào | `observed_at` **chính là** thông tin đang thiếu. Trạng thái hiệu chỉnh = f(source, observed_at, `market.ca`) ⇒ từ nan y thành tính được |
| Raw không dựng lại được từ landing | Không còn landing→bronze. `obs` **chính là** landing. Bỏ `200_migrate_from_landing.sql` ⇒ bỏ luôn cơ chế xoá |
| Khe gãy 2021-01-01 rộng dần | Không tồn tại về cấu trúc: mọi giá lưu là raw-tại-ngày-đó, điều chỉnh là hàm đọc liên tục trên toàn lịch sử. Cú nhảy chỉ còn xuất hiện khi **thiếu sự kiện** — một khuyết tật gọi được tên |
| 1.272 dòng OHLC bất khả · 1.085 dòng high/low bị đè | Vào `obs` với `source='LEGACY_BRONZE'`, `price_basis='UNKNOWN'`; hàm phân giải **không bao giờ chọn**. Bảo tồn, truy nguyên được, và **trơ** — không xoá, không sửa |
| Lẫn thang giá ×1000 (CTG 2013) | `price_unit` khai theo nguồn, quy đổi **lúc đọc**. Sai quy đổi ⇒ sửa 1 hàm, không phải viết lại 4.300 dòng |
| `trading_calendar` stale hai lần | Thành VIEW dẫn xuất từ `obs.index_daily` (ngày có dòng VNINDEX = phiên thật). Mỏ neo mà phải nhớ nạp thì sẽ stale lại |

## B4. Vì sao KHÔNG dùng schema (a) của `REBUILD_BLUEPRINT.md`

Blueprint chốt: **1 dòng/(date,symbol)**, cột `close` raw + `close_adj`, nhãn `raw_source`.

Cách đó chỉ **dời va chạm từ mức dòng xuống mức cột**. Job đêm vẫn `UPDATE close` được;
`raw_source` vẫn là vết sẹo ghi lại *ai ghi cuối*, không đảm bảo gì. Chính S15 đã học lại
bài này khi phải **xoá** guard `backfilled` vì nó khoá vào một cột đã hỏng.

Khác biệt còn lại so với blueprint: xem `C:/Users/OS/Lucius/Projects/PRJ quant/handoffs/` phiên S16 (bảng 11 điểm).

## B5. Kích thước

| | Dòng |
|---|---|
| Nạp một lần: SSI raw 2020-02-26+ (204 mã × ~1.600 phiên) | ~330k |
| Nạp một lần: DNSE adjusted 2015+ (204 × ~2.700) | ~550k |
| Kế thừa `LEGACY_BRONZE` | ~663k |
| **Tổng ban đầu** | **~1,5M** |
| Tăng trưởng | 204 mã × 3 nguồn × 250 phiên ≈ **150k/năm** |

**Điều kiện giữ mức này:** sau lần nạp đầu, **không kéo lại toàn lịch sử adjusted mỗi ngày**
— DNSE chỉ append bar hôm nay; kéo full-history là **thao tác kiểm định** xoay vòng, không
phải ingest. So với 21,2M dòng intraday đang có thì không đáng kể.

---

## C. Rủi ro của chính thiết kế này

Nói trước, không để phiên sau phát hiện:

1. **Độ chính xác dựng lại raw 2015→2020-02 CHƯA KIỂM.** Tập kiểm thử có sẵn: vùng chồng
   lấn 2020-02-26 → nay có **đồng thời** raw thật (SSI) và raw dựng lại (DNSE × sự kiện),
   dài 6,4 năm. Chạy hàm dựng lại trên vùng đó, so với SSI raw — khớp thì mở rộng về 2015
   là **đã được kiểm chứng**; không khớp thì biết trước khi đầu độc 5 năm lịch sử.
   **Đây là phép đo phải chạy trước khi thi công.**
2. **Append-only không chữa được quan sát SAI** — nó làm cái sai *phục hồi được và truy
   nguyên được* thay vì *huỷ diệt*. Job ghi rác vẫn ghi rác được.
3. **`REVOKE` cần tách role**, hiện mọi thứ chạy bằng `postgres`. Dính luôn việc mật khẩu
   DB nằm trần trong `C:/Users/OS/Lucius/Projects/PRJ quant/AGENTS.md` — phải xử cùng lúc.
4. **1.085 dòng `open` ngoài `[low,high]`**: loại VCI `trading` gỡ được nguồn *nghi ngờ*,
   nhưng chưa ai tìm ra dòng code sinh ra nó. Chưa coi là đã giải quyết.
5. **70 bậc gãy cơ sở giá**: có giả thuyết (thiếu sự kiện do `events()` bị cắt — E0014),
   **chưa chứng minh**. Kiểm bằng: sửa fetch → nạp đủ sự kiện → đo lại `check_price_basis --mode full`.
6. **Mốc SSI 2020-02-26**: đã bác cuộn-hằng-ngày (E0012), **chưa bác** cuộn theo quý.

---

## D. Thứ tự thi công đề xuất

| # | Việc | Cổng nghiệm thu |
|---|---|---|
| 0 | **Kiểm dựng lại raw trên vùng chồng lấn 6,4 năm** | sai số theo mã/theo năm; quyết định #6 sống hay chết |
| 1 | Sửa `build_corp_actions.py` (E0014) + nạp đủ sự kiện từ 2010 | số sự kiện/mã tăng đúng như đo tay (ACB 6 → 30) |
| 2 | Dựng `obs.*` + `REVOKE` + tách role | test: role app `UPDATE` vào `obs` **phải bị từ chối** |
| 3 | Nạp `LEGACY_BRONZE` → `obs` (`price_basis='UNKNOWN'`) | tổng dòng khớp 662.991 |
| 4 | Dựng `market.px_raw` · `market.ca` · `px_adj` | tái tạo được chuỗi adjusted của DNSE trong dung sai |
| 5 | Backfill SSI raw 2020-02-26+ (~5,6h) | **cần duyệt riêng** |
| 6 | Chuyển job fetch sang ghi `obs`; bỏ migration 200 | 3 lần chạy theo lịch liên tiếp XANH |
| 7 | Dọn: `close_adj` builder cũ, round-robin `_multi_source` | quét liên tục toàn lịch sử: 0 cú nhảy không giải thích được |

**Mỗi bước phải có test ĐỎ trước khi vá, XANH sau.** Test chưa từng đỏ = chưa chứng minh gì.

---

# §E — THIẾT KẾ VẬT LÝ

> Bổ sung 2026-07-30 sau khi chốt: **DB riêng `quant_v2`**.
> Đây là phần phải xong trước khi viết DDL — §A/§B là thiết kế logic, §E là cách nó nằm trên đĩa.

## E1. Vị trí & phân quyền

| | |
|---|---|
| **DB** | `quant_v2` — **tách hẳn** khỏi `quant_db` của ver 1 |
| Extension | `timescaledb` · `pgcrypto` (cho `digest()`) · `postgres_fdw` (đọc ver 1 khi nạp legacy) |
| Schema | `obs` · `market` · `meta` · `features` · `ops` |

**Ba role — đây là chỗ luật L1 được cưỡng chế:**

| Role | Quyền | Ai dùng |
|---|---|---|
| `v2_owner` | tạo/sửa schema, DDL | migration, người |
| `v2_writer` | **`INSERT` trên `obs.*` và `ops.*`** — không `UPDATE`, không `DELETE` | mọi job fetch |
| `v2_reader` | `SELECT` toàn bộ | model, backtest, webapp, MCP |

```sql
REVOKE UPDATE, DELETE, TRUNCATE ON ALL TABLES IN SCHEMA obs FROM v2_writer;
ALTER DEFAULT PRIVILEGES IN SCHEMA obs
  REVOKE UPDATE, DELETE, TRUNCATE ON TABLES FROM v2_writer;   -- áp cho cả bảng TẠO SAU
```

> Dòng `ALTER DEFAULT PRIVILEGES` quan trọng ngang dòng `REVOKE` — thiếu nó thì bảng nào
> thêm sau này lại mặc định mở, và luật thủng đúng chỗ không ai nhìn.
>
> Mật khẩu vào `.env`, **không** vào `AGENTS.md` hay bất kỳ file nào commit được.

**Hệ quả của việc tách DB** (so với dùng chung `quant_db`):
- Nạp legacy (P3) phải qua `postgres_fdw` — không `INSERT…SELECT` nội bộ được
- Mọi phép đối chiếu ver1 ↔ ver2 cần **hai kết nối**; script kiểm phải nhận 2 DSN
- Đổi lại: **không thể lỡ tay ghi nhầm sang ver 1** — ranh giới là vật lý, không phải kỷ luật

## E2. Hypertable & phân mảnh

| Bảng | Cột thời gian | Chunk | Vì sao |
|---|---|---|---|
| `obs.price_daily` | **`trade_date`** | 1 năm | Truy vấn luôn hỏi theo ngày giao dịch. `observed_at` chỉ dùng để chọn dòng mới nhất **trong** một `trade_date` ⇒ phân mảnh theo nó sẽ làm mọi truy vấn quét toàn bảng. ~600 dòng/ngày × 250 ≈ 150k dòng/chunk |
| `obs.price_intraday` | `ts` | 1 tháng | ~204 mã × 220 nến ≈ 45k dòng/ngày ⇒ ~1M dòng/chunk |
| `obs.index_daily` · `obs.foreign_flow` | `trade_date` | 1 năm | như trên |
| `obs.macro` | `date` | 5 năm | thưa |
| `obs.corp_action` · `obs.security_ref` · `obs.financial` · `obs.ownership` · `obs.document` | — | **bảng thường** | quá nhỏ, hypertable chỉ thêm phức tạp |

## E3. `payload_hash` — băm cái gì

```sql
payload_hash bytea GENERATED ALWAYS AS (
  digest(
    coalesce(open::text,'')  || '|' || coalesce(high::text,'')   || '|' ||
    coalesce(low::text,'')   || '|' || coalesce(close::text,'')  || '|' ||
    coalesce(volume::text,'')|| '|' || price_unit || '|' || price_basis || '|' ||
    coalesce(extra::text,'')
  , 'sha256')
) STORED
```

**Có trong hash:** giá · khối lượng · `price_unit` · `price_basis` · `extra`
**KHÔNG có trong hash:** `observed_at` · `run_id`

Lý do: hash trả lời câu *"nguồn có nói điều gì khác không?"*. Nhét `observed_at` vào thì
mỗi lần fetch đều ra hash mới ⇒ dedupe vô nghĩa, bảng phình theo số lần chạy chứ không
theo lượng thông tin. `extra::text` ổn định vì `jsonb` của Postgres đã chuẩn hoá thứ tự khoá.

**Cưỡng chế dedupe bằng chỉ mục, không bằng code ứng dụng:**
```sql
CREATE UNIQUE INDEX ON obs.price_daily (source, symbol, trade_date, payload_hash);
-- job fetch dùng: INSERT ... ON CONFLICT DO NOTHING
```

> ⚠️ **Cái giá phải trả, nói rõ ra:** nếu một nguồn đổi giá A → B rồi **quay lại A**, lần
> quay lại bị chỉ mục chặn ⇒ mất thông tin "ngày X nguồn nói A một lần nữa". Với chuỗi
> adjusted thì hệ số chỉ đi một chiều theo sự kiện nên ca này gần như không xảy ra. Nếu
> sau này thấy nó xảy ra thật, đổi sang dedupe phía ứng dụng (so hash với quan sát gần
> nhất) và **bỏ** unique index — lúc đó luật không còn được máy cưỡng chế nữa, phải đánh đổi.

## E4. Khoá và chỉ mục

```sql
-- obs.price_daily
PRIMARY KEY (source, symbol, trade_date, observed_at)
UNIQUE       (source, symbol, trade_date, payload_hash)      -- dedupe
INDEX        (symbol, trade_date DESC)                        -- truy vấn theo mã
INDEX        (trade_date, source) WHERE price_basis = 'RAW'   -- phần px_raw dùng
INDEX        (run_id)                                         -- truy nguyên theo lần chạy
```

Chỉ mục thứ ba là **chỉ mục có điều kiện** — `px_raw` chỉ quan tâm dòng `RAW`, mà dòng
`RAW` là thiểu số (SSI thôi), nên chỉ mục nhỏ hơn nhiều so với chỉ mục đầy đủ.

## E5. `ops.run_ledger` — mọi job ghi một dòng

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `run_id` | `uuid` PK | sinh lúc job khởi động; `obs.*.run_id` là FK trỏ về đây |
| `job` | text | tên script |
| `started_at` `ended_at` | timestamptz | |
| `status` | text | `OK` \| `EMPTY` \| `FAIL` — **`EMPTY` khi kỳ vọng có data = LỖI**, không phải thành công |
| `scope` | text | universe/dataset đã chạy |
| `n_expected` | int | số mã/chuỗi **kỳ vọng** — so với thực tế mới bắt được co rổ trong im lặng |
| `n_seen` `n_inserted` | int | nguồn trả về bao nhiêu / ghi được bao nhiêu (chênh = dedupe, không phải lỗi) |
| `error` | text | |
| `git_commit` | text | |

> `n_expected` vs `n_seen` là thứ ver 1 thiếu: `get_vn100()` từng tụt 100 → 30 mã và chỉ
> ghi `log.warning`. Có hai cột này thì việc đó thành **một phép so số, tự đỏ.**

## E6. `meta.source_contract` — khai ngữ nghĩa của từng nguồn

Bảng nhỏ nhưng là chỗ luật L2 sống:

| Cột | Mẫu |
|---|---|
| `source` · `dataset` | `DNSE` · `price_daily` |
| `price_unit` | `kVND` |
| `price_basis` | `ADJUSTED` |
| `depth_from` | `2012-03-20` |
| `granularity` | `1D` |
| `verified_at` · `evidence_id` | `2026-07-30` · `E0017` |
| `notes` | `không có tham số bật raw; bỏ hẳn ngày không giao dịch` |

Job fetch **đọc bảng này để gán `price_unit`/`price_basis`** thay vì hardcode. Nguồn đổi
hành vi ⇒ sửa một dòng dữ liệu + ghi `evidence_id` mới, không phải đi tìm hằng số trong code.

## E7. Ước tính đĩa

| | Dòng | Ước |
|---|---|---|
| `obs.price_daily` ban đầu | ~1,5M | ~600 MB (có `extra jsonb`) |
| `obs.price_intraday` (nếu chuyển 21,2M) | 21,2M | ~2,5 GB |
| Còn lại | < 1M | ~200 MB |
| **Tổng** | | **~3,3 GB** · tăng ~250 MB/năm |

`extra jsonb` là phần tốn nhất của `price_daily`. Chấp nhận được: nó chính là thứ khiến
6 mục dữ liệu không phải fetch lại lịch sử khi cần dùng đến.

---

## Bảo trì

- Thêm bảng mới → thêm dòng ở §A1 **và** một dòng `meta.data_catalog` (hiện catalog phủ
  17/27 đối tượng — E0015; bỏ sót đúng `corporate_actions`, `trading_calendar`,
  `tracked_universe`).
- Đổi hàm phân giải ở L1 → ghi `CHANGELOG.md`, nêu rõ số liệu downstream đổi thế nào.
- Sơ đồ dạng ảnh: `.\diagram\` theo quy ước `-vN`, **không ghi đè bản cũ**.
