# 2026-08-19 · Techdoc từng nguồn dữ liệu — SSI · DNSE · VCI, và đường lấy BCTC

**Trạng thái:** ✅ xong · **Người làm:** Claude Code

> Tài liệu kỹ thuật đầy đủ **không nằm ở repo này** — chúng lưu trong repo project
> `vn-quant/prj-quantitative-investment`, commit `3ba5492`, thư mục `2. Data design/main/`.
> File này chỉ ghi *đã làm gì, kết luận gì, còn treo gì*.

## 1. Mục tiêu

Lập tài liệu tra cứu **một file cho mỗi nguồn dữ liệu**: cửa nào gọi được, trường nào có,
giới hạn gì, ánh xạ sang bảng nào trong kho.

Hai yêu cầu bổ sung giữa chừng: ghi rõ **version API** để khi nhà cung cấp ra bản mới còn
biết mình đang ở đâu; và liệt kê **tới từng trường**, không mô tả chung chung.

Nhánh phát sinh: xác minh **đường lấy báo cáo tài chính**, so chất lượng giữa VCI,
`vnstock` và `vnfinancialdata`.

## 2. Kết quả

| Tài liệu | Dòng | Trạng thái |
|---|---|---|
| `SOURCE_SSI.md` | 335 | ✅ viết lại từ nguồn v2 |
| `SOURCE_DNSE.md` | 160 | ✅ viết lại + gọi xác minh trực tiếp |
| `SOURCE_VCI.md` | 316 | ✅ viết lại từ nguồn v2 + đo trực tiếp |
| `INGEST_SPEC.md` §D12 | +102 −14 | ✅ techdoc tải BCTC, thay hoàn toàn |
| `SOURCE_VNSTOCK.md` | 70 | 🔴 **chưa làm lại** — chủ động bỏ khỏi phạm vi |

## 3. Kết luận đáng nhớ

**Version API — ba bề mặt không đánh version.** SSI iBoard, DNSE `senses-api`, VCI trading
đều không có đoạn `/vN/` trong đường dẫn, nghĩa là **đổi hành vi sẽ không kèm đổi URL**.
Ba bề mặt còn lại (SSI FastConnect `/api/v2/`, DNSE chart `/chart-api/v2/`, VCI IQ
`/iq-insight-service/v1/`) ít nhất còn cho tín hiệu.

**BCTC lấy từ VCI IQ, không phải vnstock.** Cơ chế quan trọng nhất: `/metrics` là **theo
từng mã**, trả về khuôn của đúng loại hình doanh nghiệp. Tiền tố trường mã hoá loại hình —
`*a` thường · `*b` ngân hàng · `*i` bảo hiểm · `*s` chứng khoán — và khuôn một mã **trộn
nhiều tiền tố**, nên không được lọc theo tiền tố.

**VCI hơn `vnfinancialdata` ở mọi ngành** về số chỉ tiêu, hơn rõ ở bảng cân đối khối tài
chính (VCB 66/39 · VND 96/69), và **độc quyền toàn bộ thuyết minh** (78–149 chỉ tiêu tuỳ mã;
`vnfinancialdata` không có `NOTE` nào). Giá trị hai nguồn khớp **tới từng đồng**.

`vnfinancialdata` còn đúng **một** lý do để dùng: lịch sử **2011–2017** mà VCI không có
(VCI bắt đầu 2018). Kèm ba hạn chế: không có `publicDate` ⇒ không dựng được point-in-time ·
chỉ theo năm · phủ ~50% HOSE và **thiếu hẳn SSI**.

**`vnstock` hỏng cả hai bản.** 4.0.6 lỗi `UnboundLocalError: hosting_service` — vỡ trước khi
gọi mạng. 3.4.1 (bản đang cài) lỗi `KeyError: 'data'` — không parse nổi phản hồi VCI hiện
tại. Cùng lúc gọi thẳng VCI thì lấy đủ. **Cửa tốt, chỉ lớp bọc hỏng** — củng cố luật
"KHÔNG dùng vnstock", và project v2 hiện không có tham chiếu nào tới nó.

## 4. Hai kết luận sai đã rút

Cả hai cùng một gốc: **đọc số bằng khuôn của thứ khác**.

1. **"Khối ngoại không được dùng"** — chép cột "đang dùng" từ tài liệu mô tả pipeline **v1**
   sang tài liệu **v2**. Thực tế v2 ghi đủ 5 trường khối ngoại.
2. **"VCI thiếu dòng ngân hàng"** — lấy `/metrics` của FPT đọc bản ghi của VCB, rồi đếm theo
   tiền tố `isa*`. Dòng ngân hàng nằm ở `isb*`. Kéo theo khuyến nghị sai *"dùng
   vnfinancialdata cho khối tài chính"* — đã rút.

**Bài học:** với API dùng mã trường mờ nghĩa (`isa1`, `bsb27`), **không bao giờ suy tên
trường bằng bảng của mã khác** — luôn lấy từ điển của chính đối tượng đang đọc.

Bài học thứ hai, về tài liệu: một bảng trộn **thuộc tính nguồn** (tên trường, đơn vị — dùng
chung được giữa các phiên bản) với **thuộc tính code** ("đang dùng" — gắn với một phiên bản
cụ thể) sẽ khiến người đọc sau trích nhầm. Tách hai loại đó ra.

## 5. Còn treo

- 🔴 `SOURCE_VNSTOCK.md` chưa làm lại — vẫn dựa tài liệu v1, chưa có bằng chứng gọi thật.
- `DATA_REQUIREMENTS.md` lạc hậu: mục 1.7 · 1.8 · 1.9 · 2.1–2.3 vẫn đánh 🔴 "chưa lấy" nhưng
  v2 đã lấy đủ.
- **Đặc tả lệch code 12 trường** ở D1 — chưa rõ `test_spec_vs_code.py` có phủ tới mức từng
  trường không.
- **Sáu cửa VCI IQ có đặc tả nhưng chưa có loader**: `financial-statement` ·
  `statistics-financial` · `insider-transaction` · `shareholder` · `sectors/icb-codes` ·
  `news-events-for-chart`. Nhóm tham chiếu cũng chưa có loader — bảng `securities` hiện nạp
  tay từ CSV.
- **`DailyIndex` chưa gọi** ⇒ độ rộng thị trường vẫn trống, dù đó là cửa duy nhất có
  `Advances/Declines/NoChanges/Ceilings/Floors`.
- Số đo phiên này **chưa có mã `E####`** — cần đăng ký vào sổ bằng chứng.
- **Phụ thuộc v1 chưa gỡ**: loader SSI của v2 nạp `.env` và `import ssi_fc` từ project v1
  bằng đường dẫn tuyệt đối cứng; v2 vẫn chưa có venv riêng.

## 6. Nơi lưu tài liệu liên quan

Repo `vn-quant/prj-quantitative-investment` (private), commit `3ba5492`:

| Cần gì | Mở |
|---|---|
| Cửa · trường · giới hạn · ánh xạ của một nguồn | `2. Data design/main/SOURCE_{SSI,DNSE,VCI}.md` |
| Cách tải BCTC | `2. Data design/main/INGEST_SPEC.md` §D12 |
| Bản handoff kỹ thuật đầy đủ | `handoffs/S3-techdoc-nguon-du-lieu.md` |
