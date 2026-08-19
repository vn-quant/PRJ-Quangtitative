# Runbook — dùng kho dữ liệu ver 2

> Lập 2026-08-19. **File này không lặp lại nội dung các file khác** — nó nối chúng lại và
> trả lời bốn câu: *kho có gì · hỏi thế nào · hằng ngày chạy gì · hỏng thì làm sao*.
> Chi tiết ở đâu thì ghi rõ ở từng mục.
>
> Phạm vi: **chỉ ver 2** (`quant_v2`). Ver 1 đã tắt job hằng ngày ngày **2026-08-05**.

## 0. Cần gì → đọc cái nào

| Câu hỏi | Mở |
|---|---|
| Hỏi giá thế nào, raw/adj khác gì | `OHLCV.md` §5 |
| Bảng nào có cột gì, tầng nào ghi được | `DB_DESIGN.md` |
| Lấy dữ liệu từ nguồn ra sao, endpoint nào | `INGEST_SPEC.md` (D1–D17) |
| Tải BCTC | `INGEST_SPEC.md` §D12 |
| Một nguồn cụ thể: cửa, trường, giới hạn | `SOURCE_SSI.md` · `SOURCE_DNSE.md` · `SOURCE_VCI.md` |
| Mã `C-xx` là gì | `BANG_TRA_MA_KIEM.md` |
| Nhìn toàn cảnh bằng hình | `diagram/db-va-luong-chay-v4.svg` |
| Dữ liệu nào cần lấy, còn thiếu gì | `DATA_REQUIREMENTS.md` |

---

## 1. Kho có gì — bốn tầng

Nguyên tắc gốc: **`obs` là sổ quan sát chỉ-thêm**; Postgres cưỡng chế cấm sửa. Mọi phán xét
chất lượng nằm ở tầng đọc, không nằm lúc ghi.

| Tầng | Là gì | Ai ghi |
|---|---|---|
| **`obs`** | ghi **cái đã thấy**, chưa lọc, chưa quy đổi. Khoá gồm `source` và `observed_at` | job nạp |
| **`market`** | **tầng đọc** — đã khoanh nguồn, quy về VND, chỉ mã đang cập nhật | không ai ghi tay |
| **`meta`** | khai báo: nguồn, vai trò, đơn vị, universe, lịch giao dịch | **người** |
| **`ops`** | nhật ký chạy, cách ly, trạng thái | job |

Bảng chính trong `obs`: `price_daily` · `corp_action` · `index_daily` · `index_member`.
`observed_at` (**lúc ta thấy**) khác `trade_date` (**ngày giao dịch**) — đây là thứ cho phép
cùng một ngày có nhiều quan sát mà không mất bản nào.

## 2. Hỏi dữ liệu thế nào

**Đọc qua `market.*`. Không truy vấn thẳng `obs.*`** — `obs` chưa khoanh nguồn, chưa quy đổi
đơn vị, còn lẫn số suy ra.

| Gọi | Trả về | Đơn vị |
|---|---|---|
| `market.px_raw` | giá **thô** | VND |
| `market.px_adj` | giá điều chỉnh, **mặt bằng hôm nay** | VND |
| `market.px_adj_as_of(ngày)` | giá điều chỉnh, **mặt bằng ngày chỉ định** | VND |
| `market.ca` | sự kiện điều chỉnh giá | — |
| `market.factor_exchange` | hệ số sàn công bố mỗi phiên | — |

Ba điều bắt buộc nhớ:

- **Backtest dùng `px_adj_as_of(ngày_đó)`**, không dùng `px_adj`. `px_adj` là mặt bằng hôm
  nay — dùng cho quá khứ là dùng thông tin từ tương lai.
- **Mọi truy vấn giá phải lọc `source`.** Kho chứa nhiều nguồn với **đơn vị khác nhau**
  (DNSE kVND · SSI_SP VND). Một truy vấn quên lọc đã so neo `22.600 VND` với `22,6 kVND`
  ⇒ tỷ lệ `0,001` ⇒ **chặn oan 2.610 dòng đúng**.
- **Không có dạng scalar.** Không có hàm trả một số cho `(mã, ngày)` — engine backtest gọi
  từng dòng trên hàng triệu điểm sẽ chết. Chỉ có dạng bảng, tính một lượt, nạp thẳng vào Polars.

**Giá trị không đáng tin thì trả `NULL`**, không trả số gần đúng kèm cờ — *"thiếu còn hơn
sai"*. `px_raw` trả `NULL` cho O/H/L ngoài HOSE, vì phép suy `raw_x = adj_x ÷ (adj_close/raw_close)`
chỉ đúng ở HOSE; HNX và UPCoM hỏng (E0061, E0063).

