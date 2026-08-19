# PRJ-Quangtitative

Sổ tài liệu của dự án **PRJ Quantitative Investment**: trạng thái, bàn giao, quyết định,
kiến trúc, rule, review paper. Một nguồn duy nhất cho cả Claude chat (qua GitHub) và
Claude Code (qua ổ đĩa).

## Cấu trúc repo

```
PRJ-Quangtitative/
├── README.md              bản đồ repo — file này
├── STATE.md               (chưa có)  đang làm gì, còn treo gì         [ghi đè]
├── ARCHITECTURE.md        (chưa có)  bức tranh tổng thể hiện hành     [ghi đè]
├── RULES.md               (chưa có)  rule làm việc                    [ghi đè]
├── TAXONOMY.md            (chưa có)  giá trị hợp lệ: tags, verdict    [ghi đè]
├── chat-instructions.md   (chưa có)  rule đang dán ở claude.ai        [ghi đè]
│
├── handoffs/                         1 file/phiên  YYYY-MM-DD-slug.md [append]
├── decisions/             (rỗng)     1 file/quyết định  D####-slug.md [append]
│
├── data-design/           tài liệu kỹ thuật DB + nguồn dữ liệu        [ghi đè]
│   ├── README.md          bảng tra cứu của thư mục
│   ├── RUNBOOK.md         ★ bắt đầu từ đây — kho có gì, hỏi thế nào,
│   │                        chạy gì mỗi ngày, hỏng thì làm sao
│   ├── DB_DESIGN.md       thiết kế bảng: obs / market / meta / ops
│   ├── DATA_REQUIREMENTS.md   dữ liệu nào cần lấy — 56 mục, 8 nhóm
│   ├── DATA_SOURCE_MAP.md     map từng mục sang nguồn
│   ├── INGEST_SPEC.md     lấy cụ thể thế nào — D1…D17, §D12 = tải BCTC
│   ├── OHLCV.md           giá ngày: raw/adj, công thức, 18 phép kiểm
│   ├── BANG_TRA_MA_KIEM.md    mã C-01…C-18 là gì
│   ├── SOURCE_SSI.md · SOURCE_DNSE.md · SOURCE_VCI.md · SOURCE_VNSTOCK.md
│   └── diagram/*.svg      sơ đồ (chỉ SVG — PNG @2x để ở máy)
│
├── requirements/
│   └── URD.md             yêu cầu gốc — mâu thuẫn thì URD đúng
│
└── reviews/
    ├── INDEX.md           danh sách paper: id · tên · tags · verdict · năm
    ├── P0001-mau.md       mẫu để viết review mới
    └── P####-<slug>.md    1 file/paper                                [append]
```

Nằm ngoài repo, không có đường vào git:

```
<thư mục local>/
├── papers/    PDF gốc — file review trỏ tới bằng đường dẫn tuyệt đối
├── data/
└── .env
```

## Hai loại file

| Nhãn | Nghĩa |
|---|---|
| `[ghi đè]` | Trạng thái hiện hành, luôn một bản, sửa đè lên |
| `[append]` | Bản ghi có ngày, viết xong không sửa lại |

Trộn hai loại thì file vừa kể lịch sử vừa nói hiện trạng, sai cả hai.

## Không chứa

**Code và dữ liệu** — hai thứ đó ở máy local và ở repo project
`vn-quant/prj-quantitative-investment`. Cùng với chúng: API key, `.env`, PDF, file `.parquet`,
và ảnh PNG `@2x` của sơ đồ (chỉ SVG lên đây — SVG là văn bản, đọc được; PNG chỉ làm nặng repo).

Lý do chia như vậy: repo này tồn tại để **đọc được từ nơi không mở được repo project** —
Claude chat chỉ lấy được nội dung dạng văn bản, và git thì giữ mọi phiên bản vĩnh viễn nên
không hợp với file nhị phân.

## Bản nào là bản gốc

`data-design/` và `requirements/` là **bản làm việc** — sửa ở đây. Repo project giữ bản
archive cùng code và dữ liệu.

⚠️ Hiện hai repo **đang có cùng nội dung**. Chỉ nên còn một nơi được sửa, nếu không hai bên
sẽ lệch — đúng cái bệnh mà cấu trúc này sinh ra để tránh.

## Thứ tự đọc — Claude chat

1. `README.md` + `STATE.md` trước mọi việc.
2. Hỏi về paper → `reviews/INDEX.md`, lọc theo `tags`, chỉ lấy đúng các file `P####`
   liên quan. Không quét cả thư mục.
3. Paper không có trong INDEX → nói rõ "chưa review" trước khi suy luận.
4. Hỏi về một quyết định → tìm trong `decisions/`, trích tên file.
5. Nhận định mới mâu thuẫn với file đã lưu → nói rõ là đang mâu thuẫn, không lặng lẽ
   đưa kết luận khác.

`reviews/INDEX.md` là bản đồ, phải luôn đủ nhỏ để nằm trọn trong context.

## Quy ước

- `id` tăng dần, không tái sử dụng kể cả khi xoá.
- Tên file review `P####-<slug>.md` — dòng INDEX trỏ tới file không cần đoán.
- Một paper mang nhiều `tags`; không nhân bản dòng trong INDEX.
- Thêm giá trị `tags`/`verdict` mới → sửa `TAXONOMY.md` trước, dùng sau.
- Trong file review, tóm tắt (paper nói gì) tách hẳn khỏi lời bình (mình nghĩ gì).
- Commit theo Conventional Commits.

## Repo liên quan

`vn-quant/prj-quantitative-investment` (private) giữ dữ liệu, thiết kế, code. Hai repo
không đồng bộ tự động với nhau.

Chưa chốt: `handoffs/` đang tồn tại ở cả hai repo — cần chọn một chỗ.
