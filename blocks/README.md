# `blocks/` — khối dùng lại cho mạch CORDIC

**10 khối `.bdf`**, đã kiểm **đóng phụ thuộc** (mọi khối con nằm trong thư mục này hoặc là
primitive Quartus) và **biên dịch được** bằng Quartus 13.0sp1 cho `EP2C35F672C6`.

Nguồn: `CE118_lab/lab45_24521271/blocks` (6), `CE118_lab/Lab6/Lab6` (2),
`CE118_lab/Assignment_1b` (1). Tất cả tạo bằng Quartus II 13.0 — đúng phiên bản đang cài.

**Đã kiểm:** 8 khối cũ 2026-11-17 (10/10 PASS khi đó), 2 khối mới `not_8bit` + `HA`
2026-11-27 (`quartus_map` PASS). Tổng hiện tại 10/10.

## Danh sách

| File | Cổng | Nguồn | Vai trò |
|---|---|---|---|
| [`fa1`](fa1.bdf) | `A, B, Cin → S, Cout` | lab45 | Bộ cộng toàn phần — nền của mọi bộ cộng |
| [`HA`](HA.bdf) | `A, B → Sum, Cout` | Lab6 | Nửa bộ cộng (bit LSB của bộ đếm) |
| [`mux2_1`](mux2_1.bdf) | `D0, D1, Sel → Y` | lab45 | Mux 1 bit |
| [`mux2_8`](mux2_8.bdf) | `D0, D1[7:0], Sel → Y[7:0]` | lab45 | Ghép 2→16 bit, 3→24 bit |
| [`mux4_8`](mux4_8.bdf) | `D0..D3[7:0], Sel[1:0] → Y[7:0]` | lab45 | Ghép 3 → mux 4:1 24 bit |
| [`not_8bit`](not_8bit.bdf) | `I[7:0] → O[7:0]` | **Lab6** | Ghép 3 → `neg24`, `quadrant_mux` |
| [`reg8`](reg8.bdf) | `D[7:0], CLK, RESET_N, EN → Q[7:0]` | lab45 | ⚠️ reset bất đồng bộ — phải bọc |
| [`addsub8`](addsub8.bdf) | `A, B[7:0], Sub → Y[7:0], Cout` | lab45 | ⚠️ thiếu `Cin` — phải sửa rồi ghép |
| [`count2`](count2.bdf) | `CLK, RST_N, LOAD, STEP → COUNT[1:0], LAST` | Assignment_1b | Ghép 2 → `counter4` |
| [`decoder2_4`](decoder2_4.bdf) | `Addr[1:0], EN → W0..W3` | lab45 | Giải mã 3 trạng thái FSM |

Thiếu `.bsf` (Quartus tự sinh khi mở file): `count2`.

## Đã xoá khỏi thư mục này

| Khối | Vì sao không dùng được |
|---|---|
| `eq4_07` | Chuyển sang Verilog thấy `EQ = (A0·A1)·(A2·¬A3)` → so với **7**, không phải 15. Vẽ `cmp15` mới chỉ 3 cổng AND, hoặc bỏ hẳn dùng `I15 = LAST0 AND LAST1` |
| `state_to_hex` | `H1 = 0` cố định, `H0 = H3` → **không phải** bộ giải mã số 0–3. Không hiển thị được chữ số nào |

## Đã xét trong `CE118_lab/Lab6` và loại

| Khối | Lý do loại |
|---|---|
| `FA_8bit`, `FA_4bit` | Có `Cin` nhưng **chỉ xuất `Overflow`, không có `Cout`** → không truyền nhớ sang byte kế, không ghép được 24 bit. Cùng lỗi `addsub8` nhưng nặng hơn vì phải thêm cả cổng ra |
| `AU`, `ALU`, `Control_Unit`, `datapathV1` | Gắn với bài MIPS (opcode, thanh ghi), trạng thái khác hoàn toàn |
| `Mux21_8bit`, `Decoder24`, `FA`, `Mux21-2bit/3bit`, `Decoder38` | Trùng chức năng với `mux2_8`/`decoder2_4`/`fa1` đã có, chỉ khác tên cổng |
| `tri_8bit` | Tri-state cho bus sharing — mạch này không dùng bus chung |
| `RAM_32x4`, `RAM_32x8`, `RAM_4x4`, `Inst_Mem`, `Register_file`, `RFC` | Bộ nhớ/thanh ghi file 32 bit, không khớp ROM 16×24 |

## `reference/lab6-rom/` — mẫu LPM_ROM để làm `rom_atan`

Không phải khối để compile, chỉ để tham chiếu:

- `ROM.vhd` — ROM 1 cổng do **Megafunction Wizard** sinh (`altsyncram`), address 5 bit,
  32 từ 32 bit. Đây là mẫu đúng cho `rom_atan`: mở MegaWizard, chọn ROM 1 port,
  `address[3:0]`, `q[23:0]`, nạp file `.mif`.
- `ROM.mif` — đúng định dạng MIF cần dùng:

```
WIDTH=24;
DEPTH=16;
ADDRESS_RADIX=UNS;
DATA_RADIX=UNS;
CONTENT BEGIN
    0 : 2097152;
    ...
END;
```

Thay 16 dòng bằng `A[i]` lấy từ `gen_constants.py --markdown` của skill `ce118-cordic-spec`.

> **Không gắn reset vào ROM.** Gắn reset vào mảng → Quartus không suy ra block RAM mà dựng
> bằng 384 flip-flop, bảng tài nguyên sai.

