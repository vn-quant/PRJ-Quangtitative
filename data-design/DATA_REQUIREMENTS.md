# Danh mục dữ liệu CẦN LẤY

> **Lập 2026-07-30 (S16).** Tài liệu **1/3** của bộ redesign DB.
> (1) cần lấy gì ← *file này* · (2) nguồn nào cho được (`C:/Users/OS/Lucius/Projects/PRJ quant/06_document/DATA_SOURCES.md`) · (3) sơ đồ bảng.
>
> File này chỉ **liệt kê**. Không nói lấy ở đâu, không nói lưu thế nào.
>
> Cột **Có**: ✅ đang lấy · 🟡 có nhưng thiếu/hỏng · 🔴 chưa lấy · ⏸ đã dựng bảng, để rỗng

---

## 1. Giá & giao dịch

| # | Dữ liệu | Độ mịn | Có |
|---|---|---|---|
| 1.1 | **OHLCV ngày — giá THÔ** (không hiệu chỉnh) | mã × ngày | 🟡 chỉ từ 2020-02-26 |
| 1.2 | **OHLCV ngày — giá ĐÃ ĐIỀU CHỈNH** | mã × ngày | 🟡 lẫn vào chung một cột với giá thô |
| 1.3 | **OHLCV intraday 1 phút** | mã × phút | ✅ 21,2M dòng |
| 1.6 | **Giá trị giao dịch (VND)** | mã × ngày | 🔴 |
| 1.7 | **Trần / sàn / giá tham chiếu** | mã × ngày | 🔴 |
| 1.8 | Giá bình quân phiên | mã × ngày | 🔴 |
| 1.9 | Khối lượng & giá trị **thoả thuận** (tách khỏi khớp lệnh) | mã × ngày | 🔴 |

> **Bỏ khỏi danh sách lấy** (quyết định 2026-07-30):
> - **1.4 khung 5m/15m/30m/1H** — dẫn xuất được từ 1.3 (nến 1 phút). Không lấy riêng.
> - **1.5 khung tuần/tháng** — dẫn xuất được từ 1.2 (nến ngày). Không lấy riêng.
>
> **Hoãn sang thiết kế sau:** 1.10 số lệnh mua/bán · 1.11 tick khớp lệnh + chiều ·
> 1.12 sổ lệnh L2. Ba mục này gắn với hạ tầng streaming, tách riêng.

## 2. Dòng tiền

| # | Dữ liệu | Độ mịn | Có |
|---|---|---|---|
| 2.1 | **Khối ngoại** — KL & giá trị mua/bán, mua ròng | mã × ngày | ✅ |
| 2.2 | **Room ngoại** còn lại / tổng room | mã × ngày | ✅ |
| 2.3 | **Tự doanh CTCK** (khớp lệnh / thoả thuận / tổng) | mã × ngày | ⏸ |
| 2.4 | Giao dịch cổ đông lớn & nội bộ | mã × sự kiện | 🔴 |

## 3. Sự kiện doanh nghiệp (corporate actions)

| # | Dữ liệu | Độ mịn | Có |
|---|---|---|---|
| 3.1 | **Ngày GDKHQ (ex-date)** · ngày chốt danh sách | mã × sự kiện | 🟡 thiếu sự kiện |
| 3.2 | **Cổ tức tiền mặt** (VND/cp) | mã × sự kiện | 🟡 |
| 3.3 | **Cổ tức cổ phiếu / cổ phiếu thưởng** (tỷ lệ) | mã × sự kiện | 🟡 |
| 3.4 | **Chia tách / gộp cổ phiếu** (tỷ lệ) | mã × sự kiện | 🟡 |
| 3.5 | **Quyền mua (RIGHTS)** — tỷ lệ | mã × sự kiện | 🟡 |
| 3.6 | **Giá phát hành của RIGHTS** | mã × sự kiện | 🔴 không có chỗ lưu |
| 3.7 | Phát hành riêng lẻ / ESOP | mã × sự kiện | 🔴 |
| 3.8 | Niêm yết mới · huỷ niêm yết · đình chỉ · chuyển sàn | mã × sự kiện | 🔴 |
| 3.9 | ĐHCĐ (thường niên / bất thường) | mã × sự kiện | 🔴 |
| 3.10 | Ngày công bố BCTC | mã × kỳ | 🔴 |

