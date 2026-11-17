# `blocks/` — khối tận dụng từ `CE118_lab`

10 khối `.bdf` chép từ các lab trước, đã kiểm **đóng phụ thuộc**: mọi khối con đều nằm
trong thư mục này hoặc là primitive của Quartus. Không có `.bdf` nào trỏ ra file thiếu.

Nguồn: `CE118_lab/lab45_24521271/blocks` (6 khối), `lab6_24521271/blocks` (1),
`lab2_24521271` (1), `Assignment_1b` (1). Tất cả tạo bằng Quartus II 13.0 — đúng phiên
bản đang cài.

**Đã kiểm ngày 2026-11-17:** cả 10 khối biên dịch qua Analysis & Synthesis của Quartus
13.0sp1 (`quartus_map`, family Cyclone II, device `EP2C35F672C6`), mỗi khối đặt làm
top-level một lần — 10/10 PASS, 0 lỗi.

## Danh sách

| File | Cổng | Nguồn | Dùng cho |
|---|---|---|---|
| `fa1` | `A, B, Cin → S, Cout` | lab45 | `fa` — bộ cộng toàn phần, nền của mọi bộ cộng |
| `mux2_1` | `D0, D1, Sel → Y` | lab45 | khối con của mọi mux |
| `mux2_8` | `D0, D1[7:0], Sel → Y[7:0]` | lab45 | mux 2:1 8 bit |
| `mux4_8` | `D0..D3[7:0], Sel[1:0] → Y[7:0]` | lab45 | mux 4:1 8 bit |
| `reg8` | `D[7:0], CLK, RESET_N, EN → Q[7:0]` | lab45 | thanh ghi 8 bit có enable |
| `addsub8` | `A, B[7:0], Sub → Y[7:0], Cout` | lab45 | **mẫu** bộ cộng/trừ |
| `decoder2_4` | `Addr[1:0], EN → W0..W3` | lab45 | giải mã 3 trạng thái FSM |
| `eq4_07` | `A[3:0] → EQ` | lab6 | **mẫu** so sánh bằng hằng |
| `count2` | `CLK, RST_N, LOAD, STEP → COUNT[1:0], LAST` | Assignment_1b | **mẫu** bộ đếm (không có `.bsf`, Quartus tự sinh) |
| `state_to_hex` | `S1, S0 → H0..H6` | lab2 | **mẫu** giải mã 7 đoạn |

## Phủ được bao nhiêu trong 24 khối CORDIC

| Khối CORDIC | Trạng thái | Cách làm |
|---|---|---|
| `fa` | ✅ có sẵn | dùng `fa1` nguyên |
| `mux2_1_16` | 🟡 ghép | 2 × `mux2_8` (chung `Sel`) |
| `mux2_1_24` | 🟡 ghép | 3 × `mux2_8` |
| `mux4_1_24` | 🟡 ghép | 3 × `mux4_8` (chung `Sel[1:0]`) |
| `reg16_load` | 🟡 ghép | 2 × `reg8` — **phải sửa reset** |
| `reg24_load` | 🟡 ghép | 3 × `reg8` — **phải sửa reset** |
| `counter4` | 🟡 mẫu | `count2` chỉ 2 bit; mở rộng lên 4 bit |
| `cmp_i15` | 🟡 mẫu | `eq4_07` so với 7 → đổi thành so với 15 |
| `seg7` (DE2) | 🟡 mẫu | `state_to_hex` cùng kiểu vẽ, khác bảng giải mã |
| `add_sub24` | 🔴 vẽ mới | 24 × `fa1`; `addsub8` **không ghép được** (xem cảnh báo 2) |
| `shifter24` | 🔴 vẽ mới | dựng từ `mux2_8` |
| `neg24` | 🔴 vẽ mới | `fa1` + `NOT` |
| `rom_atan` | 🔴 vẽ mới | `LPM_ROM` + `.mif` |
| `quadrant_mux` | 🔴 vẽ mới | `neg24` + `mux4_1_24` |
| `round_q14` | 🔴 vẽ mới | `neg24` + bộ tăng 17 bit |
| `const_x0` | 🔴 vẽ mới | `LPM_CONSTANT` |
| `datapath`, `controller`, `cordic_sincos` | 🔴 vẽ mới | ghép các khối trên |
| 5 khối DE2 còn lại | 🔴 vẽ mới | `lab2_24521271\DE2_MEALY.bdf` là mẫu nối chân board |

