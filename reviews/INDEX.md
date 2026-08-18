# INDEX — danh sách paper

Bảng **định tuyến**, không phải bảng dữ liệu. Chỉ giữ trường dùng để quyết định
*mở file nào*. DOI, link, tác giả, venue nằm trong frontmatter của từng file review.

Giá trị hợp lệ của `tags` và `verdict`: xem `../TAXONOMY.md`.
Thêm giá trị mới là hành động có ý thức — sửa TAXONOMY trước, dùng sau.

| id | Tên ngắn | tags | verdict | Năm |
|---|---|---|---|---|
| P0001 | _(file mẫu — xoá dòng này khi có paper thật)_ | `mau` | `tham-khao` | — |

## Quy ước

- `id` tăng dần, không tái sử dụng kể cả khi xoá paper.
- Tên file review: `P####-<slug>.md` — dòng index phải trỏ tới file không cần đoán.
- Một paper có thể mang nhiều `tags`; **không** nhân bản dòng cho từng lĩnh vực.
- File này phải luôn đủ nhỏ để nằm trọn trong context. Dài quá thì cắt bớt cột,
  không cắt bớt dòng.