**Nhóm 3 có dùng để kiểm tra giá hiệu chỉnh vs giá thô không? — CÓ, và đó là công dụng
quan trọng nhất của nó.** Nguyên lý: tỷ số `giá_đã_hiệu_chỉnh(d) / giá_thô(d)` là một
**hàm bậc thang** — hằng số giữa hai sự kiện, chỉ nhảy đúng tại ex-date, và độ cao mỗi bậc
bằng hệ số của sự kiện. Từ đó ra hai phép kiểm chạy được hằng ngày:

| Phép kiểm | Bắt được gì |
|---|---|
| Bậc nhảy **có** trong chuỗi giá nhưng **không** có sự kiện tương ứng trong bảng 3.x | **Thiếu sự kiện** trong bảng (hoặc sai ngày) |
| Hệ số **đo** từ giá ≠ hệ số **suy** từ tỷ lệ đã khai | Sai tỷ lệ, thiếu cấu phần (vd cổ tức tiền đi kèm cổ tức cổ phiếu) |
| Bậc nhảy tại chỗ **đổi nguồn dữ liệu** | Trộn giá thô với giá đã hiệu chỉnh trong cùng một chuỗi |

Đã chạy thật trên 163 sự kiện 2026 (gộp theo `(mã, ex_date)` vì nhiều sự kiện cùng ngày
thì hệ số là **tích**): **136 khớp · 5 lệch thật · 8 RIGHTS phải tách riêng** → 96,5% khớp.
Năm ca lệch: `GEX` `MCH` `BAF` `PET` `DSE`.

⇒ Bảng sự kiện **vừa là đầu vào để tính giá, vừa là thước kiểm chính giá đó**. Đây là lý do
nó phải nằm trên tuyến chính chứ không phải "analytics, optional".

## 4. Chỉ số & thị trường

| # | Dữ liệu | Độ mịn | Có |
|---|---|---|---|
| 4.1 | **OHLC chỉ số** (VNINDEX, VN30, HNX30, UPCOM…) | chỉ số × ngày | ✅ |
| 4.2 | **Độ rộng thị trường** — tăng/giảm/đứng giá/trần/sàn | chỉ số × ngày | 🔴 |
| 4.3 | KL & giá trị toàn thị trường | chỉ số × ngày | 🟡 |
| 4.5 | **Phái sinh VN30F1M / F2M** | HĐ × ngày | ✅ |
| 4.6 | Chứng quyền (CW) | mã × ngày | 🔴 |
| 4.7 | ETF nội (FUEVFVND, E1VFVN30…) | mã × ngày | 🟡 |
| 4.8 | **Biến động thực hiện (realized vol) của VNINDEX** | ngày | 🔴 dẫn xuất, chưa tính |

> **Bỏ:** 4.4 thành phần rổ chỉ số theo thời gian (quyết định 2026-07-30).
>
> **Đính chính "VIX" — tôi ghi sai ở bản trước.** Hệ thống **chưa bao giờ** lấy chỉ số
> biến động CBOE của Mỹ. `C:/Users/OS/Lucius/Projects/PRJ quant/01_fetch/fetch_vix.py` ghi rõ trong docstring: *"VIX Securities
> (HOSE: VIX)"* — tức **mã cổ phiếu VIX của CTCP Chứng khoán VIX**, cùng chỉ số ngành
> `VNFINLEAD`. Trùng tên viết tắt, khác hẳn bản chất. Không có chỉ số quốc tế nào trong
> hệ thống (mục "4.9 chỉ số quốc tế ✅" ở bản trước cũng sai).
>
> **Việt Nam không có chỉ số biến động ngụ ý** vì chưa có thị trường quyền chọn thanh
> khoản — không nguồn nào cấp được. Thay thế đúng là **realized vol tự tính từ VNINDEX**
> (4.8): miễn phí, không phụ thuộc nguồn ngoài, đo đúng thị trường mình giao dịch.
> VIX Mỹ chỉ là proxy risk-off toàn cầu, tác động lên VN gián tiếp qua dòng vốn ngoại —
> nếu muốn giữ thì để ở nhóm 6 như biến liên thị trường, không phải thước biến động của VN.

## 5. Cơ bản (fundamentals)

