# Vẽ mạch theo tầng — CORDIC sin/cos (CE118)

Tài liệu này là **hướng dẫn dựng hình**: từ 6 khối có sẵn trong `blocks/`, vẽ dần lên
23 khối của mạch CORDIC, chia 4 tầng. Mỗi khối ghi rõ: cổng, bên trong, lấy gì từ
`blocks/`, các bước vẽ, và lỗi hay gặp.

Đọc kèm: `roadmap.md` §7 (tổng quan), `blocks/README.md` (khối tận dụng),
skill `ce118-cordic-spec` (con số), `ce118-report-vn` (cách trình bày).

## Nguyên tắc chung

1. **Bottom-up tuyệt đối.** Vẽ xong khối nào thì *Create Symbol Files for Current File*
   rồi mới dùng ở khối cha. Không vẽ top trước.
2. **Không đổi tên file `.bdf`.** `mux2_8.bdf` gọi `MUX2_1` bên trong; đổi tên là gãy
   liên kết im lặng (Quartus báo "can't find design entity").
3. **Đường dẫn tìm kiếm dùng `/`, không dùng `\`.** Tcl đọc `\b`, `\r`, `\t` thành ký tự
   điều khiển và làm hỏng `.qsf`.
4. **File `.v`/`.bdf` lưu ASCII hoặc UTF-8 không BOM.** Quartus 13.0 báo
   `syntax error near text` nếu có BOM.
5. **Bus đặt tên có chỉ số**: `D[23..0]`, gõ `D[23..0]` vào nhãn bus, không nối 24 dây rời.
6. **Reset**: spec yêu cầu **đồng bộ, mức 1**. `reg8` trong `blocks/` là **bất đồng bộ,
   mức thấp** → xem §1.4.

## Ánh xạ `blocks/` → khối cần vẽ

| Trong `blocks/` | Dùng thế nào |
|---|---|
| `fa1` | gọi trực tiếp trong `add_sub24`, `neg24`, `counter4` |
| `mux2_1` | mux 1 bit lẻ |
| `mux2_8` | 2 tấn → `mux2_16`, 3 tấn → `mux2_24`, 4 tấn → `shifter24` mỗi tầng |
| `mux4_8` | 3 tấn → `mux4_24` |
| `reg8` | 2 tấn → `reg16`, 3 tấn → `reg24` (**phải sửa reset**) |
| `decoder2_4` | giải mã 3 trạng thái trong `controller` |
| `addsub8` | **chỉ để xem mẫu**, không ghép được (thiếu `Cin`) |
| `eq4_07` | mẫu so sánh hằng, đổi 7 → 15 |
| `count2` | mẫu bộ đếm có `LOAD`/`STEP`, mở rộng lên 4 bit |
| `state_to_hex` | mẫu giải mã 7 đoạn (DE2) |

---

# Tầng 1 — khối cơ sở (8 khối phải vẽ)

Đây là các khối nhỏ nhất dùng lại được. Vẽ đúng tầng này thì các tầng trên chỉ là nối dây.

## 1.1 `mux2_16` — mux 2:1, 16 bit

- **Cổng**: `D0[15:0], D1[15:0], Sel` → `Y[15:0]`
- **Bên trong**: 2 × `mux2_8`. `D0[15:8]/D1[15:8]` → mux thứ nhất, `D0[7:0]/D1[7:0]` → mux
  thứ hai; **cùng một `Sel`**.
- Vẽ: đặt 2 symbol `mux2_8`, nối `Sel` chung, rồi gộp 2 ngõ ra thành bus `Y[15:0]`.

## 1.2 `mux2_24` — mux 2:1, 24 bit

- **Cổng**: `D0[23:0], D1[23:0], Sel` → `Y[23:0]`
- **Bên trong**: 3 × `mux2_8` (byte 23:16, 15:8, 7:0), `Sel` chung.
- Dùng cho: chọn giữa hằng số khởi tạo và kết quả AU; chọn `D0/D1` trong mỗi tầng shifter.

## 1.3 `reg24` — thanh ghi 24 bit có Load

- **Cổng**: `D[23:0], CLK, RST, EN` → `Q[23:0]`
- **Bên trong**: 3 × `reg8`.
- **Khác `reg8` ở reset** — xem 1.4. Nếu bạn để nguyên `reg8`, `reg24` sẽ reset mức thấp
  bất đồng bộ: đổi tên cổng thành `RST_N` cho khỏi nhầm, **hoặc** làm theo 1.4.

## 1.4 Sửa `reg8` cho reset đồng bộ mức cao (bắt buộc)

`reg8` gốc: `RESET_N` → chân `CLRN` của `DFFE` (xoá ngay, không chờ clock).

Cách đúng spec, vẽ thành `reg8_sync`:

1. Mở `reg8.bdf` (đừng sửa file gốc — Save As thành `reg8_sync.bdf`).
2. Nối `PRN` và `CLRN` của cả 8 `DFFE` **lên VCC**.
3. Thêm 8 cổng `AND2`: `D_eff[i] = D[i] AND (NOT RST)`; đầu ra `AND2` vào `D` của `DFFE`.
4. Đổi cổng `RESET_N` thành `RST` (mức 1 = reset).
5. `EN` giữ nguyên vào chân `ENA`.

Kết quả: reset chỉ tác dụng ở cạnh clock → đúng yêu cầu T04.

## 1.5 `add_sub24` — bộ cộng/trừ 24 bit

- **Cổng**: `A[23:0], B[23:0], Sub` → `S[23:0]`
- **Bên trong**: 24 × `fa1` nối chuỗi nhớ (ripple carry).
  - `B_eff[i] = B[i] XOR Sub` → 24 cổng `XOR` (hoặc dùng `mux2_1` nếu muốn rõ ý).
  - `Cin` của `fa1` bit 0 = `Sub`.
  - `Cout` của `fa1` bit i → `Cin` của `fa1` bit i+1.
  - `S[i] = S` của `fa1` bit i.
- **Không ghép từ `addsub8`**: `addsub8` không có `Cin` nên không truyền nhớ được.
- Kiểm nhanh: `Sub=0, A=5, B=3` → `S=8`; `Sub=1, A=5, B=3` → `S=2`;
  `Sub=1, A=0, B=1` → `S=0xFFFFFF` (đúng bù 2 của −1).

## 1.6 `shifter24` — dịch phải số học 24 bit, 4 tầng

- **Cổng**: `D[23:0], SH[3:0]` → `Q[23:0]`
- **Bên trong**: 4 tầng `mux2_24`, mỗi tầng dịch 8 / 4 / 2 / 1 bit.

Mỗi tầng: `Q = SHK ? (cur >>> K) : cur`, với `>>> K` **điền bằng bit dấu của `cur`**.

| Tầng | Chọn bởi | Dịch | Nối dây khi dịch 8 bit |
|---|---|---|---|
| 1 | `SH[3]` | 8 | `q[23:16] = cur[23]` (8 lần), `q[15:0] = cur[23:8]` |
| 2 | `SH[2]` | 4 | `q[23:20] = cur[23]` (4 lần), `q[19:0] = cur[23:4]` |
| 3 | `SH[1]` | 2 | `q[23:22] = cur[23]`, `q[21:0] = cur[23:2]` |
| 4 | `SH[0]` | 1 | `q[23] = cur[23]`, `q[22:0] = cur[23:1]` |

- **Đây là chỗ dễ sai nhất.** Điền 0 thay vì điền bit dấu thì mọi góc âm sai hoàn toàn,
  trong khi góc dương vẫn đúng — rất khó lần ra nếu chỉ nhìn waveform vài góc dương.
- Kiểm: `D = 0xFFFFF8` (−8), `SH=1` → `−4 = 0xFFFFFC`; `SH=3` → `−1 = 0xFFFFFF`.

## 1.7 `neg24` — đảo dấu bù 2

- **Cổng**: `A[23:0]` → `Q[23:0]`
- **Bên trong**: `Q = NOT A + 1` → 24 cổng `NOT` + 1 `add_sub24` (nối `B=0, Sub=1, Cin`
  vào bit 0), hoặc 24 `fa1` với `Sub=1`.
- Dùng ở: `quadrant_mux` (tạo `−s`, `−c`) và `round_q14` (lấy trị tuyệt đối).

## 1.8 `mux4_24` — mux 4:1, 24 bit

- **Cổng**: `D0..D3[23:0], Sel[1:0]` → `Y[23:0]`
- **Bên trong**: 3 × `mux4_8`, `Sel[1:0]` chung.

## Checkpoint tầng 1

Compile từng khối làm top-level (`quartus_map`, hoặc dùng `quartus-build-timing`). Điều
kiện qua tầng: **0 lỗi**, và `add_sub24` / `shifter24` kiểm bằng 3 giá trị ở trên.
Không cần mô phỏng từng khối — tầng 3 sẽ phủ hết.

---

# Tầng 2 — khối chức năng (6 khối)

## 2.1 `counter4` — bộ đếm 4 bit

- **Cổng**: `CLK, RST, LOAD, EN` → `Q[3:0]`
- **Bên trong**: 4 `DFFE` + 4 `fa1` (cộng 1) + mux chọn `0 / Q+1`; `LOAD` nạp 0, `EN` đếm.
- Mẫu: `blocks/count2.bdf` (2 bit, có `LOAD`/`STEP`/`LAST`) — mở rộng lên 4 bit, và đổi
  reset sang đồng bộ mức cao như 1.4.
- **Không cần chức năng `LAST`**: tín hiệu dừng do `cmp_i15` lo.

## 2.2 `cmp_i15` — so sánh `i == 15`

- **Cổng**: `I[3:0]` → `EQ`
- **Bên trong**: `EQ = I[3] AND I[2] AND I[1] AND I[0]` — 3 cổng `AND2` nối tầng.
- Mẫu: `blocks/eq4_07.bdf` (so với 7). **Đổi hằng 7 → 15**: với 7 thì bit 3 = 0 nên có
  `NOT`; với 15 thì cả bốn bit = 1 nên chỉ cần AND.

## 2.3 `rom_atan` — bảng góc 16 × 24 bit

- **Cổng**: `ADDR[3:0]` → `DATA[23:0]`
- **Cách vẽ khuyến nghị**: `LPM_ROM` (Megafunction Wizard), `address[3:0]`,
  `q[23:0]`, nạp file `.mif`.
- Sinh `.mif` từ skill `cordic-golden-model` / `ce118-cordic-spec`:

```powershell
& $py .dsh\skills\ce118-cordic-spec\scripts\gen_constants.py --markdown   # lấy A[i]
```

  rồi ghi 16 dòng `d[23:0]` theo thứ tự `i = 0…15` vào `rom_atan.mif` (định dạng
  `WIDTH=24; DEPTH=16; ADDRESS_RADIX=UNS; DATA_RADIX=UNS;`).
- **Không gắn reset vào ROM.** Gắn reset vào mảng → Quartus không suy ra block RAM mà
  dựng bằng 384 flip-flop, bảng tài nguyên sai.
- Nếu giảng viên bắt vẽ từ cổng: dùng 16 ngõ ra hằng + `mux32_32` kiểu `imem32x32` của
  lab6, nhưng cách đó tốn hàng trăm LE — chỉ làm nếu bị yêu cầu.

## 2.4 `quadrant_mux` — tách góc phần tư

- **Cổng**: `S[23:0], C[23:0], Q[1:0]` → `SIN_RAW[23:0], COS_RAW[23:0]`
  (`S = R_Y`, `C = R_X` sau 16 vòng)
- **Bên trong**: 2 × `neg24` (tạo `−S`, `−C`) + 2 × `mux4_24`:

| `Q` | `SIN_RAW` | `COS_RAW` | mux chọn |
|---|---|---|---|
| 0 | `S` | `C` | `D0` |
| 1 | `C` | `−S` | `D1` |
| 2 | `−S` | `−C` | `D2` |
| 3 | `−C` | `S` | `D3` |

## 2.5 `round_q14` — làm tròn Q2.21 → Q1.14

- **Cổng**: `V[23:0]` → `OUT[15:0]`
- **Công thức**: `OUT = sign(V) × ( |V|[23:7] + |V|[6] )`
- **Bên trong** (theo thứ tự): `neg24` + `mux2_24` để lấy `|V|` (chọn theo `V[23]`);
  lấy `|V|[23:7]` (17 bit) và `|V|[6]`; một bộ tăng 17 bit (dùng `add_sub24` với
  `B = 1`); rồi `mux2_16` khôi phục dấu.
- **Đừng vẽ bộ cộng 24 bit cộng hằng 64.** Cộng 64 rồi bỏ 7 bit thấp chỉ còn đúng một
  bit nhớ là `V[6]` — đây là cách đúng và rẻ hơn 24 LE mỗi đầu ra.
- Kiểm: `V = 16384` → `10000`? Không: `V=16384` là 1.0 → `OUT = 16384`. `V = 64` (nửa
  LSB) → `OUT = 1` (làm tròn lên). `V = −64` → `OUT = −1`.

## 2.6 `const_x0` — hằng số 1273502

- **Cổng**: → `Y[23:0]`
- **Bên trong**: `LPM_CONSTANT` giá trị `1273502` (24 bit); `1273502 = 0x136E9E`.
- Quartus sinh hằng từ giá trị thập phân nên không phải gõ hex. Điều cần kiểm là
  `gen_constants.py --constants` trả `x0 = 1273502`, và bộ cộng thử `x0 + 0` ra đúng
  `0x136E9E` trong mô phỏng.

## Checkpoint tầng 2

Compile từng khối, 0 lỗi. Riêng `round_q14` kiểm 6 giá trị ở trên bằng bảng chân trị
(Megafunction/Simulator của Quartus hoặc testbench nhỏ).

---

# Tầng 3 — khối lớn (2 khối)

## 3.1 `datapath`

- **Cổng vào**: `CLK, RST, PHASE[15:0], LOAD_PHASE, LOAD_X, LOAD_Y, LOAD_Z, LOAD_I, INC_I, LOAD_OUT`
- **Cổng ra**: `SIN_OUT[15:0], COS_OUT[15:0], I15, I_DBG[3:0]`

Bên trong — nối dây đúng thứ tự này:

| # | Khối | Nối |
|---|---|---|
| 1 | `reg16` (R_PHASE) | `D=PHASE`, `EN=LOAD_PHASE` |
| 2 | 3 × `reg24` (R_X, R_Y, R_Z) | `EN = LOAD_X / LOAD_Y / LOAD_Z` |
| 3 | 2 × `shifter24` | `SX = R_X >>> I`, `SY = R_Y >>> I` (`SH = I[3:0]`) |
| 4 | `rom_atan` | `ADDR = I[3:0]` → `A` |
| 5 | 3 × `add_sub24` | `XAU = R_X ∓ SY`, `YAU = R_Y ± SX`, `ZAU = R_Z ∓ A` |
| 6 | 3 × `mux2_24` | chọn khởi tạo: `LOAD_PHASE ? const : AU` |
| 7 | `counter4` | `LOAD=LOAD_I`, `EN=INC_I` → `I[3:0]` |
| 8 | `cmp_i15` | `I[3:0]` → `I15` |
| 9 | `quadrant_mux` | `S=R_Y`, `C=R_X`, `Q=R_PHASE[15:14]` |
| 10 | 2 × `round_q14` | `SIN_RAW → SIN_OUT`, `COS_RAW → COS_OUT` qua `reg16` với `EN=LOAD_OUT` |
| 11 | `const_x0` | `Y → mux` của R_X |

Ba bộ cộng **không cùng cực tính** (đây là chỗ chép từ báo cáo tham khảo sẽ sai):

| Khối | `Sub` nối vào | Vì |
|---|---|---|
| `XAU` | `NOT R_Z[23]` | z ≥ 0 thì `x − (y>>>i)` |
| `YAU` | `R_Z[23]` | z ≥ 0 thì `y + (x>>>i)` |
| `ZAU` | `NOT R_Z[23]` | z ≥ 0 thì `z − A[i]` |

Giá trị khởi tạo:

- `R_X ← 1273502` (`const_x0`)
- `R_Y ← 0` (`GND`)
- `R_Z ← {2'b00, PHASE[13:0], 8'h00}` — 24 bit, chính là `r × 256`

## 3.2 `controller`

- **Cổng vào**: `CLK, RST, START, I15`
- **Cổng ra**: `LOAD_PHASE, LOAD_X, LOAD_Y, LOAD_Z, LOAD_I, INC_I, LOAD_OUT, BUSY, DONE, STATE[1:0]`

Mã hoá 2 bit: `Q1Q0 = 00` IDLE, `01` ITER, `10` OUT.

**Đường trạng thái** (2 `DFFE` + cổng):

```
D1 = Q0 · I15
D0 = (¬Q1 · ¬Q0 · START) ∨ (Q0 · ¬I15)
```

**Giải mã trạng thái**: một `decoder2_4` với `ADDR = {Q1, Q0}`, `EN = 1` →
`W0 = IDLE`, `W1 = ITER`, `W2 = OUT` (`W3` bỏ trống). Đây là lý do `blocks/decoder2_4`
khớp đúng: mã hoá của bạn trùng với thứ tự ngõ ra của nó.

**Ngõ ra** (chỉ phụ thuộc trạng thái):

```
LOAD_PHASE = LOAD_X = LOAD_Y = LOAD_Z = LOAD_I = W0 · START
INC_I  = W1 · ¬I15
LOAD_OUT = W2
BUSY   = W1 ∨ (W0 · START)
DONE   = W2
STATE  = {Q1, Q0}
```

`RST` đồng bộ: đưa `Q1Q0 ← 00`, các ngõ ra về 0.

`DONE = W2` là **tổ hợp** nên chỉ cao đúng 1 chu kỳ. Nếu bạn đăng ký `DONE` (cho qua
`DFF`), nó sẽ cao 2 chu kỳ → sai T05.

## Checkpoint tầng 3

Nối `datapath` + `controller` bằng dây tạm ở top, rồi chạy testbench 4.096 vector
(skill `modelsim-tb-verify`) với `cordic_ref.v` thay bằng BDF của bạn.

Điều kiện qua tầng: `[FINISH] PASS`, và đúng **18 chu kỳ** từ `start` tới `done`.
Nếu lệch bit ở góc âm → kiểm 1.6; lệch ở 90/180° → kiểm 2.4; lệch đều 1 LSB → kiểm 2.5.

---

# Tầng 4 — top (1 khối)

## 4.1 `cordic_sincos`

- **Cổng vào**: `CLK, RST, START, PHASE[15:0]`
- **Cổng ra**: `SIN_OUT[15:0], COS_OUT[15:0], BUSY, DONE`, và **`STATE[1:0]`, `I_DBG[3:0]`
  đưa ra ngoài chỉ để chụp waveform cho báo cáo**.

Nối: 9 ngõ ra điều khiển `controller → datapath`, 2 ngõ ra hồi tiếp
`datapath → controller` (`I15`), `CLK`/`RST` chung.

## Checkpoint tầng 4

1. Compile đầy đủ với `EP2C35F672C6`, `create_clock -period 20.000`.
2. Sweep **65.536 mã góc** khớp bit với model.
3. Đo Fmax / tài nguyên bằng skill `quartus-build-timing`, lưu `qor.json`.
4. Chụp waveform một góc dương và một góc âm.

Điều kiện qua tầng: PASS toàn dải + có `qor.json` + có ảnh waveform. **Đây là mốc "RTL
đóng băng"** — sau đó không sửa RTL nữa, chỉ viết báo cáo.

---

# Tầng 5 (tùy chọn) — DE2

Chỉ làm nếu nạp KIT. 5 khối: `sync2` (2 DFF đồng bộ `KEY`), `edge_detect` (xung `start`
1 chu kỳ), `mul10000` (4 phép dịch + cộng, không dùng bộ nhân), `bin2bcd_dd` (Double
Dabble 14 → 20 bit), `seg7` (giải mã, **active LOW**). Bảng pin:
`.dsh/skills/de2-board-bringup/references/de2-pins.md`.

---

# Thứ tự thực hiện và ước lượng

| Bước | Khối | Thời gian ước tính |
|---|---|---|
| 1 | `mux2_16`, `mux2_24`, `mux4_24`, `reg8_sync`, `reg16`, `reg24` | 2–3 h |
| 2 | `add_sub24`, `neg24`, `shifter24` | 2–3 h |
| 3 | `counter4`, `cmp_i15`, `rom_atan`, `const_x0` | 2 h |
| 4 | `quadrant_mux`, `round_q14` | 1,5 h |
| 5 | `datapath` (nối 22 symbol) | 3–4 h |
| 6 | `controller` | 1,5 h |
| 7 | `cordic_sincos` + testbench + compile | 2 h |

Tổng khoảng **14–18 h** cho người đã quen BDF. Tài nguyên dự kiến **500–800 LE** trên
33.216 LE của EP2C35 — nếu compile ra trên 2.000 LE thì ROM đã bị suy thành thanh ghi
hoặc nối mux sai.

# Bảng tra nhanh lỗi

| Hiện tượng | Khoanh vùng |
|---|---|
| Góc dương đúng, góc âm sai | `shifter24` điền 0 thay vì bit dấu; hoặc cực tính `Sub` của `XAU`/`ZAU` |
| Sai dấu ở góc tù (90–270°) | `quadrant_mux` sai bảng chọn, hoặc `Q` lấy sai bit (`PHASE[15:14]`) |
| Lệch đều 1 LSB mọi góc | `round_q14` (thiếu `+V[6]`) |
| Lệch vài LSB, tăng dần theo góc | `rom_atan` sai thứ tự `A[i]` hoặc `.mif` sai radix |
| `done` dài 2 chu kỳ | `DONE` bị đăng ký qua `DFF` |
| Không bao giờ `done` | `I15` không bao giờ bằng 1: `counter4` không đếm, hoặc `cmp_i15` so sai hằng |
| Tài nguyên > 2.000 LE | ROM gắn reset; hoặc `mux32_32` thay vì `LPM_ROM` |
