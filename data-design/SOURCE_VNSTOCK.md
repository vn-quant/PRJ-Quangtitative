# vnstock — vì sao KHÔNG nằm trong luồng dữ liệu

> Lập 2026-08-19. Bộ 4 file nguồn: `SOURCE_SSI` · `SOURCE_DNSE` · `SOURCE_VCI` ·
> `SOURCE_VNSTOCK`.
>
> **File này không phải tài liệu hướng dẫn dùng vnstock.** Nó là hồ sơ đánh giá, giữ lý do
> và số đo để lần sau không phải cân nhắc lại từ đầu.

**vnstock là thư viện bọc, không có dữ liệu riêng.** Nó gọi lại chính VCI/MSN/KBS/DNSE rồi
cắt bớt trường. Luật hiện hành (`DATA_SOURCE_MAP.md`, chốt 2026-07-30): **mọi đường lấy dữ
liệu gọi thẳng nguồn, không qua vnstock.**

Cơ sở của luật, đã đo: bỏ vnstock **không mất khả năng nào** — 11/16 đường VCI IQ chạy đầy
đủ bằng `requests` thuần, kết quả trùng khớp với đường qua vnstock (FPT: 79 sự kiện DIV+ISS
ở cả hai đường) — E0016.

## Đo bản 4.0.6 — 2026-08-19

Cài `vnstock==4.0.6` vào venv tách biệt (Python 3.11, thư mục scratchpad), **không đụng tới
môi trường chạy pipeline** (đang giữ 3.4.1). Chưa đăng ký `register_user`, chưa gọi mạng
tới vendor. Các số dưới đây chưa được cấp mã `E####` — cần đăng ký vào sổ bằng chứng.

| Quan sát | Chi tiết |
|---|---|
| **Module vỡ ngay khi import** | `import vnstock.core.constants` → `AttributeError: TCBS`. `constants.py` còn tham chiếu `_DataSource.TCBS.value` trong khi `TCBS` đã bị bỏ khỏi enum. Lỗi có sẵn trong bản phát hành. |
| **Nguồn mặc định đổi** | `Quote(source=DataSource.KBS)` — trước là VCI. Đổi mặc định mà không đổi chữ ký hàm. |
| **TCBS biến mất** | `DataSource` = `KBS · VCI · MSN · DNSE · BINANCE · FMP · FMARKET`. |
| **`Screener` biến mất** | Có ở 3.4.1, không còn ở 4.0.6. Breaking, không changelog. |
| **Thêm tầng gói tài trợ** | Xuất hiện `check_sponsor_package`, `setup_agent`; đã có sẵn `register_user`, `change_api_key`. |
| **Không có `__version__`** | Phải dùng `importlib.metadata.version("vnstock")`. |
| **SSI vẫn không phải nguồn** | Không có mặt trong `DataSource` ở cả 3.4.1 lẫn 4.0.6. |
| **Banner khi import** | In thông báo phiên bản mới lúc `import` — có hoạt động mạng ở thời điểm nạp thư viện. |

Kết luận: 4.0.6 **củng cố** luật chứ không lật lại. Một bản phát hành major ship kèm module
tự vỡ khi import, bỏ lớp `Screener`, đổi nguồn mặc định, và không có changelog — đúng loại
rủi ro mà luật được lập ra để tránh.

## Lịch sử hỏng — vì sao không tin lớp bọc này

| Thời điểm | Chuyện gì |
|---|---|
| 2025-08-31 | Bỏ `Vnstock().stock()`. `save_financials` **ghi 0 dòng, im lặng** — không lỗi, không cảnh báo. |
| 4.0.5 | Phát hành **không có changelog**. |
| 4.0.6 | `vnstock.core.constants` vỡ khi import; `Screener` bị gỡ. |

Vấn đề không phải một lỗi cụ thể mà là **cơ chế**: thêm một tầng phái sinh giữa hệ và nguồn
là thêm một chỗ có thể lặng lẽ đổi hành vi, ở nhịp phát hành mình không kiểm soát.

## Ánh xạ sang schema

**Không có.** Không bảng nào trong `obs.*` được khai `source='vnstock'`, và
`meta.vendor_door` không có cửa nào của vnstock — vì vnstock không sở hữu cửa nào, nó gọi
cửa của VCI/MSN/KBS/DNSE. Khai nó như một nguồn sẽ làm hỏng đúng thứ `source_contract` sinh
ra để giữ: truy vết một con số về đúng cửa đã sinh ra nó.

## Vai trò còn lại

**Công cụ đối chứng thủ công**, chạy trong venv riêng, không nằm trong pipeline.

Một chỗ nó vẫn có ích: `Company` của 4.0.6 có `officers` · `subsidiaries` · `affiliate` ·
`shareholders` · `insider_trading` · `capital_history`. Mục **5.7b ban lãnh đạo / công ty
con** đang treo vì `/v1/company` của IQ trả 404 và chưa dựng được truy vấn GraphQL tới
`trading.vietcap.com.vn/data-mt/graphql`. Đọc mã nguồn vnstock để **biết truy vấn phải viết
thế nào** là hợp lệ — đó là dùng nó làm bản đồ, không phải làm phụ thuộc.

## Nếu sau này muốn lật luật

Cần đủ ba điều, thiếu một thì không: (1) một mục dữ liệu mà **chỉ** vnstock lấy được và đã
thử gọi thẳng nguồn thất bại; (2) bản phát hành có changelog thật; (3) cách phát hiện khi nó
đổi hành vi mà không báo. Hiện chưa có điều nào.
