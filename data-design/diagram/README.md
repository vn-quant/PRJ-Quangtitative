# Sơ đồ — bản hiện hành

Sáu sơ đồ, mỗi cái trả lời một câu hỏi khác nhau. Bản `.svg` để phóng to đọc chữ,
`@2x.png` để dán vào tài liệu hay xem nhanh trên máy khác.

| Sơ đồ | Trả lời câu hỏi |
|---|---|
| **`db-va-luong-chay-v4`** | **Dữ liệu nằm ở đâu và hằng ngày chạy gì?** — 4 tầng + 9 bước, kèm tên file code, phiên bản API từng cửa, và đường lấy BCTC. **Bản hiện hành** |
| `db-va-luong-chay-v3` 🔴 | CŨ — thay bằng `v4` |
| `db-va-luong-chay-v2` 🔴 | CŨ — thay bằng `v4` |
| `van-de-ver1-v1` | **Ver 1 hỏng vì gì?** — chẩn đoán gốc, 6 nhánh hỏng, khuôn lỗi chung, và khuôn nào còn sống trong ver 2 |
| `daily-check-v1` | **Kiểm thế nào?** — ba tầng gate (lúc bắt / lúc ghi / sau khi chạy) + bốn mức leo thang |
| `on-change-v1` | **Có thay đổi thì sao?** — 6 tình huống: sự kiện mới · nguồn sửa giá cũ · bậc gãy không rõ · nguồn đổi API · đổi schema · số đã công bố hoá ra sai |
| `db-redesign-v1` 🔴 | **CŨ** — thay bằng `db-va-luong-chay-v2` |
| `daily-run-v1` 🔴 | **CŨ** — thay bằng `db-va-luong-chay-v2` |

Người mới đọc theo đúng thứ tự trên: `van-de-ver1-v1` trước (vì sao có ver 2), rồi
`db-va-luong-chay-v2` (ver 2 là cái gì), rồi hai cái còn lại.

## Luật version

Tên file mang hậu tố `-vN`. Sửa nội dung thì tạo `-v(N+1)` và **giữ nguyên bản cũ** —
số liệu trên sơ đồ gắn với một ngày đo cụ thể, bản cũ vẫn đúng với ngày của nó.
**Bản số lớn nhất ở thư mục này là bản hiện hành**; bản cũ chuyển sang `..\..\archive\diagram\`.

| File | Ngày | Nội dung |
|---|---|---|
| `db-redesign-v1` | 2026-07-30 | Kiến trúc 4 tầng `obs` / `market` / `meta` / `features`, `ops` cắt ngang |
| `daily-run-v1` | 2026-07-31 | Luồng chạy hằng ngày |
| `daily-check-v1` | 2026-07-31 | Ba tầng gate + leo thang |
| `on-change-v1` | 2026-07-31 | Sáu tình huống thay đổi |
| **`db-va-luong-chay-v2`** | **2026-08-09** | **Gộp `db-redesign-v1` + `daily-run-v1` và dựng lại từ DB thật.** Đổi so với v1: `SSI_IBOARD` là nguồn CHÍNH (v1 vẽ DNSE) · khoá một nguồn cho toàn chuỗi mỗi mã · job 8 bước 17:30 (v1: 7 bước 16:15) · thêm `SSI_SP` giá thô + trần/sàn · thêm `index_member` ảnh chụp rổ · thêm 10 bất biến I1–I10 · thêm tên file code + logic từng bước · số liệu đo từ `quant_v2` ngày 2026-08-09 |
| **`db-va-luong-chay-v3`** | **2026-08-12** | Bảy thay đổi sau đợt sửa 12/08: `px_adj` thành BẢNG · lớp tính `px_adj_calc` · `px_adj_as_of` viết lại bằng một phép chia (rỗng → chạy tới 2000; >400s → 4,0s) · GTGD sai 37,0% → 0,59% · C-01 quy trách nhiệm từng bar thay vì xoá cột · thêm C-19 cắt ngang theo ngày · job 8 → 9 bước. **Chưa từng được đăng ký vào bảng này — bổ sung ngày 19/08** |
| **`db-va-luong-chay-v4`** | **2026-08-19** | **Bản hiện hành.** Bốn đổi so với v3, đều từ đo trực tiếp: ① vnstock từ *"chỉ dùng cho BCTC"* → **cấm hoàn toàn** (4.0.6 và 3.4.1 đều không lấy được BCTC) · ② thêm đường lấy **BCTC** = VCI IQ, kèm cơ chế khuôn theo ngành và bẫy `/metrics` theo từng mã · ③ thêm **phiên bản API từng cửa** — 3 cửa không đánh version nên đổi ngầm được · ④ ghi rõ cửa nào ver 2 **thực sự gọi**, đọc từ code thay vì suy từ đặc tả. Thêm `obs.financial` (chưa có dữ liệu) và ghi ver 1 đã tắt job 05/08 |
| **`van-de-ver1-v1`** | **2026-08-09** | **Mới.** Ver 1 hỏng vì gì: chẩn đoán gốc → 6 nhánh (bảng sửa được · trộn nguồn · trộn thang giá · nuốt lỗi im lặng · tự suy thay vì đo · dòng trùng) → khuôn lỗi chung → 4 khuôn còn sống trong ver 2 (đo trực tiếp trên DB, không trích tài liệu) |

> ⚠️ **`db-redesign-v1` và `daily-run-v1` đã CŨ** — vẽ 30–31/07, trước hai thay đổi lớn ngày
> 02/08 (khoá một nguồn cho toàn chuỗi mỗi mã · `SSI_IBOARD` thay `DNSE` làm nguồn adj chính).
> Dùng `db-va-luong-chay-v2` thay cho cả hai. Chi tiết đổi gì: `..\..\archive\CHANGELOG.md`.
>
> `daily-check-v1` và `on-change-v1` vẫn dùng được — chúng tả cơ chế, không tả nguồn.

## Sơ đồ không nằm ở đây

| | |
|---|---|
| `build-steps-v1` — thứ tự thi công P0.5→P8 | `..\..\archive\diagram\` — nó đi kèm `DB_BUILD_RUNBOOK.md`, cả hai là bản ghi tiến độ |
| Schema **ver 1 đang chạy production** | `C:\Users\OS\Lucius\Projects\PRJ quant\06_document\diagram\db-schema\diagram.svg` — tả hệ ở project cũ, để nguyên ở đó |

## Nguồn số liệu trên `db-redesign-v1`

Mọi con số trích sổ bằng chứng
`C:\Users\OS\Lucius\Projects\PRJ quant\09_other\evidence\INDEX.md`: mốc SSI `2020-02-26`
(E0012) · sự kiện VCI từ 2010 (E0013) · trolyluat 2011-01-04 (E0022) · 204 mã (E0021) ·
663k dòng `LEGACY_BRONZE` (S14 mục D).