| # | Dữ liệu | Độ mịn | Có |
|---|---|---|---|
| 5.1 | **BCTC quý** — CĐKT / KQKD / LCTT / thuyết minh | mã × kỳ × chỉ tiêu | ✅ từ 2018 |
| 5.2 | BCTC năm (đã kiểm toán) | mã × năm | 🟡 |
| 5.3 | **Chỉ số tài chính** — PE, PB, PS, ROE, EV/EBITDA… | mã × kỳ | ✅ |
| 5.4 | **Số cổ phiếu lưu hành** | mã × ngày | 🟡 chưa vào chain |
| 5.5 | **Free float** | mã | 🔴 chưa lấy — **nguồn CÓ**: `VCI IQ /v1/company.freeFloat` + `freeFloatPercentage` |
| 5.6 | **Vốn hoá thị trường** | mã × ngày | 🟡 |
| 5.7 | Hồ sơ công ty · ban lãnh đạo · cổ đông lớn · công ty con | mã | 🟡 |
| 5.8 | Kế hoạch kinh doanh / nghị quyết HĐQT | mã × sự kiện | 🔴 |

## 6. Vĩ mô & liên thị trường

| # | Dữ liệu | Độ mịn | Có |
|---|---|---|---|
| 6.1 | **USD/VND** (mua/bán/trung tâm) | ngày | 🟡 hai bảng trùng |
| 6.2 | Tỷ giá khác (EUR, JPY, CNY…) | ngày | ✅ |
| 6.3 | **Lãi suất điều hành SBV** (tái chiết khấu, tái cấp vốn) | ngày/sự kiện | 🔴 0 dòng — **có sẵn trên trang đang fetch, chưa đọc** |
| 6.4 | **Lãi suất liên ngân hàng qua đêm (ON)** | ngày | 🟡 148 dòng → **cào được 3.854 dòng từ 2011** |
| 6.4b | Lãi suất liên NH kỳ hạn 1W / 2W / 1M / 3M / 6M / 9M | ngày | 🟡 nt (6M/9M thiếu 1.341 dòng ở nguồn) |
| 6.4c | Doanh số liên ngân hàng theo kỳ hạn | ngày | 🔴 5 dòng — **chỉ SBV có, trolyluat không có** |
| 6.5 | Lợi suất trái phiếu chính phủ | ngày | 🔴 |
| 6.6 | CPI · GDP · FDI · cung tiền · XNK | tháng/quý | 🟡 WorldBank, cũ |
| 6.7 | **Hàng hoá** — vàng, dầu, bạc, thép, cao su | ngày | ✅ |

## 7. Tham chiếu / master data

| # | Dữ liệu | Độ mịn | Có |
|---|---|---|---|
| 7.1 | **Danh mục mã** — tên, sàn, loại (cổ phiếu/ETF/CW/phái sinh) | mã | ✅ 3.330 |
| 7.2 | **Ngành** (ICB / GICS / phân ngành nội) | mã | ✅ |
| 7.3 | **Ngày niêm yết / ngày giao dịch đầu tiên** | mã | 🔴 chưa lấy — **nguồn CÓ**: `VCI IQ /v1/company.listingDate` |
| 7.4 | **Bảng bước giá · lô giao dịch** | mã | 🔴 |
| 7.5 | ISIN / mã định danh | mã | 🔴 |
| 7.6 | **Lịch giao dịch · nghỉ lễ** | ngày | 🟡 hết hạn 2026-07-03 |
| 7.7 | Universe theo dõi (khai báo) | mã | ✅ 261 active → xem §9 |
| 7.8 | **Hệ sinh thái / nhóm sở hữu** (Masan → MSN, MCH, MSR, MML…) | mã × nhóm | 🔴 **không nguồn nào có** — bảng khai báo tay |

**7.8 — vì sao cần:** mã cùng một hệ sinh thái biến động tương quan cao (tin về công ty
mẹ lan sang toàn nhóm), nên đây vừa là feature (return của nhóm), vừa là **ràng buộc rủi
ro danh mục** — mua 4 mã Masan không phải 4 vị thế độc lập mà gần như một. Không API nào
cung cấp; phải là bảng khai báo tay, có nguồn dẫn (báo cáo thường niên / cơ cấu sở hữu).

## 8. Văn bản & phi cấu trúc

