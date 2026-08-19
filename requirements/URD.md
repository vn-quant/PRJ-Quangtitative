# URD — Yêu cầu người dùng, PRJ Quantitative Investment

> **File này là bản chốt yêu cầu.** Spec, thiết kế DB, code và phép kiểm đều phải quy về đây.
> Khi có mâu thuẫn giữa file này và bất kỳ file nào khác — **file này đúng**, file kia phải sửa.

| | |
|---|---|
| Phiên bản | 1.0 — 2026-08-02 |
| Phạm vi bản này | Tầng dữ liệu (OHLCV · sự kiện · universe · vận hành). Chưa phủ ML, Council, webapp |
| Người duyệt | chủ project — không ai khác được đổi một dòng nào trong §3 |
| Sửa file này thế nào | xem §7 |

---

## 1. Vì sao có file này

Yêu cầu về OHLCV không đổi từ 18/06/2026. Nhưng nó đã đi qua **4 lần audit + 1 lần
thiết kế lại** mà vẫn hỏng, vì yêu cầu chỉ nằm trong hội thoại — nói xong là mất.

| Ngày | Việc | Kết cục |
|---|---|---|
| 04/07 | audit lần 1 — *"kiểm tra db và cách lấy dữ liệu, audit lại toàn bộ"* | ra danh sách lỗi, không chặn được lỗi mới |
| 05/07 | audit lần 2 — *"audit lại cơ chế tải dữ liệu về đang lỗi ở mã hay có vấn đề gì"* | như trên |
| 18/07 | audit lần 3 — *"đã audit 2 lần rồi mà… thi thoảng lại gặp lỗi như thế này"* | phát hiện ohlcv nhiều source/dòng trùng |
| 25–27/07 | audit lần 4 — *"có mỗi ohlcv mà hỏng sau 4 lần audit vẫn không sửa được và tôi phải tự vào tìm nguyên nhân?"* | quyết định đập đi làm lại |
| 29/07–02/08 | thiết kế lại (ver 2) | vẫn phát sinh: trộn nguồn (C-12, 219 bậc giả), spec lệch code |

Câu chẩn đúng bệnh, 18/07 16:15:

> *"tại sao thiết kế hệ thống vẫn bị miss những cái tôi đề cập? **với khối lượng quá lớn tôi
> không check hết được**, do logic không biết hay do giả định và tự làm hay không hiểu hoàn cảnh hệ thống?"*

URD không sửa được trí nhớ của ai. Nó chỉ làm một việc: biến "điều tôi đã dặn" thành
**thứ máy kiểm được**, để không cần một người ngồi nhớ.

---

## 2. Quy ước

Mỗi yêu cầu có mã `U-<nhóm><số>` và **bắt buộc** có cột "nghiệm thu bằng gì".
Yêu cầu không nghiệm thu được bằng máy thì ghi rõ `người kiểm` — không được để trống.

| Ký hiệu | Nghĩa |
|---|---|
| ✅ | đạt, có phép kiểm tự động đang chạy |
| ⚠️ | đạt một phần — cột ghi rõ phần nào chưa |
| ❌ | chưa đạt |
| ❓ | chưa xác nhận — **không ai được ghi ✅ thay** |

Nhóm: `W` cách làm việc · `D` nguồn · `O` OHLCV · `E` sự kiện · `U` universe ·
`R` vận hành · `T` truy vết · `X` truy cập từ xa.

---

## 3. Yêu cầu

### 3.1 `U-W` — Cách làm việc (ràng buộc lên AI, không phải lên hệ thống)

