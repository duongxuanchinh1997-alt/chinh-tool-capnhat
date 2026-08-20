# Bóc tách & Thống kê khối lượng MEP

## Mô tả

Skill hướng dẫn AI bóc tách khối lượng ống, phụ kiện, giá treo từ bản vẽ AutoCAD của CHÍNH TOOL. Dùng khi người dùng nói: "bóc tách khối lượng", "thống kê ống", "đếm phụ kiện", "tính mét ống D110", "ra bảng khối lượng", "xuất Excel khối lượng", "tổng hợp vật tư bản vẽ".

> **File này phải lưu ở UTF-8.** Bản cũ lưu ở CP1252 nên toàn bộ dấu tiếng Việt đã mất thành `?`.

## Hai đường bóc tách — chọn đúng đường

| Tình huống | Dùng | Vì sao |
|---|---|---|
| Ống do **HTSP/HTSPU** vẽ ra (có dữ liệu gắn sẵn) | **Lệnh `HTSKL`** | Đọc thẳng hệ + đường kính + cao độ từ XDATA, đếm phụ kiện theo tên block. Chính xác tuyệt đối, 1 lệnh duy nhất, không tốn tool call. |
| Bản vẽ **của người khác**, ống vẽ tay, không có dữ liệu gắn | `select_and_export` → `open_query_tools` | Phải suy đoán từ MLINE scale / layer / tên block. |

**Ưu tiên `HTSKL` trước.** Chỉ rơi về `select_and_export` khi `HTSKL` báo hệ ống là `?` (nghĩa là ống không mang dữ liệu).

## Đường 1 — lệnh HTSKL (khuyến nghị)

```json
{ "name": "run_command", "arguments": { "command": "HTSKL" } }
```

Người dùng quét chọn vùng, hoặc Enter để lấy toàn bộ Model. Kết quả tự đẩy lên bảng `htskl` trên giao diện và in ra dòng lệnh.

Lệnh đi kèm — chỉ gọi khi người dùng yêu cầu:

| Lệnh | Việc |
|---|---|
| `HTSKLB` | Vẽ bảng khối lượng vào bản vẽ (hỏi điểm đặt) |
| `HTSKLCSV` | Xuất `<tên bản vẽ>-KHOILUONG.csv` cạnh bản vẽ |
| `HTSKLZ` | Chọn một dòng → tô chọn đúng các đối tượng đó trên bản vẽ |
| `HTSKLCFG` | Đặt % hao hụt, tách nhóm theo layer, cỡ chữ bảng |

### Hai con số chiều dài — phải phân biệt khi báo cáo

- **Dài cắt** — chiều dài ống thật sự phải cắt, đã trừ phần nằm trong phụ kiện. Đây là số dùng để **mua vật tư**.
- **Dài tim** — chiều dài tim tuyến kể cả đoạn phụ kiện chiếm chỗ. Đây là số dùng để **đối chiếu với bản vẽ thiết kế**.

Chênh lệch không nhỏ: một cút 90 D110 uPVC ăn 119 mm mỗi phía, tê D110 ăn 110 mm mỗi phía; tuyến thoát nước nhiều nút có thể lệch 5–8%. Khi báo cáo luôn nêu rõ đang dùng số nào.

## Đường 2 — select_and_export (bản vẽ ngoài)

Tối đa **2 tool call**:

1. `select_and_export` — quét chọn + lưu JSON
2. `open_query_tools` — mở cửa sổ thống kê

Không gọi `read_exported_data` hay `filter_exported_data` khi đã mở `open_query_tools`. Chỉ dùng `filter_exported_data` khi người dùng chỉ cần AI trả lời bằng text.

`entityType` nhận nhiều loại cách nhau bằng dấu phẩy để quét một lần:

| Loại thống kê | entityType | Nhóm theo |
|---|---|---|
| Đường ống nước | `MLINE` | `styleName` |
| Phụ kiện, thiết bị | `INSERT` | `blockName` |
| Ống điện / dây | `LWPOLYLINE` | `layer` |
| Ống + phụ kiện | `MLINE,INSERT` | tách nhóm theo type |
| Toàn bộ MEP | `MLINE,INSERT,LWPOLYLINE` | tách nhóm theo type |

Tuyệt đối không đưa TEXT, MTEXT, DIMENSION, HATCH, LEADER vào kết quả — lọc ngay bằng `entityType`.

```json
{ "name": "select_and_export", "arguments": { "message": "Hay quet chon vung can boc tach", "entityType": "MLINE,INSERT" } }
{ "name": "open_query_tools", "arguments": {} }
```

Chiều dài trong JSON là **mm**, chia 1000 khi báo cáo.

## Quy tắc đọc tên block

Tên block trong thư viện HTS theo mẫu `HTS-<LOẠI>-D<cỡ>[x<cỡ nhánh>][ <hệ>]`. Suy ra tên gọi như sau:

| Mẫu | Tên gọi |
|---|---|
| `HTS-ELB90-D110` | Cút 90 uPVC D110 |
| `HTS-ELB45-D110` | Chếch 45 uPVC D110 |
| `HTS-TEE-D110x110` | Tê đều uPVC D110 |
| `HTS-TEE-D110x90` | Tê thu uPVC D110x90 |
| `HTS-Y-D110x90` | Chạc Y thu uPVC D110x90 |
| `HTS-SA-D110x90` | Tê cong uPVC D110x90 |
| `HTS-RED-D110x90` | Côn thu uPVC D110x90 |
| `HTS-ELB90-D63 PPR` | Cút 90 **PPR** D63 |
| `HTS-TEE-D63x63 HDPE` | Tê đều **HDPE** D63 |
| `HTS-QUANGTREO-D110` | Quang treo D110 |
| `HTS-GIADO-DN100` | Giá đỡ DN100 |
| `HTS-VanuPVC-D110` | Van uPVC D110 |
| `HTS-MANGSONGUPVC-D110` | Măng sông uPVC D110 |
| `HTS-Nutbit-D110` | Nút bịt uPVC D110 |
| `HTS-Siphong-D90` | Si phông uPVC D90 |
| `HTS-2Chech-D110` | Bộ hai chếch uPVC D110 |
| `HTS-YCHECH-D110x90` | Y + chếch uPVC D110x90 |
| `HTS-CLAMP SADDLE-50x25 HDPE` | Yên gá HDPE 50x25 |

Hệ ống lấy từ **hậu tố tên file** trước, rồi mới tới thư mục — thư mục `Combo` trộn cả uPVC lẫn PPR.

## Trình bày kết quả

Tách bảng theo nhóm, đừng gộp một bảng dài:

**A. Đường ống**

| Tên gọi | Hệ | Số đoạn | Dài cắt (m) | Dài tim (m) |
|---|---|---|---|---|
| Ống uPVC D110 | uPVC | 15 | 45.20 | 48.66 |

**B. Phụ kiện**

| Mã hiệu | Tên gọi | SL |
|---|---|---|
| HTS-ELB90-D110 | Cút 90 uPVC D110 | 12 |

**C. Giá treo** — tách riêng, không trộn vào phụ kiện vì đơn giá và đơn vị nghiệm thu khác nhau.

Luôn dùng **tên gọi tiếng Việt** thay vì tên block khi trình bày cho người dùng, nhưng giữ cột mã hiệu để tra ngược.

## Lưu ý

- Nếu `HTSKL` báo hệ là `?`, nghĩa là ống không do CHÍNH TOOL vẽ — nói rõ với người dùng rằng số liệu là suy đoán từ MLINE scale.
- Hao hụt mặc định: ống 5%, phụ kiện 2%. Báo cả số trước và sau hao hụt.
- Không tự cộng hao hụt vào bảng chính; để thành dòng riêng.