## Bốn cảnh báo khi dùng

**1. `reg8` reset BẤT ĐỒNG BỘ mức thấp.** RTL Quartus sinh ra:
`always@(posedge CLK or negedge RESET_N or negedge VCC)` → `RESET_N` vào chân `CLRN`.
Spec CORDIC cần **đồng bộ, mức 1**. Bọc thành `reg8_sync`:

- `RESET_N = VCC`, `PRN = VCC`
- `D_eff = RST ? 0 : D` (một `mux2_8`, `D0 = D`, `D1 = GND`, `SEL = RST`)
- **`EN_eff = EN OR RST`** (một `OR2`) ← thiếu vế này thì reset không xoá được khi `EN = 0`

Đã mô phỏng 11/11: biến thể "chỉ chặn `D`" giữ nguyên `0xA5` khi `EN=0`, biến thể có `EN|RST`
xoá về 0.

**2. `addsub8` không có `Cin`.** RTL xác nhận `FA0.Cin = Sub` và không có cổng `Cin`.
Sửa thành `addsub8c` (thêm 1 cổng vào `Cin`, nối vào `FA1` LSB; `Sub` vẫn vào 8 cổng `XOR`),
rồi ghép 3 tấn. Đã mô phỏng 31/31 PASS, kể cả `0x00FFFF + 1 = 0x010000`.

**3. `count2` chỉ 2 bit nhưng `LOAD=1` nạp 0 ĐỒNG BỘ** (`EN_C = LOAD|STEP`,
`DC = INC & ~LOAD`). Ghép 2 tấn với `RST_N = VCC`, `LOAD = RST OR LOAD_I`, `LAST0 → STEP1`
thành `counter4`. Đã mô phỏng PASS.

**4. `not_8bit` là khối tổ hợp thuần** (`O = ~I`), không có reset. Ghép 3 tấn cho `neg24`;
nhớ rằng `neg24 = NOT + 1` nên vẫn cần một bộ cộng cộng 1.

## Khối tầng 1 — sinh tự động, đã compile và mô phỏng

14 khối dưới đây **không gõ tay**: sinh bằng `.dsh/tools/gen_tier1.py` (script chỉ chạy
trên máy tác giả, file `.bdf`/`.bsf` là thứ được lưu trong repo). Mở bằng Quartus là thấy
sơ đồ bình thường và sửa tay được.

| Khối | Cổng | Vai trò |
|---|---|---|
| `mux2_16` | `D0,D1[15:0], Sel → Y[15:0]` | 2 × `mux2_8` |
| `mux2_24` | `D0,D1[23:0], Sel → Y[23:0]` | 3 × `mux2_8` |
| `mux4_24` | `D0..D3[23:0], Sel[1:0] → Y[23:0]` | 3 × `mux4_8` |
| `neg24` | `A[23:0] → Q[23:0]` = `~A` | 3 × `not_8bit` (chưa cộng 1) |
| `reg8_sync` | `D[7:0], CLK, RST, EN → Q[7:0]` | `reg8` bọc reset đồng bộ |
| `reg16`, `reg24` | `D[..], CLK, RST, EN → Q[..]` | 2 và 3 × `reg8_sync` |
| `addsub8c` | `A,B[7:0], Sub, Cin → Y[7:0], Cout` | 8 `fa1` + 8 `XOR`, **có `Cin`** |
| `add_sub24` | `A,B[23:0], Sub → S[23:0]` | 3 × `addsub8c` nối nhớ |
| `shift1/2/4/8` | `D[23:0], EN → Q[23:0]` | 24 × `mux2_1`, điền bit dấu |
| `shifter24` | `D[23:0], SH[3:0] → Q[23:0]` | 4 tầng `shift8→4→2→1` |

**Đã kiểm ngày 2026-11-27:**

| Kiểm | Kết quả |
|---|---|
| `quartus_map`, mỗi khối làm top-level | **14/14 PASS** |
| Mô phỏng ModelSim: `mux2_24`, `mux4_24`, `neg24`, `reg8_sync` | **16/16 checks PASS** |
| Mô phỏng: `add_sub24` (8 ca), `shifter24` (10 ca), `reg24` (5 ca) | **23/23 checks PASS** |

Ba ca đáng chú ý: `0x00FFFF + 1 = 0x010000` (nhớ xuyên 2 byte), `-16 >> 5 = -1`
(dịch số học không được ra 0), và `RST` xoá được **dù `EN = 0`**.

## Cách đưa vào project Quartus

1. Chép cả `blocks/` cạnh project, hoặc thêm đường dẫn tìm kiếm:
   `set_global_assignment -name SEARCH_PATH blocks` (dùng `/`, không dùng `\`).
2. Trong BDF cha: **Insert → Symbol**, gõ `fa1`, `mux2_8`…
3. **Không đổi tên file** — `mux2_8` gọi `MUX2_1` bên trong; đổi tên là gãy liên kết.
4. File thiếu `.bsf` thì mở `.bdf` rồi `File → Create/Update → Create Symbol Files`.

## Tận dụng thêm: 74LS283 có sẵn trong Quartus

`CE118_lab/lab3_24521271/lab3_24521271.bdf` đã dùng `74283` (bộ cộng 4 bit **có Cin và
Cout**) và project đó compile ra `.sof` — thư viện này có trong Quartus 13.0. Ghép 6 × 74283
là có bộ cộng 24 bit. Chỉ nên dùng nếu giảng viên cho phép linh kiện MSI.