| Mã | Yêu cầu | Đã nói lúc | Nghiệm thu bằng gì | TT |
|---|---|---|---|---|
| **U-W1** | Không viết thêm code, không chạy thêm nguồn, không thêm tính năng ngoài phạm vi câu lệnh | 05/07 *"tại sao lại vượt quyền đổi chuỗi, từ vnindex sang vn30?"* · 18/07 *"dừng, không sửa gì khi tôi chưa bảo"* · 01/08 *"đừng có vẽ vời thêm code khi chưa bảo"* | người kiểm: mọi file chạm trong 1 phiên phải nằm trong phạm vi đã giao | ❌ |
| **U-W2** | Không tự đổi nguồn / chuỗi / tham số đã chốt. Muốn đổi → nêu số đo, chờ duyệt | 05/07 vnindex→vn30 · 27/07 *"sao tự dưng lại chuyển VCI????? đang nói ssi mà"* | mỗi lần đổi nguồn = 1 dòng CHANGELOG + 1 mã `E####` | ❌ |
| **U-W3** | Làm hết yêu cầu rồi mới dừng; không dừng ở checkpoint để hỏi | 02/08 *"tiếp tục làm đến khi nào xong được requirement của urd, sao lại dừng giữa chừng?"* | người kiểm | ⚠️ |
| **U-W4** | Việc treo phải ghi thẳng ra, kèm **lý do** và **điều kiện gỡ** | 01/08 *"5 câu treo là gì? sao không ghi thẳng ra?"* | `3. Build\test_spec_vs_code.py` — hoãn mà thiếu lý do/điều kiện thì ĐỎ | ✅ |
| **U-W5** | Mọi mã viết tắt phải giải nghĩa tại chỗ. Không bắt nhớ, không bắt mở nhiều cửa sổ | 02/08 *"c-01 là cái j? giờ bắt tôi nhớ từng lỗi hay sao?"* + *"ghi giải thích vào, chứ không phải dẫn link"* | `2. Data design\main\BANG_TRA_MA_KIEM.md` | ✅ |
| **U-W6** | Dẫn file phải là đường dẫn **tuyệt đối đầy đủ**, kèm tên tài liệu nguồn | 02/08 *"đã bảo khi dẫn đường dẫn thì phải dẫn cả nguồn ra OHLCV_SPEC.md thì tra kiểu gì được????"* | người kiểm | ⚠️ |
| **U-W7** | Đo từ **NGUỒN** (gọi API), không đo từ **KHO** (query DB) | 29/07, mở phiên: *"9/10 lỗi của phiên trước do đo từ kho"* | sổ `E####` có cột "đo từ đâu" — API / KHO / CODE | ✅ |
| **U-W8** | Tài liệu viết **sau** khi đo xong, không viết trước rồi đo sau | 26/07 *"tại sao không đo hết rồi mới làm tài liệu? thế này thì sử dụng kiểu gì? mặc định nó đúng hay sai?"* | người kiểm | ⚠️ |

### 3.2 `U-D` — Nguồn dữ liệu

| Mã | Yêu cầu | Đã nói lúc | Nghiệm thu bằng gì | TT |
|---|---|---|---|---|
| **U-D1** | Gọi API trực tiếp SSI · DNSE · VCI. **Không dùng vnstock** trừ khi không có nguồn thay thế — vnstock là bản phái sinh tổng hợp trên mạng, không tin được | 26/07 *"cả ssi và dnse đều có api, tại sao lại cố lấy qua vnstock?"* · 29/07 17:26 | grep code: không import vnstock trong luồng GIÁ | ✅ với luồng giá. **Ngoại lệ chốt 03/08: BCTC được dùng vnstock** vì mọi nguồn gốc đều mất phí — xem §5 QĐ-1. Ngoại lệ này **chỉ** cho BCTC |
| **U-D2** | Mỗi trường chốt **một** nguồn cho **toàn chuỗi** mỗi mã. Cấm trộn nguồn giữa chừng | 02/08, sau khi C-12 bắt được 219 bậc giả (E0045) | `C-12` — nhảy >50%/phiên không có sự kiện | ⚠️ `market.px_adj` đã khoá; `market.px_adj_as_of()` **vẫn trộn** — xem §5 L-1 |
| **U-D3** | Trước khi kết luận về một nguồn, phải đo **hết mọi cửa** của nguồn đó | 02/08 *"có cần xem lại xem dnse và ssi có mấy cửa không? sao ngày xưa lại đo thiếu?"* | `2. Data design\main\DATA_SOURCE_MAP.md` liệt kê từng endpoint đã thử | ⚠️ |
| **U-D4** | Nguồn vừa đổi API thì đo bản mới, không dùng bản cũ | 26/07 *"SSI mới cập nhật lại API nên đừng dùng cái cũ… không sau này khi audit lại thành trót dùng nguồn cũ nên bị miss thông tin"* | mỗi `E####` ghi ngày đo | ✅ |
| **U-D5** | Nguồn chết / rate-limit phải có đường vòng đã đo trước, không xử lý lúc cháy | 06/07 rate-limit VCI 429 · 18/07 *"giữa việc dùng 3 nguồn cùng chạy và fall back khi 1 api die thì cách nào ổn định hơn?"* | `main\OHLCV.md` §6.1 (E0029) | ✅ |