Chi tiết: `OHLCV.md` §5.

## 3. Chạy hằng ngày

**Một điểm vào duy nhất:**

```
C:\Users\OS\Lucius\Projects\PRJ Quantitative Investment\3. Build\chay_hang_ngay.bat
```

Chín bước, chạy tuần tự:

| # | Lệnh | Việc |
|---|---|---|
| 1 | `daily_update.py` | giá **nguồn CHÍNH** — không tham số, tự đọc vai trò từ `meta.source_contract` |
| 2 | `daily_update.py --source DNSE` | giá **đối chứng** — cùng file, cùng mọi cổng |
| 3 | `load_ssi_sp.py --window` | giá **thô + trần/sàn** — nguồn duy nhất có trần/sàn |
| 4 | `load_index.py` | 5 rổ chỉ số — **chạy trước** vì lịch giao dịch dẫn xuất từ đây |
| 5 | `load_index_member.py` | **ảnh chụp** thành phần rổ — ngày nào không chụp là mất vĩnh viễn |
| 6 | `daily_events.py` | sự kiện DN từ **ba nguồn** (DNSE · VCI · Vietstock) |
| 7 | `lam_moi_tang_doc.py` | dựng lại bảng `market.px_adj` trong **một giao dịch** + C-19 |
| 8 | `test_bat_bien.py` | 13 bất biến cấu trúc I1–I13 |
| 9 | `status.py --ghi` | dead-man 36h · tuổi **từng nguồn** · tồn cách ly |

**Mã thoát lấy của `status.py`: `0` sạch · `2` có ĐỎ.** Task Scheduler ghi lại số này, nên
xem `LastTaskResult` là biết đêm qua có vấn đề không.

> ⚠️ **Phải nạp cả bốn nguồn.** Bản đầu chỉ nạp DNSE; ngày 2026-08-05 phát hiện DNSE tươi tới
> 05/08 nhưng iBoard, SSI_SP và chỉ số đứng ở 31/07 — mà `px_adj` ưu tiên iBoard, nên
> **187/196 mã đóng băng 5 ngày**. `ops.status` vẫn báo xanh vì nó hỏi `max(trade_date)` **gộp
> mọi nguồn**. Đó là lý do bước 9 phải xét tuổi **từng nguồn**.

Job **không gán cứng tên nguồn chính** — đọc từ `meta.source_contract.vai_tro`. Muốn đổi
nguồn chính thì sửa DB, không sửa code.

## 4. Kiểm thế nào

Bốn tầng, mỗi tầng bắt một loại lỗi khác nhau:

| Tầng | Khi nào | Mã |
|---|---|---|
| **A** lúc bắt | trong vòng lặp fetch | C-01 · C-02 · C-03 · C-04 |
| **B** lúc ghi | Postgres cưỡng chế mỗi `INSERT` | C-05 · C-06 · C-07 |
| **C** sau khi chạy | cuối job | C-08 · C-09 · C-10 · C-11 · C-12 · **C-19** |
| **D** đối chiếu độc lập | định kỳ | C-13 … C-18 |

**Đã cài 13 · hoãn 6.** Tra một mã cụ thể: `BANG_TRA_MA_KIEM.md`.

**Hai phép kiểm mạnh nhất — vì chúng dùng số không do ta tính:**

- **C-15** — hệ số ta tính so với **hệ số sàn công bố** `(trần+sàn)/2 ÷ close phiên trước`.
  Đây là *đáp án*, thứ duy nhất trong hệ không phải suy diễn của mình.
- **C-11** — `px_adj(d, as_of=d) == px_raw(d)`. So hai thang giá độc lập, nên sai lệch nào của
  công thức điều chỉnh cũng lộ.

Mọi phép kiểm còn lại chỉ soi **nhất quán nội tại** — mà hệ hoàn toàn có thể *nhất quán sai*,
đã xảy ra ba lần trong dự án này.

**Lệnh chạy tay:**

```
python status.py                 # trạng thái kho
python check_completeness.py     # đủ mã / đủ ngày / bar trùng
python check_c11_c15.py          # hai phép kiểm mạnh nhất
python test_bat_bien.py          # 13 bất biến cấu trúc
python test_spec_vs_code.py      # đặc tả có khớp code không
python test_golden.py            # bộ dữ liệu vàng
```

