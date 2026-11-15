# Kế hoạch đề tài: Datapath ALU 8-bit tích hợp bộ đồng xử lý EDAC (CRC-4 + Hamming SEC-DED)

* **Môn học:** Thiết kế vi mạch (VLSI) – Thiết kế full-custom trên Cadence Virtuoso
* **Nhóm thực hiện:** 5 thành viên (TV1 đến TV5)
* **Quy trình (Flow):** Schematic $\rightarrow$ Mô phỏng Spectre $\rightarrow$ Layout $\rightarrow$ DRC/LVS $\rightarrow$ Trích xuất ký sinh (PEX) $\rightarrow$ Mô phỏng sau layout $\rightarrow$ So sánh, báo cáo

> (kế hoạch này đang giả định, chưa xác nhận):**
> 1. **PDK sử dụng:** 90nm, Samsung 130nm hay Silterra 180nm? *(Khuyến nghị: dùng đúng PDK trong bài thầy hướng dẫn layout/DRC/LVS vì có video để làm theo).*
> 2. **Phạm vi Layout:** Thầy có bắt buộc layout toàn hệ thống không, hay chỉ cần một phần? *(Xem video “Hướng dẫn trình bày và nộp tập tin tiểu luận cuối kỳ VLSI” trong playlist).*
> 3. **Công cụ kiểm tra:** Công cụ DRC/LVS/PEX cài ở trường (Assura, PVS hay Calibre; Quantus/RCX có dùng được không).
> 
> *Lưu ý: Các con số ước lượng trong file (số transistor, số cổng) là tính sơ bộ, phải đếm lại sau khi chốt cell.*

---

## 1. Mục tiêu và điểm khác biệt

### 1.1 Mục tiêu
Thiết kế ở mức transistor một datapath gồm ALU 8-bit và bộ đồng xử lý bảo vệ dữ liệu:
* **Hamming SEC-DED (13,8):** Tự sửa lỗi 1 bit, phát hiện lỗi 2 bit.
* **CRC-4:** Kiểm tra đầu-cuối sau khi sửa, bắt các trường hợp Hamming sửa sai khi có từ 3 bit lỗi trở lên.
* **Khối bơm lỗi (Error Injector):** Cố tình lật bit trên “kênh truyền” để chứng minh mạch hoạt động.

### 1.2 Điểm khác biệt so với các đề tài đã có trong playlist

| Đề tài cũ | Đề tài này hơn ở đâu |
| :--- | :--- |
| ALU 8-bit / ALU 74181 | Có thêm lớp bảo vệ dữ liệu và đánh giá chi phí của nó |
| Parity generator & checker 8-bit | Không chỉ phát hiện mà còn sửa lỗi; có phân biệt lỗi 1 bit và 2 bit |
| Các đề tài chỉ mô phỏng | Có layout, DRC/LVS, PEX và so sánh trước/sau layout |

### 1.3 Câu hỏi nghiên cứu (Phần tạo điểm học thuật)
1. Thêm EDAC làm tăng bao nhiêu diện tích, công suất, độ trễ so với ALU trần?
2. Trong các kiểu cổng XOR (CMOS tĩnh, transmission-gate, pass-transistor), kiểu nào tối ưu cho cây XOR của Hamming/CRC?
3. Mức bảo vệ tăng dần ($\text{Parity} \rightarrow \text{Hamming SEC} \rightarrow \text{SEC-DED} \rightarrow \text{SEC-DED + CRC}$) đổi lấy chi phí thế nào?
4. Ký sinh sau layout làm kết quả lệch bao nhiêu so với mô phỏng schematic?

---

## 2. Đặc tả hệ thống (Spec)

### 2.1 Sơ đồ khối

```text
A[7:0] ─┬─► ALU 8-bit ─► R[7:0] ─┐
B[7:0] ─┘     ▲                  ├─► MUX_IN ─► D[7:0] ─► ENCODER ─► CW[16:0]
OP[2:0] ──────┘        A[7:0] ───┘              (Hamming P[4:0] + CRC C[3:0])
                                                              │
ERR[16:0] ────────────────────────────────────► ERROR INJECTOR (17 XOR)
                                                              │ CW'[16:0]
                                                              ▼
                                   DECODER: syndrome ─► sửa bit ─► kiểm CRC
                                                              │
        R (bypass), D' (thô), D_corr (đã sửa) ─► MUX_OUT ─► Y[7:0]
                                          Cờ: Z N C V | SEC DED CRC_FAIL UE
```

