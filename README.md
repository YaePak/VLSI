# Datapath ALU 8-bit Tích hợp Bộ Đồng Xử Lý EDAC (CRC-4 + Hamming SEC-DED)

Dự án thiết kế vi mạch số (VLSI) thực hiện theo quy trình **Full-Custom** từ mức transistor đến mức vật lý (GDSII) trên nền tảng công cụ **Cadence Virtuoso**. Hệ thống kết hợp khối tính toán số học/logic (ALU 8-bit) với cơ chế bảo vệ và tự phục hồi dữ liệu lai ghép (Hybrid Error Detection and Correction).

---

## 📌 Tổng quan kiến trúc

Hệ thống được thiết kế để đảm bảo tính toàn vẹn dữ liệu trong môi trường dễ phát sinh lỗi bit (nhiễu xuyên âm, bức xạ, sụt áp). Cơ chế bảo vệ sử dụng sự kết hợp 2 lớp:
1. **Hamming SEC-DED $(13,8)$:** Tự động sửa lỗi 1 bit (Single Error Correction) và phát hiện lỗi 2 bit (Double Error Detection).
2. **CRC-4 ($g(x) = x^4 + x + 1$):** Kiểm tra tính toàn vẹn đầu-cuối (end-to-end) sau khi giải mã Hamming, giúp phát hiện các trường hợp Hamming sửa sai khi xảy ra lỗi đa bit ($\ge 3$ bit).
3. **Error Injector:** Khối bơm lỗi chủ động trên kênh truyền 17-bit (`CW[16:0]`) phục vụ việc kiểm chứng và đánh giá độ tin cậy của mạch.

### Sơ đồ khối dữ liệu (Datapath)

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

---

## ⚙️ Đặc tả kỹ thuật chính

* **Quy mô thiết kế:** ~2,500 – 3,000 transistors (Full-custom).
* **Tập lệnh ALU (`OP[2:0]`):** `ADD`, `SUB`, `AND`, `OR`, `XOR`, `NOT`, `SHL`, `SHR` kèm đầy đủ 4 cờ trạng thái (`Z`, `N`, `C`, `V`).
* **Chế độ hoạt động (`MODE[1:0]`):**
  * `00` (**BYPASS**): Xuất trực tiếp kết quả từ ALU (dùng để đo baseline công suất/độ trễ).
  * `01` (**PROTECTED**): Chế độ hoạt động chính có mã hóa, truyền qua kênh lỗi và giải mã phục hồi.
  * `10` (**CODEC**): Đưa thẳng ngõ vào `A` qua bộ EDAC để kiểm thử độc lập khối mã hóa/giải mã.
  * `11` (**RAW**): Xuất dữ liệu thô nhận được từ kênh mà không qua sửa lỗi (dùng để đối chứng trực quan với chế độ `PROTECTED`).

---

## 🛠️ Quy trình thiết kế & Công cụ (Design Flow)

1. **Mô hình hóa & Kiểm chứng chức năng:** Python (Golden Model) & Verilog HDL.
2. **Thiết kế mạch nguyên lý (Schematic) & Thư viện Cell:** Cadence Virtuoso Schematic Editor (xây dựng thư viện cell chuẩn riêng `EDAC_ALU`).
3. **Mô phỏng mức Transistor (Pre-layout):** Cadence Spectre (đo độ trễ, công suất, PDP, phân tích góc PVT).
4. **Thiết kế vật lý (Layout):** Cadence Virtuoso Layout Suite (tuân thủ quy ước chuẩn hóa chiều cao cell, lưới nguồn/đất).
5. **Kiểm tra vật lý & Trích xuất ký sinh:** DRC, LVS và PEX (RC Extraction).
6. **Mô phỏng sau Layout (Post-layout):** Spectre & OCEAN scripts để đánh giá tác động của ký sinh lên đường tới hạn (Critical Path).

---

## 📂 Cấu trúc thư mục dự án

```text
edac-alu/
├── docs/            # Tài liệu đặc tả (Spec), kế hoạch thực hiện, biên bản họp
├── model/           # Golden model viết bằng Python (tính toán vector & thống kê lỗi)
├── rtl/             # Mã nguồn Verilog tham chiếu và Testbench kiểm chứng logic
├── vectors/         # Dữ liệu kích thích (Stimulus) và kết quả kỳ vọng (Expected outputs)
├── scripts/         # Script sinh stimulus cho Spectre, OCEAN script tự động hóa đo đạc
├── results/         # Dữ liệu thô (.csv), ảnh chụp dạng sóng (Waveform) và ảnh Layout
└── report/          # Báo cáo tổng kết cuối kỳ, slide thuyết trình và video demo
```

---

## 🎯 Mục tiêu đánh giá & Phân tích học thuật

Ngoài việc đảm bảo mạch hoạt động đúng chức năng và sạch lỗi DRC/LVS, dự án tập trung phân tích 4 khía cạnh chuyên sâu:
1. **Đánh đổi chi phí (PPA Overhead):** Mức gia tăng về Diện tích (Area), Công suất (Power) và Độ trễ (Delay) khi tích hợp EDAC so với ALU nguyên bản.
2. **Tối ưu hóa cổng logic:** So sánh các kiến trúc cổng XOR (Static CMOS, Transmission-Gate, Pass-Transistor) để tìm ra cấu trúc tối ưu nhất cho cây XOR sâu của Hamming và CRC.
3. **Phân cấp độ tin cậy:** Đánh giá chi phí tương ứng với từng cấp độ bảo vệ: $\text{Không bảo vệ} \rightarrow \text{Parity} \rightarrow \text{Hamming SEC} \rightarrow \text{SEC-DED} \rightarrow \text{SEC-DED + CRC-4}$.
4. **Tác động của ký sinh vật lý:** So sánh sự sai lệch hiệu năng giữa mô phỏng Pre-layout và Post-layout.

---

## 👥 Nhóm thực hiện

Dự án được thực hiện bởi nhóm 5 thành viên trong khuôn khổ môn học **Thiết kế vi mạch (VLSI)**:
* **TV1 (Nhóm trưởng):** Kiến trúc hệ thống, Golden Model, Tích hợp & Layout Top-level.
* **TV2:** Thiết kế và đặc tính hóa thư viện Standard Cell, nghiên cứu tối ưu cổng XOR.
* **TV3:** Thiết kế Schematic và Layout khối ALU 8-bit.
* **TV4:** Thiết kế Schematic và Layout khối Hamming SEC-DED $(13,8)$ Encoder/Decoder.
* **TV5:** Thiết kế khối CRC-4, Error Injector, tổng hợp số liệu và tài liệu báo cáo.