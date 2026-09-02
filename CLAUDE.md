# Dự án: Lõi IP Ascon-AEAD128 tích hợp AMBA APB

Đồ án tốt nghiệp ngành thiết kế vi mạch, UIT.
Hướng FPGA kết hợp kiểm chứng chức năng.

Môi trường: Windows, thư mục dự án `E:\Project_FPGA\DACN`.
Công cụ: Icarus Verilog (mô phỏng), Vivado (tổng hợp), Python 3 (mô hình tham chiếu).

---

## Ràng buộc tuyệt đối

### 1. Đúng biến thể thuật toán

Thuật toán là **Ascon-AEAD128 theo NIST SP 800-232**:

- rate 128 bit, capacity 192 bit
- hoán vị khởi tạo và hoàn tất: p12
- hoán vị trung gian: **p8**
- biểu diễn **little-endian**
- IV = `0x00001000808c0001`

**KHÔNG PHẢI** Ascon-128 bản v1.2 (rate 64, p6, big-endian, IV khác).
Hai thuật toán cho ra bản mã khác nhau hoàn toàn.

### 2. Mô hình tham chiếu là bất khả xâm phạm

`model/ascon_model.py` là golden reference model. Nó được viết từ đặc tả
và kiểm chứng bằng bộ test vector chính thức của NIST.

- **Không bao giờ sửa model để khớp với RTL.**
- Khi RTL và model lệch nhau, mặc định RTL sai.
- Chỉ sửa model nếu chính nó fail test vector NIST.

### 3. Kiểm chứng trước khi commit

Mọi thay đổi trong `rtl/` phải chạy lại toàn bộ testbench và pass
trước khi commit.

### 4. Ngôn ngữ mô tả phần cứng

- `rtl/` dùng **Verilog-2001** thuần. Không dùng SystemVerilog.
- `tb/` được phép dùng SystemVerilog.

### 5. Quy tắc viết RTL

- Trong `always @(*)` luôn gán giá trị mặc định ở đầu khối để tránh sinh latch.
- `always @(*)` dùng phép gán blocking `=`.
- `always @(posedge clk)` dùng phép gán non-blocking `<=`.
- Reset bất đồng bộ, tích cực mức thấp, đặt tên `rst_n` hoặc `presetn`.
- Không dùng mảng lớn cho bảng hằng số — dùng `case` để tổng hợp ra logic,
  tránh công cụ suy ra Block RAM.
- Thiết kế không được suy ra BRAM hay DSP. Nếu báo cáo tổng hợp có,
  đó là lỗi cần sửa.

### 6. Testbench phải tự phán đúng sai

Testbench in ra PASS hoặc FAIL, không bao giờ bắt người đọc waveform
để kết luận. Dùng `!==` chứ không phải `!=` khi so sánh, để bắt được
giá trị `x`.

---

## Cấu trúc thư mục

```
docs/spec.md        đặc tả chốt, register map — ĐỌC TRƯỚC khi sửa RTL
docs/uarch.md       sơ đồ khối, FSM, ngân sách chu kỳ
docs/test_plan.md   bảng kế hoạch kiểm chứng
docs/BUGS.md        nhật ký lỗi, ghi mỗi khi tìm ra bug
model/              mô hình tham chiếu Python
rtl/core/           ascon_sbox, ascon_linear, ascon_round, ascon_perm, ascon_aead_fsm
rtl/ip/             ascon_apb.v — đơn vị tổng hợp
tb/unit/            test từng module
tb/directed/        test vector NIST
tb/sva/             assertion
constraints/        file XDC
scripts/            script Tcl và shell
reports/            báo cáo do công cụ sinh ra
vectors/            file KAT của NIST
```

**Quy tắc phụ thuộc một chiều:** `rtl/core/` không tham chiếu gì từ
`rtl/ip/`, và cả hai không tham chiếu gì từ phần demo trên board.

---

## Lệnh hay dùng

```
make model     chạy mô hình Python với test vector NIST
make unit      chạy test từng module
make kat       chạy test vector qua RTL
make regress   chạy toàn bộ testbench
make synth     tổng hợp Out-of-Context bằng Vivado
```

---

## Thứ tự làm việc

Đây là quy trình chuẩn, mỗi bước có điều kiện thoát. Không đi tiếp
khi chưa đạt.

| Bước | Nội dung | Điều kiện thoát |
|---|---|---|
| 1 | Đặc tả và register map | Có `docs/spec.md` đầy đủ |
| 2 | Mô hình Python | Pass 100 % test vector NIST |
| 3 | Vi kiến trúc | Có sơ đồ FSM và datapath |
| 4 | Viết RTL từ dưới lên | Không latch, mỗi module khớp Python |
| 5 | Kiểm chứng | Regression pass hết |
| 6 | Tổng hợp OOC | Không có BRAM, DSP; tài nguyên hợp lý |
| 7 | Implement, quét Fmax | WNS ≥ 0 |
| 8 | Gate-level sim, đo công suất | Pass test vector trên netlist |
| 9 | Khảo sát kiến trúc thứ hai | Có bảng PPA và biểu đồ Pareto |

Xây RTL **từ dưới lên**, mỗi tầng test xong mới lên tầng trên:

```
ascon_sbox + ascon_linear  →  ascon_round  →  ascon_perm
  →  ascon_aead_fsm  →  ascon_apb
```

---

## Quy ước

- Giải thích và thảo luận bằng **tiếng Việt**.
- Commit message và comment trong code bằng **tiếng Anh**, ngắn gọn.
- Tên tín hiệu `snake_case`, tham số `UPPER_CASE`.
- Sau khi viết testbench, **tự chạy iverilog và sửa cho tới khi pass**,
  đừng chỉ viết code rồi dừng.
- Khi tìm ra bug, ghi vào `docs/BUGS.md` với đủ: triệu chứng, cách phát hiện,
  nguyên nhân gốc, cách sửa, bài học.

---

## Những chỗ hay sai — kiểm trước khi báo lỗi

Xếp theo tần suất. Khi test vector fail, dò lần lượt từ trên xuống.

1. Nhầm Ascon-128 v1.2 với Ascon-AEAD128 (rate 64 vs 128, p6 vs p8)
2. Thứ tự byte — nạp byte vào từ 64 bit sai thứ tự little-endian
3. Quên khối đệm phụ khi dữ liệu dài đúng bội số 16 byte
4. Chạy p8 sau khối bản rõ cuối cùng (không được chạy)
5. Quên XOR bit phân tách miền khi AD rỗng (vẫn phải XOR)
6. Vẫn xử lý giai đoạn AD khi AD rỗng (phải bỏ qua hoàn toàn)
7. Sai vị trí XOR khóa: khởi tạo vào S3/S4, hoàn tất vào S2/S3
8. Sai khối cuối lẻ byte khi giải mã
9. Xoay trái thay vì xoay phải trong lớp tuyến tính
10. Bảng hằng số lệch index — p12 bắt đầu ở 4, p8 bắt đầu ở 8

**Chiến lược debug:** đi từ dưới lên. Kiểm một vòng hoán vị trước, rồi p12
từ trạng thái toàn 0, rồi chỉ giai đoạn khởi tạo, rồi mới tới thông điệp
đầy đủ. Đừng debug thẳng ở mức test vector.
