# Thiết kế dữ liệu — bản dùng

Đây là **tài liệu tra cứu cuối cùng**. Mở file ở đây để biết hệ đang chạy thế nào và
dùng nó ra sao. Không có tiến độ, không có lịch sử sửa đổi, không có ghi chép quá trình.

Quá trình nằm ở `..\archive\` — đọc khi cần biết **vì sao** một quyết định lại như vậy.

## Cần gì → mở cái nào

| Cần biết | Mở |
|---|---|
| **Giá ngày dùng thế nào** — định nghĩa raw/adj, công thức, gọi view nào, 18 phép kiểm, xử lý lỗi | `OHLCV.md` |
| **Thiết kế bảng** — tầng `obs` / `market` / `meta` / `ops`, sơ đồ, thứ tự thi công | `DB_DESIGN.md` |
| **Dữ liệu nào cần lấy** — 56 mục, 8 nhóm, danh sách mã | `DATA_REQUIREMENTS.md` |
| **Nguồn nào cho được dữ liệu nào** — map từng mục sang API | `DATA_SOURCE_MAP.md` |
| **Một nguồn cụ thể: cửa, trường, giới hạn, ánh xạ sang bảng** | `SOURCE_SSI.md` · `SOURCE_DNSE.md` · `SOURCE_VCI.md` |
| **Vì sao không dùng vnstock** — hồ sơ đánh giá, số đo 4.0.6 | `SOURCE_VNSTOCK.md` |
| **Lấy cụ thể thế nào** — endpoint, từng cột và ý nghĩa, lịch chạy | `INGEST_SPEC.md` |
| **Mã `C-01`…`C-18` là gì** — một dòng mỗi mã | `BANG_TRA_MA_KIEM.md` |
| **Sơ đồ** — 4 cái: dữ liệu ở đâu · chạy hằng ngày · kiểm thế nào · có thay đổi thì sao | `diagram\` — đọc `README.md` trong đó |

## Hỏi giá — đường vào duy nhất

Mọi thứ ngoài kho đọc qua `market.*`. **Không truy vấn thẳng `obs.*`.**

| Gọi | Trả về |
|---|---|
| `market.px_raw` | giá thô, VND |
| `market.px_adj` | giá điều chỉnh, mặt bằng **hôm nay**, VND |
| `market.px_adj_as_of(ngày)` | giá điều chỉnh, mặt bằng **ngày chỉ định** — **backtest dùng cái này** |
| `market.ca` | sự kiện điều chỉnh giá |
| `market.factor_exchange` | hệ số sàn công bố mỗi phiên |

Chi tiết và cạm bẫy: `OHLCV.md` §5.

## Chạy và kiểm

| Việc | Lệnh (chạy trong `3. Build\code\`) |
|---|---|
| Cập nhật hằng ngày | `python daily_update.py` |
| Cập nhật bảng sự kiện | `python daily_events.py` |
| Xem trạng thái kho | `python status.py` |
| Kiểm C-11 và C-15 | `python check_c11_c15.py` |
| Kiểm đủ mã / đủ ngày / bar trùng | `python check_completeness.py` |
| **Kiểm đặc tả có khớp code không** | `python test_spec_vs_code.py` |
| Bộ dữ liệu vàng | `python test_golden.py` |

> ⚠️ **Ver 2 chưa có venv riêng.** `python` của hệ thống **không** có `psycopg`. Dùng
> interpreter của ver 1:
> `C:\Users\OS\Lucius\Projects\PRJ quant\.venv\Scripts\python.exe`

Log mỗi lần chạy: `logs\INDEX.csv` (một dòng/lần) và `logs\<ngày>_<job>.csv` (chi tiết).

## Luật giữ thư mục này không mục

1. **File ở đây mô tả trạng thái cuối, không mô tả hành trình.** Câu kiểu *"bản đầu làm sai
   vì…"*, *"đã sửa lần 4"*, *"còn treo 3 việc"* thuộc về `..\archive\`.
2. **Đổi thiết kế → sửa file ở đây, rồi ghi một dòng vào `..\archive\CHANGELOG.md`.**
   Không ghi lịch sử vào chính file thiết kế.
3. **Trạng thái cài đặt không viết tay vào tài liệu** — để `test_spec_vs_code.py` in ra.
   Số liệu viết tay là số liệu sẽ cũ đi mà không ai biết.
4. **Yêu cầu gốc nằm ở `0. Requirements\URD.md`.** File ở đây tả *cách làm*; URD tả
   *phải đạt cái gì*. Mâu thuẫn thì URD đúng.
