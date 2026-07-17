# BreathSafe

Trợ lý AI hỗ trợ nhân viên y tế tuyến xã sàng lọc sớm nguy cơ bệnh hô hấp.

> **Không phải thiết bị y tế.** Hệ thống chỉ hỗ trợ sàng lọc — không chẩn đoán, không kê đơn, không thay thế bác sĩ.

## Cài đặt (làm một lần)

```bash
pip install -r requirements.txt
python tao_du_lieu.py      # sinh 2.000 ca mô phỏng
python huan_luyen.py       # huấn luyện + đánh giá + lưu mô hình
```

## Các file trong thư mục này

| File | Việc nó làm | Độ khó |
|---|---|:---:|
| `dac_trung.py` | Danh sách 11 thông tin mà mô hình dùng | Dễ |
| `quy_tac.py` | **Rule Engine** — bộ quy tắc cảnh báo đỏ, viết bằng `if` | Dễ |
| `ca_kinh_dien.py` | 20 ca lâm sàng kinh điển để kiểm tra hệ thống | Dễ |
| `tao_du_lieu.py` | Sinh dữ liệu mô phỏng theo thang điểm nguy cơ | Vừa |
| `he_thong.py` | **Ghép 3 lớp an toàn** lại với nhau | Vừa |
| `huan_luyen.py` | Chia dữ liệu, train 5 mô hình, so sánh, đánh giá | Vừa |
| `app.py` | Giao diện Streamlit 7 trang | Vừa |
| `am_thanh.py` | Ghi âm + trích đặc trưng tiếng ho | Vừa |
| `huan_luyen_tieng_ho.py` | **Train mô hình phân loại tiếng ho** (module riêng) | Vừa |
| `chay_desktop.py` + `BreathSafe.bat` | Chạy app dạng cửa sổ desktop | Vừa |
| `retrain.py` | Huấn luyện lại từ phản hồi bác sĩ | Vừa |
| `tieng_viet.py` | Giúp cửa sổ lệnh Windows in được tiếng Việt | Dễ |

## Bảy trang của app

| Trang | Việc nó làm |
|---|---|
| Sàng lọc | Nhập một ca → mức nguy cơ + lý do + khuyến nghị. Có nút **tải phiếu kết quả** ra file HTML, mở bằng trình duyệt rồi Ctrl+P là in được hoặc lưu thành PDF. |
| Sàng lọc hàng loạt | Tải lên file CSV nhiều ca, chấm cả bộ một lượt, tải kết quả về. Nếu file có cột `expected` hoặc `risk_level` thì app tự chấm điểm và đếm **số ca Cao bị bỏ sót**. |
| Lịch sử ca | Mọi ca đã sàng lọc trong phiên, kèm biểu đồ phân bố mức nguy cơ và **tỉ lệ kết luận đến từ quy tắc so với từ AI**. Tải về CSV được. Chỉ nằm trong bộ nhớ — tắt app là mất. |
| Phản hồi bác sĩ | Ghi lại ý kiến bất đồng của nhân viên y tế để `retrain.py` học lại. |
| Ca lâm sàng | Bộ 20 ca kinh điển, kèm nút **chạy lại tại chỗ** bằng đúng mô hình app đang nạp. |
| Về hệ thống | Ba lớp an toàn, bốn cơ chế, bảng so sánh mô hình, reliability diagram. |
| Giới hạn | Những gì hệ thống không làm được — nêu thẳng. |

## Mô hình phân loại tiếng ho (module RIÊNG)

App có thể train một mô hình chỉ nghe tiếng ho để đoán "ho của người khỏe" hay
"ho bất thường". **Đây là một thí nghiệm đứng riêng, KHÔNG cộng vào mức nguy cơ
của hệ thống chính.** Khi có mô hình, app hiện kết quả của nó trong một ô riêng
có ghi rõ điều đó.

**Kiểm tra pipeline chạy đúng (không cần dữ liệu):**

```bash
python huan_luyen_tieng_ho.py --tu-kiem
```

**Train thật với COUGHVID:**

```bash
# 1. Cài ffmpeg (một lần) vì COUGHVID là file .webm:  winget install Gyan.FFmpeg
#    Cài xong PHẢI mở cửa sổ lệnh MỚI để nhận ffmpeg.
# 2. Tải + giải nén COUGHVID: đặt metadata_compiled.csv cạnh script,
#    file .webm vào thư mục du_lieu_ho/
python chuan_bi_coughvid.py       # tạo nhan_tieng_ho.csv từ nhãn chuyên gia
python huan_luyen_tieng_ho.py     # train + đánh giá + lưu mô hình
```

> Mức nguy cơ của hệ thống chính dựa trên **dấu hiệu lâm sàng** (SpO2, nhịp thở,
> triệu chứng), **không** dùng tiếng ho. Theo nhiều nghiên cứu lớn, sàng lọc
> bệnh qua tiếng ho hoạt động **kém** ngoài thực tế — nên báo cáo đúng con số
> AUC/ECE đo được (xem `ket_qua_tieng_ho.csv`), kể cả khi khiêm tốn.

## Ba lớp an toàn

