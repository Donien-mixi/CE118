# Roadmap CE118 — Mạch sin/cos fixed-point 16 bit bằng CORDIC

**Nhóm:** [A] · [B] · [C]  
**Hạn nộp:** [điền ngày]  
**Thời lượng dự kiến:** 4 tuần

## 1. Phạm vi và sản phẩm

Theo mục 5 và phần yêu cầu chung của [đề bài](CE118%20Project%20Topics%20%281%29.pdf), nhóm thiết kế mạch tính đồng thời sin và cos, trình bày phân tích, biểu thức, lưu đồ và sơ đồ khối, hiện thực và mô phỏng với Quartus, rồi đo tài nguyên, Fmax và số chu kỳ. Nạp kit DE2 là tùy chọn.

Bàn giao project Quartus, RTL, mô hình tham chiếu, vector kiểm thử, testbench, waveform, báo cáo timing và tài nguyên, báo cáo thiết kế và hướng dẫn chạy lại.

## 2. Đặc tả đề xuất

| Mục | Quy ước |
|---|---|
| Góc vào | `phase[15:0]` không dấu. Góc thực bằng `phase × 360° / 65536`. Mã 0000, 4000, 8000, C000 (hex) lần lượt là 0°, 90°, 180°, 270°. |
| Đầu ra | `sin_out`, `cos_out` có dấu, mỗi cổng 16 bit với 14 bit phần lẻ. Chia số nguyên có dấu cho 16384 để giải mã. |
| Nội bộ | `x`, `y` có dấu 24 bit, 21 bit phần lẻ. `z` có dấu 24 bit, một đơn vị tương ứng 1/2²⁴ vòng. |
| Kiến trúc | CORDIC chế độ quay, một bộ tính dùng lại cho 16 vòng, mỗi chu kỳ một vòng. |
| Điều khiển | `clk`, `rst`, `start`, `busy`, `done`. Nhận xung `start` một chu kỳ ở cạnh lên khi `busy=0`. Chốt góc tại cạnh nhận và bỏ qua yêu cầu khi bận. |
| Reset và kết quả | Reset đồng bộ mức 1, hủy phép đang chạy và đưa trạng thái cùng các đầu ra về 0. Giữ kết quả đến lần `done` tiếp theo hoặc reset. |
| Timing | E0 chốt góc, khởi tạo và bật `busy`. E1–E16 chạy 16 vòng. E17 ghi hai kết quả, hạ `busy` và bật `done` một chu kỳ. E18 có thể nhận lệnh tiếp theo. |
| Nghiệm thu | RTL khớp từng bit với mô hình số nguyên. Sai số so với sin/cos lý tưởng đã lượng tử hóa tối đa 2 LSB mỗi đầu ra. |

Nhóm xác nhận với giảng viên về độ rộng nội bộ 24 bit, yêu cầu BDF nếu có, phiên bản Quartus, công cụ mô phỏng và mã FPGA đích. Các thông số trên là lựa chọn thiết kế của nhóm.

## 3. Sơ đồ và thuật toán

```text
phase → tách góc phần tư → CORDIC 16 vòng → khôi phục tọa độ → làm tròn → sin/cos
                            IDLE → ITERATE → OUTPUT → IDLE
```

1. Tách hai bit cao thành góc phần tư `q`, lấy 14 bit thấp làm góc dư `r`. Nạp `z = r × 256`, `y = 0`, `x = 1273502`. Giá trị `x` là hệ số bù độ dài CORDIC cho 16 vòng, đã nhân với 2²¹ và làm tròn.
2. Tạo bảng `A[i] = round(atan(2^(-i)) × 2²⁴ / (2π))` với `i = 0…15`. Mô hình xuất bảng hằng số để RTL dùng chung.
3. Chọn `d = +1` nếu `z ≥ 0`, còn lại chọn `−1`. Cập nhật đồng thời `x' = x − d × (y >>> i)`, `y' = y + d × (x >>> i)`, `z' = z − d × A[i]`. Cả ba vế phải dùng giá trị cũ. `>>>` là dịch phải số học. RTL dùng `d` để chọn cộng hoặc trừ.
4. Gọi `s = y` và `c = x` sau vòng cuối. Cặp `(sin, cos)` ứng với `q = 0, 1, 2, 3` lần lượt là `(s,c)`, `(c,−s)`, `(−s,−c)`, `(−c,s)`.
5. Đổi dấu ở độ rộng nội bộ rồi chuyển từ 21 xuống 14 bit phần lẻ. Chia mã nguyên cho 128 và làm tròn gần nhất, trường hợp đúng nửa bước thì làm tròn ra xa 0.

### Ví dụ chạy tay gần 30°