### 3.3 `U-O` — OHLCV (chốt 02/08 12:45, nguyên văn 4 gạch đầu dòng)

| Mã | Yêu cầu | Nghiệm thu bằng gì | TT |
|---|---|---|---|
| **U-O1** | Đủ **giá raw** và **giá adj** để backtest và kiểm tra lại sau này | `market.px_raw` + `market.px_adj` cùng phủ 203 mã | ⚠️ raw đủ 577.607 dòng/203 mã từ 2000-07-28 (E0063), **nhưng O/H/L raw chỉ tin được ở HOSE** — HNX 14/26, UPCoM 0/29 |
| **U-O2** | Khi một event xảy ra thì **chỉ thay adj**, raw giữ nguyên. Có bảng event | `C-11` — adj ≡ raw ở vùng sau ex-date gần nhất | ✅ 26.520/26.520 khớp |
| **U-O3** | adj hỏi được theo mặt bằng **một ngày bất kỳ** (as-of), không chỉ mặt bằng hôm nay | `market.px_adj_as_of(ngày)` | ⚠️ có hàm, nhưng hàm còn lỗi trộn nguồn (L-1) |
| **U-O4** | Một chuỗi giá = một nguồn. Áp cho **mọi** đường đọc, kể cả as-of | `C-12` | ❌ xem L-1 |
| **U-O5** | Độ sâu lấy tới khi nguồn còn có, không tự cắt tại 2012 hay 2020 | đếm ngày sớm nhất mỗi mã | ✅ tới 2000-07-28 |
| **U-O6** | Biết được ngày nào **đáng lẽ** phải có phiên, để phát hiện thủng | `meta.trading_calendar` + `C-09`, `C-10` | ⚠️ log 02/08 còn `n_mã_thiếu_ngày = 7` mà cổng vẫn OK |
| **U-O7** | O/H/L phải là số **ĐO**. Số **SUY** ra thì phải đánh dấu, không được ghi lẫn vào sổ quan sát | `C-01` + view lọc theo sàn | ⚠️ đã lọc khi đọc, nhưng số suy vẫn nằm trong `obs.price_daily` (E0063) |
| **U-O8** | Hệ số điều chỉnh phải đối chiếu được với **giá tham chiếu sàn công bố**, không tự tính rồi tin | `C-15` qua `market.factor_exchange` | ❌ còn **406 ca lệch** (E0065) |

### 3.4 `U-E` — Sự kiện doanh nghiệp

| Mã | Yêu cầu | Nghiệm thu | TT |
|---|---|---|---|
| **U-E1** | Bảng sự kiện cập nhật **hằng ngày** và chạy lại không sinh dòng mới | chạy `daily_events.py` 2 lần: chèn 88 → 0 | ✅ (E0066) |
| **U-E2** | Hai nguồn đối chiếu, so **tập với tập**, cấm lấy phần tử đầu | `C-17` | ✅ |
| **U-E3** | Đủ ngày GDKHQ · cổ tức tiền · cổ tức cổ phiếu · quyền mua + giá phát hành | `main\OHLCV.md` §2.2 | ✅ DNSE senses 95/100 |

### 3.5 `U-U` — Universe