| # | Dữ liệu | Độ mịn | Có |
|---|---|---|---|
| 8.1 | **Báo cáo phân tích CTCK** (rating, target price, luận điểm) | báo cáo | ✅ 9.456 |
| 8.2 | Tin tức / công bố thông tin | tin | 🔴 |
| 8.3 | Bản cáo bạch · nghị quyết · biên bản ĐHCĐ | tài liệu | 🔴 |

---

## 9. Danh sách cổ phiếu nạp vào DB

Nguồn: `C:/Users/OS/Lucius/Projects/PRJ quant/06_document/universe_261.xlsx` (bản 2026-07-30) + rổ VN100 từ SSI `IndexComponents`.

**Luật chọn:** mã **không bị tô đỏ** trong Excel (đỏ = vốn hoá nhỏ / ít giao dịch)
∪ toàn bộ **VN100** ∪ mã thuộc **hệ sinh thái** đã khai (7.8).

| | Số mã | Ghi chú |
|---|---|---|
| Trong Excel (sheet `261 mã active`) | 261 | |
| — bị tô đỏ (`accent2` = `C0504D`) → **loại** | **81** | |
| — không tô → **giữ** | **180** | |
| + VN100 tô đỏ → **giữ lại** (anh chốt 2026-07-30) | **+7** | `BWE` `HT1` `IMP` `KOS` `SJS` `SZC` `VPI` |
| + hệ sinh thái, đang tô đỏ → **giữ lại** | **+7** | `PGB` `PGS` `PGV` `PLC` `PPC` `VC3` `VTZ` |
| + hệ sinh thái, **không có trong file 261** → thêm mới | **+10** | `CAV` `GEG` `HNG` `NET` `NRC` `PVB` `SGT` `VCR` `VSN` `VTK` |
| **TỔNG NẠP** | **204** | |
| Sheet `Đã tắt` (huỷ niêm yết / chết) — không nạp | 5 | `AMD` `AVF` `BBC` `DAG` `HAI` |

VN100 nằm **trọn** trong 261, không thiếu mã nào.

**67 mã còn bị loại** (tô đỏ, không thuộc VN100, không thuộc hệ sinh thái nào đã khai):
`AAA ACG ADG ADS AGF AGG AGM AMS ANT APH API APS ASG ASM AST C4G C69 CST CVT DBD DCG DCL
FIR GSP HAD HAP HAR HAX HBC HCC HUT IJC KHG L40 LBM LHG LIG LIX LSS LUT MAC MAS MST NTP
PHH PLP PMC PPH SJD SKG SKH SLS SMC SPM SRA SRF TCM TCW TIG TVN UDC UIC UNI VCP VVS WSB WSS`

### 9.1 Bảng hệ sinh thái (mục 7.8) — **đã chốt 2026-08-03**

⚠️ **Hệ sinh thái ≠ ngành.** Nguồn trên mạng hay lẫn hai thứ. Ngành đã có ở 7.2 (ICB).
Giá trị của 7.8 là **quan hệ sở hữu / kiểm soát** — thứ làm giá các mã chạy cùng nhau.
Ví dụ hay bị nhầm: `PLX` (Petrolimex) **không** thuộc PVN · `POW`/`NT2` là **PV Power
(PVN)**, không phải EVN · `VEF` **không** thuộc Vingroup (chốt 2026-08-03).

| Hệ sinh thái | Mã | Độ chắc |
|---|---|---|
| **Vingroup** | `VIC` `VHM` `VRE` `VPL` | cao |
| **Masan – Techcombank** | `MSN` `MCH` `MML` `MSR` `TCB` `VCF` `VSN` `NET` | cao |
| **Sovico** | `VJC` `HDB` | cao |
| **Viettel** | `VGI` `VTP` `CTR` `VTK` `VTZ` | cao |
| **Gelex** | `GEX` `VGC` `GEE` `GEL` `CAV` `VSC` `EIB` `VIX` | trung bình |
| **PVN (dầu khí)** | `GAS` `PVS` `PVD` `PVT` `PVC` `BSR` `OIL` `POW` `NT2` `DPM` `DCM` `PVB` | trung bình |
| **EVN (phát điện)** | `PGV` `PPC` `QTP` `HND` | trung bình |
| **FPT** | `FPT` `FRT` `FOX` `FTS` | cao |
| **Petrolimex** | `PLX` `PGB` `PLC` `PGS` | trung bình |
| **HAGL** | `HAG` `HNG` | trung bình |