Chạy trong `3. Build\code\`. **Interpreter phải là của ver 1** — xem §7.

**`test_bat_bien.py` khác `test_spec_vs_code.py`:** file kia hỏi *"đã cài chưa"*, file này hỏi
*"đã áp cho MỌI chỗ nó chạm chưa"*.

## 5. Hỏng thì làm gì

| Triệu chứng | Nghĩ tới |
|---|---|
| `status.py` trả mã thoát `2` | đọc dòng ĐỎ trước, đừng chạy lại job |
| Một nguồn đứng ngày, các nguồn khác vẫn tươi | đúng ca 05/08 — xét tuổi **từng nguồn**, không xét gộp |
| Giá quá khứ đổi | nguồn chia lại lịch sử. Job tự phát hiện bằng cách so **chính dữ liệu**, ngưỡng 0,2% |
| Chênh dưới 0,2% | làm tròn 2 chữ số kVND, **không phải** điều chỉnh giá |
| Bar tự mâu thuẫn (`low > open`…) | C-01 **ghi nhận, không xoá cột**. Cổng chỉ được bỏ cột khi có giá trị độc lập chỉ đích danh cột đó |
| Nhiều mã cùng hỏng một ngày | C-19 — so với **nền cục bộ**, không dùng ngưỡng tuyệt đối |
| `px_adj_as_of` trả rỗng | bản cũ lọc `observed_at ≤ as_of` nên rỗng với mọi mốc trước 2026-08-01; đã viết lại 12/08 |

Giá trị gốc của dòng bị chặn nằm ở `ops.quarantine` — **không mất gì**, dòng vẫn trong `obs`.

Ma trận triệu chứng đầy đủ và bốn mức leo thang: `OHLCV.md` §9.

## 6. Đang thiếu gì

Không mục nào cho ra **số sai** — đều là **thiếu dữ liệu**:

| | |
|---|---|
| 🔴 Vietcap chưa nạp giá | 0 dòng, dù phủ từ 2006 và có dữ liệu sạch đúng những ngày iBoard hỏng 2008–09 |
| 🟡 GTGD thiếu 11,9% | 74.196 dòng; 55.441 dòng trước 2020-02-26 thì **không nguồn nào** có |
| 🟡 O/H/L thô | HNX 0% · UPCoM 0% — sẽ có một phần nhờ SSI FastConnect (từ 2020-02-26) |
| 🟡 Hệ số sự kiện | chỉ 83,7% kiểm chứng được — gốc của C-13 đỏ vĩnh viễn |
| 🟡 Ảnh chụp rổ | bỏ 178 mã ngoài `meta.securities` (TASK-008) |
| 🟡 Độ rộng thị trường | mục 4.2 trống — cần gọi SSI `DailyIndex`, hiện chưa cài |
| 🟡 BCTC | có đặc tả (§D12) nhưng **chưa có loader** |

## 7. Phụ thuộc ver 1 — chưa gỡ

Ver 2 **chưa tự chạy được**:

- `chay_hang_ngay.bat` dùng interpreter `C:\Users\OS\Lucius\Projects\PRJ quant\.venv\Scripts\python.exe`
  — ver 2 chưa có venv riêng.
- `load_ssi_fc_price.py` nạp `.env` của ver 1 và `import ssi_fc` từ `PRJ quant\01_fetch\`
  bằng **đường dẫn tuyệt đối cứng**.

Hệ quả: **xoá hoặc di chuyển thư mục ver 1 sẽ làm gãy job hằng ngày của ver 2.** Job hằng
ngày của ver 1 thì đã tắt từ 2026-08-05, nhưng thư mục phải giữ nguyên.

## 8. Chỗ tài liệu đang lệch thực tế

Ghi ra để người đọc sau không tin nhầm:

| Chỗ | Lệch gì |
|---|---|
| `OHLCV.md` §6 | mô tả job **16:15** với bộ bước khác. Bản thật là `chay_hang_ngay.bat`, **9 bước**, và ver 1 (16:15) đã tắt |
| `INGEST_SPEC.md` §D1 | cột "Dùng" lệch code **12 trường** — 10 trường spec đánh chưa dùng mà code có ghi, 2 trường ngược lại |
| `DATA_REQUIREMENTS.md` | mục 1.7 · 1.8 · 1.9 · 2.1–2.3 vẫn đánh 🔴 "chưa lấy" nhưng ver 2 đã lấy đủ |
| `diagram/README.md` | bảng version dừng ở `db-va-luong-chay-v2`; thực tế đã có `v3` (12/08) và `v4` |

**Quy tắc để không tái phát:** mỗi tài liệu ghi ở dòng đầu **nó mô tả version nào và đo ngày
nào**. Hai kết luận sai trong phiên 19/08 đều xảy ra vì dòng đó không tồn tại.