* **Từ mã trên kênh:** `CW[16:0] = {C[3:0], P0, P8, P4, P2, P1, D[7:0]}` (17 bit).

### 2.2 Chân tín hiệu Top-level

| Tín hiệu | Chiều | Số bit | Ý nghĩa |
| :--- | :---: | :---: | :--- |
| `A`, `B` | Vào | 8, 8 | Toán hạng |
| `OP` | Vào | 3 | Mã lệnh ALU |
| `MODE` | Vào | 2 | Chế độ hoạt động |
| `ERR` | Vào | 17 | Mặt nạ bơm lỗi (bit nào = 1 thì bit đó của `CW` bị lật) |
| `Y` | Ra | 8 | Kết quả |
| `Z, N, C, V` | Ra | 4 | Cờ ALU |
| `SEC` | Ra | 1 | Đã sửa 1 bit lỗi |
| `DED` | Ra | 1 | Phát hiện 2 bit lỗi (không sửa được) |
| `CRC_FAIL` | Ra | 1 | CRC của dữ liệu sau sửa không khớp |
| `UE` | Ra | 1 | Lỗi không sửa được (tổng hợp) |
| `CLK`, `RST` | Vào | 1, 1 | Chỉ có ở bản mở rộng có thanh ghi |

### 2.3 Chế độ hoạt động

| MODE | Tên | Đường dữ liệu | Dùng để |
| :---: | :--- | :--- | :--- |
| `00` | **BYPASS** | `Y` = kết quả ALU | Đo baseline không bảo vệ |
| `01` | **PROTECTED** | $\text{ALU} \rightarrow \text{mã hóa} \rightarrow \text{kênh} \rightarrow \text{giải mã} \rightarrow \text{Y đã sửa}$ | Chế độ chính |
| `10` | **CODEC** | $\text{A} \rightarrow \text{mã hóa} \rightarrow \text{kênh} \rightarrow \text{giải mã} \rightarrow \text{Y}$ | Kiểm thử riêng EDAC |
| `11` | **RAW** | `Y` = dữ liệu trên kênh, không sửa | Demo: cho thấy lỗi nếu không có EDAC |

### 2.4 ALU 8-bit

| OP | Lệnh | Mô tả | Cờ |
| :---: | :--- | :--- | :---: |
| `000` | `ADD` | $A + B$ | `Z N C V` |
| `001` | `SUB` | $A - B$ ($A + \sim B + 1$) | `Z N C V` |
| `010` | `AND` | $A \ \& \ B$ | `Z N` |
| `011` | `OR` | $A \mid B$ | `Z N` |
| `100` | `XOR` | $A \oplus B$ | `Z N` |
| `101` | `NOT` | $\sim A$ | `Z N` |
| `110` | `SHL` | $A \ll 1, \quad C = A_7$ | `Z N C` |
| `111` | `SHR` | $A \gg 1, \quad C = A_0$ | `Z N C` |

* **Cấu trúc:** Bộ cộng ripple-carry 8 bit (full adder kiểu mirror 28T), hàng XOR đảo B cho phép trừ, khối logic, MUX 8:1 cho mỗi bit, mạch tạo cờ.

### 2.5 Hamming SEC-DED (13,8)
Vị trí trong từ mã Hamming: bit kiểm tra ở vị trí `1, 2, 4, 8`; dữ liệu ở vị trí `3, 5, 6, 7, 9, 10, 11, 12`; `P0` là parity tổng.

| Vị trí | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Bit** | `P1` | `P2` | `D0` | `P4` | `D1` | `D2` | `D3` | `P8` | `D4` | `D5` | `D6` | `D7` |

* **Mã hóa:**
  * $P_1 = D_0 \oplus D_1 \oplus D_3 \oplus D_4 \oplus D_6$
  * $P_2 = D_0 \oplus D_2 \oplus D_3 \oplus D_5 \oplus D_6$
  * $P_4 = D_1 \oplus D_2 \oplus D_3 \oplus D_7$
  * $P_8 = D_4 \oplus D_5 \oplus D_6 \oplus D_7$
  * $P_0 = D_0 \oplus D_1 \oplus D_2 \oplus D_4 \oplus D_5 \oplus D_7$ *(XOR của cả 12 bit còn lại, đã rút gọn)*