`phase = 0x1555` tương ứng khoảng 29,998°. Mũi tên bắt đầu ở 0°. Ba bước đầu lần lượt quay +45°, −26,565°, +14,036°, đưa hướng mũi tên qua 45°, 18,435°, rồi 32,471°. Mỗi bước nhìn phần góc còn thiếu để chọn chiều quay tiếp. Sau 16 bước, mô hình cho `sin_out = 8191`, `cos_out = 14189`. Khi debug, A xuất dấu vết `i, x, y, z`, B so các thanh ghi và C kiểm tra hai đầu ra tại E17.

## 4. Phân công và mốc nghiệm thu

| Tuần | A · Thuật toán và đặc tả | B · RTL | C · Kiểm thử và Quartus | Điều kiện qua mốc |
|---|---|---|---|---|
| 1 | Chốt mã hóa, công thức, bảng góc phần tư. | Vẽ sơ đồ khối, lưu đồ FSM và bảng cạnh E0–E18. | Dựng project, chọn FPGA, chạy mô phỏng tối thiểu. | Cả ba thống nhất đặc tả và giải mã được vector 45°. |
| 2 | Viết mô hình số nguyên và tham chiếu lý tưởng, xuất hằng số và vector. | Viết phần tách góc, vòng lặp, điều khiển và đầu ra. | Viết testbench tự so dữ liệu và timing. | Các góc 0°, 45°, 90°, 180°, 270° khớp mô hình, `done` tại E17. |
| 3 | Quét toàn bộ mã góc, phân tích sai số và ghi trường hợp lớn nhất. | Sửa lỗi từ ca tái hiện và chốt phiên bản RTL để đo. | Chạy bộ kiểm thử, compile đầy đủ và thu báo cáo timing/tài nguyên. | Kiểm thử đạt, số liệu lấy từ cùng phiên bản RTL và FPGA đích. |
| 4 | Viết phân tích và sai số. | Hoàn thiện sơ đồ, lưu đồ và bảng chu kỳ. | Hoàn thiện waveform, báo cáo Quartus và hướng dẫn chạy. | Một người khác chạy lại mô phỏng từ bộ bàn giao. |

Cuối mỗi tuần, cả nhóm dành 15 phút kiểm tra điều kiện qua mốc. A quản lý đặc tả, B quản lý RTL, C quản lý testbench và cấu hình Quartus. Nếu đổi cổng, định dạng, số vòng hoặc timing, cập nhật đặc tả, mô hình, RTL và testbench trong cùng lần tích hợp.

### Mẫu đầu việc

| Trường | Ví dụ |
|---|---|
| Mã và tên | RTL-02 · Khôi phục tọa độ và ghi kết quả |
| Người làm / kiểm tra | B / A |
| Phụ thuộc | Bảng góc phần tư và kết quả x/y sau vòng 15 |
| Đầu ra | Module RTL, waveform bốn góc trục |
| Điều kiện đạt | Hai đầu ra khớp mô hình, ghi tại E17 |
| Hạn và bằng chứng | [Ngày] · [File, commit hoặc log] |

## 5. Ma trận kiểm thử

| Mã | Kích thích | Điều kiện đạt |
|---|---|---|
| T01 | 0°, 45°, 90°, 180°, 270° | Hai đầu ra khớp mô hình, đúng dấu và vị trí. |
| T02 | Ranh giới góc phần tư và các mã sát hai bên, kể cả FFFF → 0000 | Hai đầu ra khớp mô hình, sai số lý tưởng trong ngưỡng. |
| T03 | Đổi góc và gửi thêm xung `start` khi bận | Chỉ một `done`, kết quả thuộc góc đã chốt. |
| T04 | Reset giữa phép tính rồi gửi lệnh mới | Phép cũ bị hủy, phép mới trả đúng ở E17 tính từ lần nhận mới. |
| T05 | Gửi lệnh thứ hai ở E18 | Hai kết quả đúng, mỗi xung `done` dài một chu kỳ. |
| T06 | Quét 65.536 mã góc | Ghi số ca lệch bit, sai số cực đại từng đầu ra và mã góc gây sai số. |

Mỗi ca ghi mã góc, kết quả mô hình số nguyên, kết quả RTL, kết quả lý tưởng đã lượng tử hóa và trạng thái đạt/chưa đạt. Đo riêng sai lệch RTL so với mô hình và sai số so với giá trị lý tưởng. Một LSB bằng 1/16384.

## 6. Khung báo cáo và việc làm buổi đầu