| Mã | Yêu cầu | Đã nói lúc | TT |
|---|---|---|---|
| **U-U1** | Nạp các mã **không tô đỏ** trong `universe_261.xlsx`, cộng VN100, cộng theo họ sinh thái | 29/07 17:51 | ✅ 203 mã |
| **U-U2** | VN100 **point-in-time** — không dùng danh sách hôm nay cho dữ liệu lịch sử | 01/08 08:30 | ❓ chưa xác nhận |
| **U-U3** | Có trường **họ sinh thái** (MASAN → MSR · MCH · MML) | 29/07 17:51 | ❌ `main\DATA_REQUIREMENTS.md` §9.1 còn là bản nháp chờ duyệt |

### 3.6 `U-R` — Vận hành hằng ngày

| Mã | Yêu cầu | Đã nói lúc | TT |
|---|---|---|---|
| **U-R1** | Chạy **tự động hằng ngày**, vào cuối ngày | 11/07 *"nếu lấy theo ngày thì bắt buộc phải chạy vào cuối ngày"* · 25/07 *"dữ liệu lịch sử giá cần chạy hằng ngày, chạy hàng tuần thì có ích gì?"* | ❌ v2 mới chạy **tay 1 lần** (02/08 22:00). Task Scheduler chỉ có `\PRJQuant_DailyUpdate` của **ver 1** |
| **U-R2** | Một file log tải về hằng ngày, dựa vào đó truy vết được | 02/08 12:45 | ✅ `logs\INDEX.csv` + `logs\<ngày>_daily_update.csv` |
| **U-R3** | Ngày bị ngắt quãng (mất phiên, không mở máy) phải phát hiện và bù | 02/08 12:45 | ⚠️ phát hiện được; bù tự động chưa có |
| **U-R4** | Nguồn chia lại lịch sử thì hệ tự phát hiện và nạp lại, không chờ người | 19/07 *"1 năm giá cổ phiếu được trả cổ tức chỉ đổi 1-2 lần, tại sao lại mặc định chạy 11 phút mỗi ngày?"* | ✅ E0062 — so chính dữ liệu, không dựa bảng sự kiện |

### 3.7 `U-T` — Truy vết & bằng chứng

| Mã | Yêu cầu | Đã nói lúc | TT |
|---|---|---|---|
| **U-T1** | Mọi số đi vào tài liệu phải có mã `E####` tra lại được — *"khi báo cáo, nếu có nghi ngờ có thể check lại luôn thay vì làm lại từ đầu"* | 28/07 14:27 | ✅ 66 mã E |
| **U-T2** | `E####` phải trỏ tới **commit chứa code đã đo** | hệ quả của U-T1 | ⚠️ ver 2 đã có git nên từ giờ trỏ được; E0061→E0066 vẫn ghi `0417bf5` (commit ver 1) — chưa sửa lại. **Chỉ giữ được nếu mỗi cổng nghiệm thu là một commit** |
| **U-T3** | Spec và code lệch nhau thì phải **ĐỎ được**, không dựa trí nhớ | 02/08 13:17 *"đây là yêu cầu từ đầu mà nhắc lại xong vẫn làm không đúng"* | ⚠️ `test_spec_vs_code.py` có, nhưng bảng đăng ký khai tay và chỉ kiểm "file có nhắc mã" |

### 3.8 `U-X` — Truy cập từ xa

| Mã | Yêu cầu | Đã nói lúc | TT |
|---|---|---|---|
| **U-X1** | Xem được **thiết kế** từ máy công ty / tablet | 18/06 13:41 *"có thể truy cập webapp github hoặc dùng tablet"* · 02/08 | ✅ repo private `vn-quant/prj-quantitative-investment` |
| **U-X2** | **Không** đẩy lên: API key · credential · mật khẩu DB · dữ liệu thị trường · file Excel danh mục | 02/08 | ✅ đã gỡ mật khẩu khỏi `Blueprint.md` và `Blue print data.txt`; `.gitignore` chặn phần còn lại, quét lại remote không còn |
| **U-X3** | **GitHub là bản ĐỌC, máy là nguồn CHẠY.** Đồng bộ một chiều máy → GitHub. Không sửa trực tiếp trên GitHub | 03/08 *"nguồn github chỉ là nguồn đọc tham khảo, không phải nguồn chạy chính trên máy"* | ⚠️ đang đúng theo cách làm, nhưng **không có gì cưỡng chế** — xem QĐ-5 |