**Ba sửa ngày 2026-08-03:** bỏ `VEF` khỏi Vingroup · thêm `VSC` `EIB` `VIX` vào Gelex ·
**bỏ 5 nhóm "độ chắc thấp"**.

> 🕐 **Chờ xác nhận sau** — 5 nhóm đã bỏ, giữ lại đây để không phải tra lại từ đầu:
> **SHB – T&T** `SHB` `SHS` `BVS` · **Vinaconex** `VCG` `VC3` `VCR` ·
> **Novaland** `NVL` `NRC` · **TTC** `SBT` `GEG` `SCR` · **Kinh Bắc** `KBC` `SGT`

> ⚠️ **Ba mã trong bảng nay KHÔNG còn trong universe** (bỏ ngày 2026-08-03 theo §9.2):
> `VCF` `NET` (Masan) · `PGS` (Petrolimex). Quan hệ sở hữu vẫn đúng, nhưng **không có
> dữ liệu giá** cho chúng.

### 9.2 Mã đã BỎ khỏi universe

| Ngày | Mã | Lý do |
|---|---|---|
| 2026-08-03 | `GTA` `BVB` `PGS` `TVT` `NET` `VCF` `PRE` | thiếu ngày trong cửa sổ 30 ngày **và** không thuộc VN100 (QĐ-3). Universe 203 → 196 |
| trước đó | `CAV` | VCI khai `listing=False` — đã huỷ niêm yết |

Không xoá khỏi `meta.universe`, chỉ đặt `active=false` — giữ vết để tra lại được.
Dữ liệu giá cũ của chúng vẫn nằm trong `obs`, chỉ là job hằng ngày không hỏi nữa.

Nguồn: [Báo Đầu tư — Những hệ sinh thái hàng đầu trên sàn chứng khoán Việt Nam](https://baodautu.vn/nhung-he-sinh-thai-hang-dau-tren-san-chung-khoan-viet-nam-d366623.html)
· [Tin nhanh Chứng khoán](https://www.tinnhanhchungkhoan.vn/nhung-he-sinh-thai-hang-dau-tren-san-chung-khoan-viet-nam-post375393.html)
· [Doanh nghiệp & Tiếp thị — 6 "họ" cổ phiếu](https://doanhnghieptiepthi.vn/6-ho-co-phieu-chiem-gan-31-gia-tri-von-hoa-toan-thi-truong-161260210211034321.htm)

**Sáu nhóm "độ chắc thấp" là suy đoán từ tên/tiền sử, chưa đối chiếu cơ cấu sở hữu** —
đừng đưa vào tính toán trước khi xác nhận. Cách xác nhận không tốn công: VCI IQ
`/v1/company/{mã}/shareholder` cho danh sách cổ đông lớn ⇒ dò quan hệ sở hữu chéo bằng
dữ liệu thay vì bằng trí nhớ.

---

## Tổng

| Nhóm | Mục | ✅ | 🟡 | 🔴 | ⏸ |
|---|---|---|---|---|---|
| 1. Giá & giao dịch | 7 | 2 | 1 | 4 | 0 |
| 2. Dòng tiền | 4 | 2 | 0 | 1 | 1 |
| 3. Sự kiện DN | 10 | 0 | 5 | 5 | 0 |
| 4. Chỉ số | 7 | 3 | 2 | 2 | 0 |
| 5. Cơ bản | 8 | 2 | 5 | 1 | 0 |
| 6. Vĩ mô | 9 | 2 | 4 | 3 | 0 |
| 7. Tham chiếu | 8 | 3 | 1 | 4 | 0 |
| 8. Văn bản | 3 | 1 | 0 | 2 | 0 |
| **Tổng** | **56** | **15** | **18** | **22** | **1** |

*(Đã bỏ 1.4 · 1.5 · 4.4; hoãn 1.10–1.12 sang thiết kế streaming riêng.)*

---

## Bảo trì

- Thêm loại dữ liệu mới → thêm dòng ở đây **trước** khi viết script fetch.
- Cột **Có** cập nhật khi trạng thái đổi; không xoá dòng đã thôi dùng, đổi sang 🔴 kèm lý do.
- File này **không** chứa endpoint, tham số, schema — xem tài liệu (2) và (3).