```
Người bệnh đến trạm y tế
        ↓
[LỚP 1] Quy tắc cảnh báo đỏ ──trúng quy tắc──→ CAO ngay (không hỏi AI)
        ↓ (không trúng)
[LỚP 0] Kiểm tra ca lạ (OOD) ──ca quá lạ────→ Từ chối dự đoán,
        ↓                                       khuyến cáo chuyển tuyến
[LỚP 2] AI phân loại 3 mức (Random Forest đã hiệu chuẩn)
        ↓
[LỚP 3] Hậu kiểm ──AI nói Thấp mà có ≥2 dấu hiệu đáng ngờ, hoặc 1 chỉ số──→ nâng lên Trung bình
        │        đã bất thường (SpO2 < 92 / nhiệt độ > 39)
        ↓
Kết quả + lý do (2 cấp độ: nhân viên y tế / gia đình)
```

**Nguyên tắc cốt lõi:** AI không bao giờ được một mình kết luận rằng một ca là AN TOÀN.

## Kết quả (dữ liệu mô phỏng, tập test 300 ca)

> Các con số dưới đây được chép từ `ket_qua_so_sanh.csv` và
> `ket_qua_ca_kinh_dien_tong_hop.csv` — hai file do `python huan_luyen.py` sinh
> ra. Chạy lại lệnh đó thì phải cập nhật lại bảng này.

| Mô hình | Recall (Cao) | Bỏ sót (Cao) | F1 macro | ECE | Accuracy |
|---|---:|---:|---:|---:|---:|
| Chỉ Rule Engine | 0.850 | 0.150 | 0.768 | — | 0.770 |
| Logistic Regression | 0.817 | 0.183 | 0.808 | 0.029 | 0.823 |
| Decision Tree | 0.917 | 0.083 | 0.862 | 0.021 | 0.857 |
| Chỉ AI (RF + hiệu chuẩn) | 0.917 | 0.083 | **0.903** | **0.017** | **0.903** |
| **HỆ THỐNG ĐẦY ĐỦ** | **0.967** | **0.033** | 0.792 | — | 0.783 |

**Bộ 20 ca kinh điển:** chỉ AI đạt 15/20 → hệ thống đầy đủ đạt **18/20**. Rule Engine bắt được 3 ca mà AI bỏ sót.

Hai ca hệ thống vẫn bỏ sót là **TC16** (ngộ độc khí CO — máy đo SpO2 vẫn hiện 98%) và **TC18** (lao kê — sốt kéo dài nhưng dấu hiệu hô hấp nhẹ). Cả hai được phân tích trong phần Giới hạn.

### Cách đọc bảng này

**Vì sao hệ thống đầy đủ có Accuracy thấp hơn chỉ dùng AI?**
Vì Rule Engine cố tình đẩy nhiều ca lên mức Cao. Nó đánh đổi: bắt được nhiều ca nguy hiểm hơn (Recall 0.917 → 0.967) nhưng báo động giả nhiều hơn (Accuracy 0.903 → 0.783). Đây là **lựa chọn có chủ ý**, không phải lỗi. Bỏ sót một ca viêm phổi nặng có thể khiến một người chết; một báo động giả chỉ tốn của nhân viên y tế vài phút xem lại. Dự án tối ưu Recall, không tối ưu Accuracy.

**Vì sao cột ECE của Rule Engine và hệ thống đầy đủ để trống?**
Vì hai mô hình đó không đưa ra xác suất. Khi Rule Engine kết luận "SpO2 = 88% kèm khó thở là nguy hiểm", đó là một quyết định y khoa dứt khoát, không phải con số xác suất — nên không có gì để hiệu chuẩn.

## Giới hạn — nêu thẳng

1. **Dữ liệu là mô phỏng, không phải bệnh nhân thật.** Đây là giới hạn lớn nhất. Quy tắc sinh nhãn được viết ra, rồi mô hình học từ nhãn đó — nên điểm cao trên tập test chỉ chứng minh mô hình *học thuộc được quy tắc*, KHÔNG chứng minh nó đúng về y khoa. Vì vậy có hai kiểm chứng độc lập: bộ 20 ca kinh điển do bác sĩ duyệt, và dataset công khai bên ngoài.
2. **Máy đo SpO2 có thể đánh lừa hệ thống.** Người ngộ độc khí CO vẫn hiện SpO2 98% — hệ thống bỏ sót ca TC16.
3. **Phần "lý do" chưa phải giải thích thật của mô hình.** Random Forest có 200 cây, không thể nói chính xác vì sao nó chọn mức này cho ca này. Muốn làm đúng phải dùng SHAP — ngoài phạm vi dự án.
4. **Có thể có thiên lệch.** Người sống lâu năm ở vùng núi cao thường có SpO2 nền thấp hơn — hệ thống có thể báo động thừa với nhóm này.
5. **Ranh giới 3 mức nguy cơ được chọn** để tỉ lệ ra 50:30:20. Thực tế ranh giới là mờ, không sắc nét.
6. **n = 3 nhân viên y tế không đủ để kết luận thống kê.** Đây là đánh giá định tính khám phá.

## Công cụ & thư viện

Dự án sử dụng các thư viện bên thứ ba cài từ PyPI: **scikit-learn** (Random
Forest), **Streamlit** (giao diện), **librosa** (xử lý âm thanh), và
**`health-core`** (hiệu chuẩn xác suất, phát hiện ca lạ, cơ chế trọng số khi
retrain) — dùng như dùng `numpy` hay `pandas`.

Phần tự xây trong repo này: bộ quy tắc cảnh báo đỏ (`quy_tac.py`), kiến trúc 3
lớp an toàn (`he_thong.py`), bộ 20 ca lâm sàng kinh điển (`ca_kinh_dien.py`),
cách sinh dữ liệu (`tao_du_lieu.py`), và giao diện (`app.py`).