---

## 4. Ma trận truy vết

Đọc theo chiều: yêu cầu → tài liệu nào tả → code nào làm → phép kiểm nào canh.

| Yêu cầu | Đặc tả | Code | Phép kiểm |
|---|---|---|---|
| U-O1 · U-O5 | `main\OHLCV.md` §1, §3.1 | `load_ssi_sp.py` · `load_iboard.py` | C-01 · C-10 |
| U-O2 · U-O3 | `main\OHLCV.md` §5 | `schema\060_market_views.sql` | C-11 |
| U-O4 · U-D2 | `main\OHLCV.md` §2.1 | `schema\060_market_views.sql` | C-12 |
| U-O8 | `main\OHLCV.md` §4.3 | `check_c11_c15.py` | C-15 |
| U-E1 · U-E2 | `main\OHLCV.md` §2.2 | `daily_events.py` · `schema\090_ca_dedupe.sql` | C-04 · C-17 |
| U-R1 · U-R2 · U-R3 | `main\OHLCV.md` §6 | `daily_update.py` · `schema\080_fetch_log.sql` | C-08 · C-09 · C-16 |
| U-R4 | `main\OHLCV.md` §6 | `daily_update.py` | `test_nap_lai_khi_co_event.py` |
| U-W4 · U-T3 | file này | `test_spec_vs_code.py` | chính nó |