* **Giải mã** *(dấu $'$ là giá trị nhận được)*:
  * $S_1 = P_1' \oplus D_0' \oplus D_1' \oplus D_3' \oplus D_4' \oplus D_6'$
  * $S_2 = P_2' \oplus D_0' \oplus D_2' \oplus D_3' \oplus D_5' \oplus D_6'$
  * $S_4 = P_4' \oplus D_1' \oplus D_2' \oplus D_3' \oplus D_7'$
  * $S_8 = P_8' \oplus D_4' \oplus D_5' \oplus D_6' \oplus D_7'$
  * $P_{ALL} = \text{XOR của cả 13 bit nhận được}$
  * $S = \{S_8, S_4, S_2, S_1\}$

| $S$ | $P_{ALL}$ | Kết luận | Hành động |
| :---: | :---: | :--- | :--- |
| `0` | `0` | Không lỗi | Giữ nguyên |
| $\neq 0$ (`1..12`) | `1` | Lỗi 1 bit tại vị trí $S$ | Lật bit đó, `SEC = 1` |
| `0` | `1` | Lỗi ở chính `P0` | Dữ liệu đúng, `SEC = 1` |
| $\neq 0$ | `0` | Lỗi 2 bit | `DED = 1`, không sửa |
| `13..15` | `1` | Từ 3 bit lỗi trở lên | `UE = 1` |

* **Sửa lỗi:** $D_{corr\_i} = D_i' \oplus (S == \text{vị trí của } D_i \text{ AND } P_{ALL})$, với vị trí `3, 5, 6, 7, 9, 10, 11, 12` ứng với `D0..D7`.
* **Ví dụ kiểm tra:** `D = 0x01` $\rightarrow$ `P1 = 1, P2 = 1, P4 = 0, P8 = 0, P0 = 1`.

### 2.6 CRC-4
Đa thức sinh: $g(x) = x^4 + x + 1$, giá trị khởi tạo `0`, tính song song trên 8 bit (`D7` là bit cao nhất).
* $C_3 = D_2 \oplus D_3 \oplus D_5 \oplus D_7$
* $C_2 = D_1 \oplus D_2 \oplus D_4 \oplus D_6 \oplus D_7$
* $C_1 = D_0 \oplus D_1 \oplus D_3 \oplus D_5 \oplus D_6 \oplus D_7$
* $C_0 = D_0 \oplus D_3 \oplus D_4 \oplus D_6$

* **Phía nhận:** Tính lại CRC trên $D_{corr}$, so với $C'$ nhận được; khác nhau thì `CRC_FAIL = 1`.
* **Ví dụ kiểm tra:** `D = 0x01` $\rightarrow$ `C = 0011`.

> *Lưu ý: Các phương trình ở mục 2.5 và 2.6 bắt buộc đối chiếu với golden model (mục 5) trước khi vẽ schematic.*

### 2.7 Cờ tổng hợp
$$\text{UE} = \text{DED} \mid \text{CRC\_FAIL} \mid (S \ge 13 \text{ AND } P_{ALL})$$
* **Hạn chế đã biết (cần ghi trong báo cáo):** Lỗi nằm ở chính các bit CRC sẽ làm `CRC_FAIL = 1` dù dữ liệu đúng.

### 2.8 Ước lượng quy mô (sơ bộ)

| Khối | Thành phần chính | Ước lượng |
| :--- | :--- | :--- |
| **ALU** | 8 full adder, 8 XOR đảo B, logic, 8 MUX 8:1, cờ | Khoảng 850 transistor |
| **Hamming encoder** | 19 XOR2 | Khoảng 230 transistor |
| **Hamming decoder + sửa** | Khoảng 38 XOR2, 8 cổng giải mã vị trí | Khoảng 550 transistor |
| **CRC (phát + kiểm)** | Khoảng 34 XOR2, 1 OR4 | Khoảng 420 transistor |
| **Error injector** | 17 XOR2 | Khoảng 200 transistor |
| **MUX chế độ, đệm** | Các cổng MUX và buffer | Khoảng 250 transistor |
| **Tổng cộng** | | **Khoảng 2.500 đến 3.000 transistor** |

---

## 3. Phân công 5 thành viên

| Thành viên | Vai trò chính | Sở hữu (chịu trách nhiệm đến LVS sạch) | Phần báo cáo |
| :--- | :--- | :--- | :--- |
| **TV1 (Nhóm trưởng)** | Kiến trúc, tích hợp, kiểm chứng | Spec, golden model, bộ vector, top-level schematic + layout, MUX chế độ | Chương 1, chương tích hợp, kết luận |
| **TV2** | Thư viện cell | INV, NAND, NOR, XOR/XNOR (3 kiểu), MUX2, BUF, DFF; quy ước layout cell; so sánh kiểu XOR | Chương thư viện cell |
| **TV3** | ALU | Full adder, bộ cộng 8 bit, khối logic, MUX 8:1, cờ, layout ALU | Chương ALU |
| **TV4** | Hamming SEC-DED | Encoder, cây syndrome, giải mã vị trí, mạch sửa, layout khối Hamming | Chương Hamming |
| **TV5** | CRC, bơm lỗi, tài liệu | CRC phát/kiểm, error injector, bản CRC nối tiếp (mở rộng), tổng hợp số liệu, slide, video | Chương CRC, chương kết quả so sánh |

### Quy tắc làm việc chung
* Mỗi cell/khối có đúng một người sở hữu. Người khác không sửa, chỉ báo lỗi cho chủ sở hữu.
* Mỗi khối phải có người thứ hai rà soát trước khi đưa vào top-level: `TV3` $\leftrightarrow$ `TV4`, `TV5` $\leftrightarrow$ `TV2`, `TV1` rà tất cả giao diện.
* Giai đoạn 2, cả 5 người đều tự vẽ layout ít nhất 1 cell để nắm quy trình DRC/LVS.
* Họp ngắn định kỳ: mỗi người báo đã xong gì, đang kẹt gì.

---

## 4. Tổ chức thư viện và file

### 4.1 Trong Cadence
* Một thư viện chung `EDAC_ALU`, gắn với PDK đã chốt.
* **Quy ước tên cell:**
  * `c_<tên>` cho cell cơ bản (`c_inv`, `c_nand2`, `c_xor2_cmos`, `c_xor2_tg`, `c_mux2`, `c_fa`, `c_dff`)
  * `b_<tên>` cho khối (`b_alu8`, `b_ham_enc`, `b_ham_dec`, `b_crc4`, `b_errinj`)
  * `top_edac_alu` cho top-level
  * `tb_<tên>` cho testbench
* Mỗi cell đủ các view: `schematic`, `symbol`, `layout`, và view trích xuất sau `PEX`.
* Tên chân thống nhất: bus viết `D<7:0>`, nguồn `VDD`, đất `VSS`.
* Nếu không có thư viện dùng chung trên máy chủ: mỗi người làm trong bản sao riêng, TV1 gom cell về thư viện chính sau khi cell qua rà soát.

### 4.2 Ngoài Cadence (Repo Git)

```text
edac-alu/
├── docs/            spec.md, kế hoạch, biên bản họp
├── model/           golden model Python
├── rtl/             mô hình Verilog tham chiếu + testbench
├── vectors/         file vector kích thích và kết quả mong đợi
├── scripts/         sinh stimulus Spectre, script OCEAN, xử lý kết quả
├── results/         CSV số liệu đo, hình dạng sóng, ảnh layout
└── report/          báo cáo, slide
```

---

## 5. Các giai đoạn thực hiện
*(Không gắn mốc ngày. Mỗi giai đoạn có điều kiện hoàn thành; chỉ chuyển sang giai đoạn sau khi đạt).*

### Giai đoạn 0 – Chuẩn bị và chốt spec
* [ ] Xem video hướng dẫn tiểu luận cuối kỳ, ghi lại yêu cầu và tiêu chí chấm *(TV5)*
* [ ] Chốt PDK, điện áp nguồn, công cụ DRC/LVS/PEX *(TV1, hỏi thầy nếu cần)*
* [ ] Xin thầy duyệt đề tài kèm sơ đồ khối mục 2.1 *(TV1)*
* [ ] Cả nhóm làm lại bài inverter của thầy trên PDK đã chốt: schematic, mô phỏng, layout, DRC, LVS *(Cả 5)*
* [ ] Tạo thư viện `EDAC_ALU`, repo Git, quy ước đặt tên *(TV1)*
* **Điều kiện hoàn thành:** Cả 5 người tự chạy được DRC/LVS sạch cho inverter; spec được thầy đồng ý.

### Giai đoạn 1 – Golden model và bộ vector *(TV1 chính, TV5 hỗ trợ)*
* [ ] Viết golden model Python cho ALU, Hamming, CRC, bơm lỗi, đúng theo mục 2
* [ ] Tự kiểm model: $256 \text{ dữ liệu} \times 13 \text{ vị trí lỗi đơn}$ đều sửa đúng; $256 \times 78 \text{ cặp lỗi đôi}$ đều báo `DED` và không sửa sai
* [ ] Thống kê lỗi 3 bit: tỉ lệ Hamming sửa sai, tỉ lệ CRC-4 bắt được *(số liệu này đưa vào báo cáo)*
* [ ] Viết mô hình Verilog tham chiếu + testbench, chạy bằng trình mô phỏng Verilog bất kỳ, đối chiếu với Python
* [ ] Sinh bộ vector cho từng khối và cho top: file kích thích và file kết quả mong đợi
* [ ] Viết script so khớp kết quả Spectre với kết quả mong đợi (lấy mẫu giữa mỗi chu kỳ vector, ngưỡng $V_{DD}/2$)
* **Điều kiện hoàn thành:** Python và Verilog cho kết quả giống nhau trên toàn bộ vector; phương trình mục 2.5, 2.6 đã được xác nhận.

### Giai đoạn 2 – Thư viện cell *(TV2 chính, cả nhóm tham gia)*

**Danh sách cell:**

| Cell | Ghi chú | Người vẽ layout |
| :--- | :--- | :---: |
| `c_inv`, `c_buf` | Xác định tỉ lệ $W_p/W_n$ tại đây | TV2 |
| `c_nand2`, `c_nor2` | | TV1 |
| `c_nand3`, `c_nand4` (hoặc `c_and4`) | Cho giải mã vị trí syndrome | TV4 |
| `c_xor2` (3 kiểu), `c_xnor2` | CMOS tĩnh dạng compound gate, transmission-gate, pass-transistor | TV2 |
| `c_mux2` | Kiểu transmission-gate | TV5 |
| `c_fa` | Full adder mirror 28T | TV3 |
| `c_dff` | Chỉ cần cho bản mở rộng có thanh ghi | TV2 |

**Việc cần làm cho mỗi cell:**
* [ ] Schematic và symbol
* [ ] Testbench: tải chuẩn FO4, sườn vào thực tế (qua 2 inverter đệm)
* [ ] Đo: $t_{pLH}$, $t_{pHL}$, thời gian lên/xuống, công suất trung bình, công suất tĩnh, tính PDP
* [ ] Layout theo quy ước chung (mục 5.5.1), DRC sạch, LVS khớp
* [ ] PEX và đo lại sau layout

> **Nghiên cứu chọn XOR (TV2):** So sánh 3 kiểu XOR về số transistor, diện tích, trễ, công suất, PDP, và mức suy giảm biên độ khi nối chuỗi 4 đến 6 tầng *(quan trọng vì cây XOR của Hamming sâu)*. Chốt 1 kiểu dùng cho toàn hệ thống, nêu lý do.

* **Điều kiện hoàn thành:** Mọi cell có đủ schematic, symbol, layout, DRC/LVS sạch, và một bảng đặc tính chung.

### Giai đoạn 3 – Thiết kế khối ở mức schematic *(TV3, TV4, TV5 song song)*

**ALU (TV3):**
* [ ] Bộ cộng/trừ 8 bit, khối logic, dịch, MUX 8:1, mạch cờ
* [ ] Mô phỏng đủ 8 lệnh với vector biên: `0x00`, `0xFF`, `0x80`, `0x7F`, tràn, mượn
* [ ] Đo trễ đường xấu nhất: `0xFF + 0x01` (carry chạy hết 8 bit)

**Hamming (TV4):**
* [ ] Encoder (19 XOR2), cân bằng cây XOR để giảm độ sâu
* [ ] Decoder: 4 cây syndrome, cây `PALL` (dùng lại nút trung gian nếu được), 8 cổng giải mã vị trí, 8 XOR sửa, logic SEC/DED
* [ ] Mô phỏng: không lỗi, lỗi đơn ở cả 13 vị trí, lỗi đôi

**CRC và bơm lỗi (TV5):**
* [ ] Khối CRC-4 song song (15 XOR2), dùng 2 bản: phát và kiểm; bộ so sánh 4 bit
* [ ] Error injector: 17 XOR2
* [ ] Mô phỏng cả 256 giá trị dữ liệu cho CRC

**MUX chế độ và giao diện (TV1):**
* [ ] `MUX_IN`, `MUX_OUT`, giải mã `MODE`, logic `UE`
* [ ] Soạn bảng giao diện chân giữa các khối, phát cho cả nhóm

* **Điều kiện hoàn thành:** Từng khối qua 100% vector của nó ở mức schematic, có số liệu trễ và công suất.

### Giai đoạn 4 – Tích hợp top-level và kiểm chứng *(TV1 chính, cả nhóm)*
* [ ] Ghép `top_edac_alu` từ symbol các khối
* [ ] Chạy 8 kịch bản kiểm chứng (mục 6)
* [ ] Đo đường tới hạn toàn hệ thống: $\text{ALU (carry dài nhất)} \rightarrow \text{encoder} \rightarrow \text{kênh} \rightarrow \text{decoder} \rightarrow \text{sửa} \rightarrow \text{MUX\_OUT}$
* [ ] Đo công suất trung bình với chuỗi vector ngẫu nhiên ở tần số cố định; đo công suất tĩnh
* [ ] Mô phỏng theo góc PVT nếu PDK có model: TT/SS/FF/SF/FS, $V_{DD} \pm 10\%$, nhiệt độ thấp/thường/cao
* **Điều kiện hoàn thành:** Top-level qua toàn bộ kịch bản ở góc TT, và chức năng đúng ở các góc còn lại.

### Giai đoạn 5 – Layout

#### 5.5.1 Quy ước layout cell *(TV2 soạn, cả nhóm tuân theo)*
* Chiều cao cell cố định, thanh `VDD` ở trên, `VSS` ở dưới, cùng bề rộng, để ghép cell bằng cách đặt sát nhau.
* N-well liên tục giữa các cell kề nhau; mỗi cell có tiếp xúc thân (body contact) cho cả NMOS và PMOS.
* Kim loại 1 dùng trong cell; kim loại 2 đi dọc, kim loại 3 đi ngang ở mức khối.
* Chân đặt trên lưới đi dây, có nhãn trùng tên với schematic.

#### 5.5.2 Layout khối
* [ ] Hamming encoder + decoder *(TV4)*, ưu tiên số 1 vì là phần trọng tâm
* [ ] CRC + error injector *(TV5)*
* [ ] ALU *(TV3)*, xếp theo kiểu bit-slice: 8 lát giống nhau xếp chồng
* [ ] MUX chế độ *(TV1)*
* [ ] Mỗi khối: DRC sạch, LVS khớp, đo diện tích

#### 5.5.3 Layout top-level *(TV1 chính, TV2 hỗ trợ)*
* [ ] Sơ đồ mặt bằng: đặt khối theo luồng dữ liệu để dây ngắn
* [ ] Lưới nguồn: vòng hoặc thanh `VDD`/`VSS` đủ rộng, nối tới mọi hàng cell
* [ ] Đi dây giữa các khối, đặt chân ra biên
* [ ] DRC, LVS toàn chip; kiểm tra antenna và mật độ kim loại nếu bộ luật của PDK có
* [ ] Xuất GDSII, chụp ảnh layout từng khối và toàn chip
* **Điều kiện hoàn thành:** Top-level DRC sạch, LVS khớp, có file GDSII.

### Giai đoạn 6 – Trích xuất ký sinh và mô phỏng sau layout
* [ ] PEX (RC) cho từng khối và top *(mỗi chủ khối tự làm)*
* [ ] Chạy lại bộ vector trên view đã trích xuất
* [ ] Lập bảng trước/sau layout: trễ, công suất cho từng khối và top
* [ ] Chạy góc PVT sau layout cho đường tới hạn
* [ ] Monte Carlo cho đường tới hạn nếu PDK có model thống kê *(TV2)*
* **Điều kiện hoàn thành:** Chức năng sau layout đúng 100% vector; có bảng so sánh trước/sau.

### Giai đoạn 7 – Phân tích so sánh *(TV5 tổng hợp, mỗi người cấp số liệu)*
* [ ] **Bảng A – Chi phí bảo vệ:** Đo số transistor, diện tích, trễ, công suất, PDP cho các cấu hình:

| Cấu hình | Khả năng |
| :--- | :--- |
| ALU trần | Không bảo vệ |
| ALU + parity 1 bit | Phát hiện lỗi lẻ |
| ALU + Hamming (12,8) | Sửa 1 bit |
| ALU + SEC-DED (13,8) | Sửa 1 bit, phát hiện 2 bit |
| ALU + SEC-DED + CRC-4 | Thêm kiểm tra đầu-cuối |

*(Ghi chú: Parity và Hamming (12,8) lấy được từ chính mạch SEC-DED bằng cách bỏ bớt phần, không phải thiết kế lại).*

* [ ] **Bảng B – So sánh kiểu XOR** *(từ giai đoạn 2)*
* [ ] **Bảng C – Trước và sau layout**
* [ ] **Bảng D – Độ phủ lỗi:** Lỗi 1 bit, 2 bit, 3 bit; tỉ lệ sửa đúng, phát hiện, bỏ sót *(số liệu từ golden model, đối chiếu mô phỏng)*
* [ ] **Bảng E – Góc PVT:** Trễ và công suất tốt nhất, điển hình, xấu nhất
* [ ] **Phân bổ diện tích và công suất theo khối** *(biểu đồ tròn hoặc cột)*

### Giai đoạn 8 – Báo cáo, slide, video
* [ ] Viết báo cáo theo mục 7 *(mỗi người viết chương của mình, TV5 biên tập, TV1 duyệt)*
* [ ] Slide thuyết trình, khoảng 15 đến 20 slide
* [ ] Quay video demo trên Cadence: kịch bản bơm lỗi 1 bit và mạch tự sửa, so với chế độ RAW
* [ ] Tập dượt thuyết trình, chuẩn bị câu hỏi phản biện (mục 9)
* [ ] Đóng gói nộp đúng định dạng thầy yêu cầu

---

## 6. Kịch bản kiểm chứng

| # | Kịch bản | Nội dung | Tiêu chí đạt |
| :---: | :--- | :--- | :--- |
| **1** | ALU cơ bản | 8 lệnh $\times$ vector biên + ngẫu nhiên, `MODE = 00` | `Y` và cờ khớp golden model |
| **2** | CRC | Cả 256 giá trị dữ liệu | CRC khớp |
| **3** | Không lỗi | `MODE = 01`, `ERR = 0` | `Y` = kết quả ALU, mọi cờ lỗi = 0 |
| **4** | Lỗi 1 bit | Lần lượt lật từng bit trong 13 bit Hamming | `Y` đúng, `SEC = 1`, `DED = 0` |
| **5** | Lỗi 2 bit | Các cặp bit trong 13 bit Hamming | `DED = 1`, `UE = 1`, không sửa sai |
| **6** | Lỗi 3 bit | Một số mẫu chọn từ thống kê | Ghi nhận trường hợp sửa sai, `CRC_FAIL` bắt được hay không |
| **7** | Lỗi ở bit CRC | Lật 1 bit trong `C[3:0]` | `Y` đúng, `CRC_FAIL = 1` *(hạn chế đã biết)* |
| **8** | So sánh RAW | Cùng mẫu lỗi, `MODE = 11` rồi `MODE = 01` | RAW cho kết quả sai, PROTECTED cho kết quả đúng |

* **Mức độ phủ:** Chạy toàn bộ tổ hợp ở mô hình Verilog/Python; ở mức transistor chạy đủ kịch bản 4 cho ít nhất vài giá trị dữ liệu $\times$ 13 vị trí, nhiều hơn nếu thời gian mô phỏng cho phép.

---

## 7. Cấu trúc báo cáo

1. **Tổng quan:** Lý do chọn đề tài, EDAC trong thực tế (ECC RAM, bus dữ liệu), mục tiêu, phạm vi, phân công.
2. **Cơ sở lý thuyết:** Mã CRC, mã Hamming, SEC-DED, khoảng cách Hamming, giới hạn sửa lỗi.
3. **Kiến trúc hệ thống:** Sơ đồ khối, chân tín hiệu, chế độ hoạt động, định dạng từ mã.
4. **Thư viện cell:** Schematic, kích thước transistor, kết quả đặc tính, so sánh kiểu XOR.
5. **Thiết kế các khối:** ALU, Hamming, CRC, bơm lỗi; schematic và mô phỏng từng khối.
6. **Tích hợp và kiểm chứng:** Top-level, 8 kịch bản, dạng sóng tiêu biểu.
7. **Layout:** Quy ước, layout cell, khối, top; kết quả DRC/LVS; diện tích.
8. **Kết quả sau layout và phân tích:** Các bảng A đến E, thảo luận.
9. **Kết luận và hướng phát triển.**
* **Phụ lục:** Bảng vector, script, ảnh báo cáo DRC/LVS, đóng góp từng thành viên.

---

## 8. Mở rộng (Làm sau khi xong phần chính)
*Xếp theo thứ tự ưu tiên:*
1. Thêm thanh ghi vào/ra (DFF): biến thành datapath đồng bộ, đo setup/hold và tần số tối đa.
2. Bộ cộng nhanh hơn: carry-lookahead hoặc carry-select, so với ripple-carry.
3. CRC nối tiếp bằng LFSR (4 DFF + XOR): so diện tích và thời gian tính với bản song song.
4. Pipeline 2 tầng ($\text{ALU} \mid \text{EDAC}$): so tần số tối đa.
5. Giảm công suất: tắt hoạt động khối EDAC khi `MODE = 00` (chặn tín hiệu vào bằng cổng AND).
6. Monte Carlo đầy đủ cho toàn hệ thống; phân tích sụt áp trên lưới nguồn bằng tính tay.
7. Vòng pad I/O nếu PDK có thư viện pad.

---

## 9. Rủi ro và câu hỏi phản biện

### 9.1 Quản lý rủi ro

| Rủi ro | Cách xử lý |
| :--- | :--- |
| Layout toàn hệ thống quá nặng | Ưu tiên: $\text{cell} \rightarrow \text{Hamming} \rightarrow \text{CRC} \rightarrow \text{ALU} \rightarrow \text{top}$. Dừng ở mức nào cũng có kết quả hoàn chỉnh để báo cáo |
| Đổi PDK giữa chừng | Chốt ở giai đoạn 0, không đổi |
| Mô phỏng transistor quá lâu | Kiểm chứng toàn bộ ở Verilog, mức transistor chạy tập con có chọn lọc |
| XOR pass-transistor suy giảm biên độ qua nhiều tầng | Đo ở giai đoạn 2, chèn đệm hoặc chọn kiểu khác |
| Lệch giao diện giữa các khối | Bảng giao diện do TV1 quản lý, rà soát chéo trước khi ghép |
| LVS top-level lỗi khó tìm | LVS sạch từng cấp từ dưới lên, không ghép khối chưa sạch |
| Một thành viên chậm tiến độ | Mỗi khối có người rà soát thứ hai, người đó nắm đủ để hỗ trợ |

### 9.2 Câu hỏi thầy có thể hỏi
* Vì sao cần cả CRC lẫn Hamming? *(Hamming sửa, CRC kiểm tra đầu-cuối bắt trường hợp sửa sai).*
* Lỗi 3 bit thì sao? *(Trả lời bằng số liệu bảng D).*
* Vì sao chọn CRC-4 và đa thức này? CRC-8 thì tốn thêm bao nhiêu?
* Vì sao chọn kiểu XOR này? *(Bảng B).*
* Đường tới hạn nằm ở đâu, cải thiện bằng cách nào?
* Chi phí của EDAC có đáng không? *(Bảng A).*
* Kết quả sau layout lệch bao nhiêu, vì sao?

---

## 10. Danh sách kiểm tra cuối cùng (Checklist)

* [ ] Spec được thầy duyệt
* [ ] Golden model và Verilog khớp nhau
* [ ] Mọi cell: schematic, symbol, layout, DRC/LVS sạch, có số liệu đặc tính
* [ ] Mọi khối: qua vector ở mức schematic và sau layout
* [ ] Top-level: 8 kịch bản đạt, DRC/LVS sạch, có GDSII
* [ ] Đủ bảng A đến E
* [ ] Báo cáo, slide, video demo
* [ ] Bảng đóng góp từng thành viên
* [ ] Nộp đúng định dạng yêu cầu