1. **Phân tích** — Mục tiêu, cách mã hóa, dải góc và lý do chọn mạch tuần tự.
2. **Thiết kế** — Biểu thức lặp, bảng hằng số, đổi góc phần tư, sơ đồ khối, lưu đồ và bảng chu kỳ.
3. **Hiện thực và mô phỏng** — Top-level, phiên bản công cụ, FPGA đích, cách chạy và waveform.
4. **Độ chính xác** — Số ca chạy, số ca lệch bit và sai số cực đại theo LSB.
5. **Tài nguyên và timing** — Tài nguyên dùng/tổng số [ ], thanh ghi [ ], DSP [ ], RAM [ ], ràng buộc clock [ ] ns, Fmax [ ] MHz, độ trễ đo [ ] chu kỳ. Đính kèm báo cáo Quartus và waveform tương ứng.
6. **Bàn giao** — Tiêu chí đã đạt, mục còn thiếu nếu có và các bước chạy lại.

Trong buổi đầu, điền người phụ trách và hạn nộp, xác nhận công cụ và FPGA, tạo các thư mục `docs/`, `model/`, `rtl/`, `sim/`, `quartus/`. Kiểm tra chung `phase = 0x2000` tương ứng 45°, sin và cos lý tưởng đều lượng tử hóa thành 11585.

## 7. Vẽ mạch theo tầng (từ `blocks/` lên top)

Mạch được dựng **bottom-up** thành 4 tầng, tổng **23 khối**: 6 khối dùng lại từ lab cũ (trong `blocks/`) và 17 khối vẽ mới. Hướng dẫn vẽ từng khối, nối dây từng chân và bảng tra lỗi nằm ở **`docs/ve-mach-theo-tang.md`** — đọc file đó trước khi mở Quartus.

```text
Tầng 4  cordic_sincos.bdf
        ├── controller.bdf
        └── datapath.bdf
Tầng 3      ├── reg16 ×3, reg24 ×3, mux2_24 ×3, add_sub24 ×3,
            ├── shifter24 ×2, rom_atan, counter4, cmp_i15,
            └── quadrant_mux, round_q14 ×2, const_x0
Tầng 2  ← dựng từ tầng 1
Tầng 1  add_sub24, shifter24, neg24, mux2_16, mux2_24, mux4_24, reg16, reg24
```

| Tầng | Khối phải vẽ | Điều kiện qua tầng |
|---|---|---|
| 1 — cơ sở | 8 (`add_sub24`, `shifter24`, `neg24`, `mux2_16`, `mux2_24`, `mux4_24`, `reg16`, `reg24`) | `quartus_map` 0 lỗi; `add_sub24`/`shifter24` đúng 3 giá trị kiểm |
| 2 — chức năng | 6 (`counter4`, `cmp_i15`, `rom_atan`, `quadrant_mux`, `round_q14`, `const_x0`) | 0 lỗi; `round_q14` đúng 6 giá trị kiểm |
| 3 — khối lớn | 2 (`datapath`, `controller`) | testbench 4.096 vector PASS, đúng **18 chu kỳ** `start→done` |
| 4 — top | 1 (`cordic_sincos`) | sweep **65.536 mã** PASS + có `qor.json` + ảnh waveform → **RTL đóng băng** |
| 5 — tùy chọn DE2 | 5 (`sync2`, `edge_detect`, `mul10000`, `bin2bcd_dd`, `seg7`) | ảnh KIT kèm mã `SW` |

**Dùng lại 6 khối** (xem `blocks/README.md`): `fa1`, `mux2_1`, `mux2_8` (ghép 2–3), `mux4_8` (ghép 3), `reg8` (ghép 2–3, **phải sửa reset**), `decoder2_4` (giải mã 3 trạng thái, khớp đúng mã `00/01/10`). `addsub8` chỉ để xem mẫu vì **thiếu `Cin`**, không ghép chuỗi được.

### Ba quyết định kiến trúc đã chốt

1. **18 chu kỳ ⇒ 3 bộ cộng, 2 bộ dịch.** Một vòng lặp cập nhật đồng thời `x`, `y`, `z` nên không thể dùng một bộ cộng. "Một bộ tính dùng lại" ở §2 hiểu là *một datapath dùng lại 16 lần*, không phải một bộ cộng. Nếu chọn 1 bộ cộng thì phải trả giá bằng chu kỳ (~98 chu kỳ, 7 trạng thái).
2. **Ba bộ cộng không cùng cực tính**: `AU_X` và `AU_Z` trừ khi `z ≥ 0`, `AU_Y` cộng khi `z ≥ 0`. Báo cáo tham khảo dùng một `Sel_AU` chung — đúng với mạch của họ, **sai** với mạch song song.
3. **Reset đồng bộ mức 1**, `done` là tín hiệu tổ hợp (rộng đúng 1 chu kỳ). `reg8` trong `blocks/` là reset **bất đồng bộ mức thấp** nên phải sửa trước khi dùng.

Trạng thái phiên làm việc, bằng chứng đã kiểm và việc tiếp theo: xem **`handoff.md`**.