Tóm lại: **6 khối dùng được ngay (ghép nhiều bản cho đủ độ rộng), 4 khối làm mẫu, 14
khối phải vẽ mới** — nhưng mọi khối mới đều dựng từ `fa1` và `mux2_1` đã có.

## Bốn cảnh báo — đọc trước khi dùng

**1. `reg8` dùng reset BẤT ĐỒNG BỘ, mức thấp.** Nó nối `RESET_N` vào chân `CLRN` của
`DFFE`, tức xoá ngay khi `RESET_N=0`, không chờ clock. Spec CORDIC yêu cầu **reset đồng
bộ, mức 1**. Phải sửa một trong hai cách:

- đổi cực tính ở cổng vào (`RESET_N = ~rst`) **và** chấp nhận reset bất đồng bộ — nhanh
  nhưng sai spec, dễ mất điểm ở mục T04;
- nối `CLRN`/`PRN` lên VCC rồi **cổng hoá đầu vào D**: `D_eff = rst ? 0 : D` — đúng spec
  reset đồng bộ. Cách này đúng, tốn thêm 8 cổng AND mỗi khối 8 bit.

**2. `addsub8` không có `Cin`.** Cổng chỉ có `A, B, Sub → Y, Cout`. Ghép 3 khối để thành
24 bit **sẽ sai** vì khối sau không nhận được nhớ của khối trước. Hoặc thêm `Cin` vào
`addsub8`, hoặc vẽ `add_sub24` trực tiếp từ 24 × `fa1` (khuyến nghị — sạch và đúng ý đồ
"vẽ từ cổng").

**3. `eq4_07` so với hằng 7, không phải 15.** Cấu trúc dùng được, nhưng phải đổi hằng:
với 7 thì bit 3 = 0 nên có cổng NOT; với 15 thì cả bốn bit = 1 nên chỉ cần AND.

**4. `state_to_hex` chưa rõ mức tích cực.** Đầu ra `H0..H6` có thể là mức cao; DE2 cần
**mức thấp**. Trước khi dùng, thử cho `S1S0 = 00` và xem LED có sáng đúng số 0 không.

## Cách đưa vào project Quartus

1. Chép cả thư mục `blocks/` vào cạnh project, hoặc thêm đường dẫn tìm kiếm:
   ```
   set_global_assignment -name SEARCH_PATH blocks
   ```
   (đường dẫn dùng `/`, đừng dùng `\` — Tcl hiểu `\b` thành ký tự backspace).
2. Trong BDF cha: **Insert → Symbol**, gõ `fa1`, `mux2_8`… Quartus tìm theo tên file.
3. **Không đổi tên file.** `mux2_8.bdf` gọi `MUX2_1` bên trong; đổi tên `mux2_1.bdf`
   thành `mux_1bit.bdf` là gãy liên kết. Muốn đổi tên thì phải sửa mọi file gọi nó.
4. File nào thiếu `.bsf` (`count2`) thì mở `.bdf` rồi **File → Create/Update → Create
   Symbol Files for Current File**.

## Tận dụng thêm: 74LS283 và 74LS151 có sẵn trong Quartus

`CE118_lab/lab3_24521271/lab3_24521271.bdf` đã dùng `74283` (bộ cộng 4 bit **có Cin và
Cout**) và `74151` (mux 8:1), và project đó đã compile ra `.sof` — tức thư viện này có
trong Quartus 13.0.

Nếu muốn bộ cộng 24 bit nhanh mà không vẽ 24 FA: ghép **6 × 74283** nối chuỗi nhớ. Đổi
lại là mất phần "vẽ từ cổng" — chỉ nên dùng nếu giảng viên cho phép dùng linh kiện MSI.

## Không chép (và vì sao)

`alu8`, `alu32`, `alu_control`, `control_unit`, `controller`, `datapath` của các lab:
FSM và datapath gắn với bài toán khác (nhân tuần tự, MIPS), trạng thái và tín hiệu điều
khiển khác hoàn toàn — chép vào chỉ gây nhầm. `controller.bdf` của lab45 vẫn hữu ích để
**xem cách trình bày** một FSM bằng DFF + cổng: `CE118_lab/lab45_24521271/blocks/controller.bdf`.

`rf4x8`, `rf32x32`, `storage32x32`, `dmem32x32`, `imem32x32`, `rom_word_*`: bộ nhớ/thanh
ghi file 32 bit, không khớp ROM 16×24.