Tất cả đường dẫn tính từ `C:\Users\OS\Lucius\Projects\PRJ Quantitative Investment\`,
code trong `3. Build\`, đặc tả trong `2. Data design\main\`.

---

## 5. Đang chặn — cần quyết hoặc cần sửa

| Mã | Loại | Nội dung |
|---|---|---|
| ~~**L-1**~~ | ✅ **XONG 03/08** | `market.px_adj_as_of()` từng chọn nguồn **theo từng dòng** — nay chốt nguồn theo mã, giống `px_adj`. Đồng thời sửa quy tắc chọn nguồn: ưu tiên iBoard trừ khi nó thủng >10% (quy tắc "nguồn nào nhiều dòng hơn" chọn nhầm 10 mã, gồm SHP). Thêm `code\test_view_va_ham_khop.py` cưỡng chế view và hàm phải cho cùng kết quả — vì đây là lần **thứ hai** sửa view mà quên hàm |
| **L-2** | lỗi | Mật khẩu `quant_db` nằm nguyên văn ở `1. Blue print\Blueprint.md:59` và `Blue print data.txt:29`. `archive\DB_BUILD_RUNBOOK.md:92` lại khai *"grep toàn repo → 0 kết quả"* — khai sai. Vi phạm U-X2 |
| **L-3** | 🕐 **HOÃN CÓ CHỦ Ý 03/08** | Hệ số điều chỉnh còn 432/2.445 ex-date lệch, và 564/3.448 ex-date chưa kiểm chứng được. **Chốt: bỏ qua bảng sự kiện, khi backtest sẽ tự xử lý.** Vướng mắc ghi đầy đủ ở `issues\TASK-006.md`. U-O8 đứng, KHÔNG chặn tầng giá |
| ~~**QĐ-1**~~ | ✅ **CHỐT 03/08** | **BCTC dùng vnstock.** Ba nguồn gốc có BCTC (Fiingroup · Wifeed · Vietstock) đều **mất phí**; SSI/DNSE không có BCTC. Đây là **ngoại lệ có ý thức của U-D1**, chỉ áp cho BCTC, không mở cho dữ liệu giá. Điều kiện kèm theo: mọi bảng BCTC phải mang cờ `nguon_phai_sinh = true` và ghi `source = 'VNSTOCK'` để mọi thứ đọc nó đều biết đây là số tổng hợp lại, không phải số đo từ nguồn gốc |
| ~~**QĐ-2**~~ | ✅ **CHỐT 03/08** | bỏ `VEF` khỏi Vingroup · thêm `VSC` `EIB` `VIX` vào Gelex · bỏ 5 nhóm độ chắc thấp (giữ trong ghi chú §9.1 để xác nhận sau) |
| ~~**QĐ-3**~~ | ✅ **CHỐT 03/08** | KHÔNG đặt ngưỡng. Bỏ 7 mã `GTA` `BVB` `PGS` `TVT` `NET` `VCF` `PRE` khỏi universe vì không thuộc VN100. Universe 203 → 196, `n_mã_thiếu_ngày` = **0** |
| **QĐ-4** | đã trả lời, **chưa làm** | `meta.universe` chỉ có ảnh chụp hiện tại (cột `active`), KHÔNG có lịch sử thành phần rổ ⇒ backtest sẽ dính survivorship bias. Không API nào cho point-in-time; HOSE công bố 2 lần/năm (T1 và T7), Vietstock đưa tin mỗi kỳ ⇒ dựng lại được bằng cách gom ~25–50 lần đảo rổ |
| **QĐ-5** | hoãn có chủ ý | **Làm cho máy và GitHub đồng nhất.** Hiện tại một chiều máy → GitHub, và chỉ mới bằng lần commit gần nhất. Chưa có gì chặn việc sửa nhầm trên GitHub, cũng chưa có gì báo khi hai bên lệch. Chốt 03/08: **để sau, thiết kế lại cả hai nguồn cùng lúc.** Tới lúc đó U-T2 chỉ giữ được bằng kỷ luật commit, không bằng cơ chế |

---

## 6. Yêu cầu đã đúng ngay từ đầu nhưng vẫn bị làm sai

Giữ lại để đối chiếu, không phải để trách. Mỗi dòng là một lần một yêu cầu đã nói
rồi vẫn bị vi phạm — đó là lý do §3 tồn tại.

| Ngày | Yêu cầu bị vi phạm | Biểu hiện |
|---|---|---|
| 05/07 | U-W2 | tự đổi chuỗi từ VNINDEX sang VN30 |
| 18/07 | U-W1 | sửa code khi chưa được bảo |
| 19/07 | U-D1 | dựng lại luồng không theo hướng dẫn đã có ở trên trong cùng phiên |
| 26/07 | U-D1 | vẫn lấy qua vnstock khi SSI/DNSE đều có API |
| 26/07 | U-W8 | viết tài liệu nguồn trước khi đo xong |
| 27/07 | U-W2 | đang nói SSI, tự chuyển sang VCI |
| 01/08 | U-W1 · U-W2 | chạy SSI intraday khi nguồn đã chốt là DNSE |
| 02/08 | U-W6 | dẫn đường dẫn không kèm tài liệu nguồn |
| 02/08 | U-D2 | `px_adj` trộn hai nguồn trong một chuỗi → 219 bậc giả |

---

## 7. Sửa URD thế nào

1. Không sửa §3 trong lúc đang code. Muốn đổi yêu cầu → dừng, sửa file này trước, ghi ngày.
2. Mỗi lần sửa: tăng số phiên bản ở đầu file, thêm một dòng vào bảng dưới.
3. Thêm một yêu cầu mới thì **bắt buộc** điền cột "nghiệm thu bằng gì". Không nghiệm thu
   được thì ghi `người kiểm` — nhưng phải nói ra, không để trống.
4. Đổi trạng thái từ ❓ hoặc ❌ sang ✅ thì phải kèm mã `E####` hoặc tên phép kiểm.

| Phiên bản | Ngày | Đổi gì |
|---|---|---|
| 1.0 | 2026-08-02 | Bản đầu. Rút từ hội thoại 18/06 → 02/08, 1.460 lượt trao đổi qua 42 phiên |
