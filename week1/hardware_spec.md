# EMBEDDED SYSTEM HARDWARE SPECIFICATION (AIoT NODE)
* **Dự án:** Hệ thống tưới cây tự động thích ứng AIoT (ESP32-S3 + TinyML LSTM)
* **Phân hệ:** Hardware & Firmware Core (Thành viên A)
* **Phiên bản:** v1.1 | Milestone: Week 1 Deliverable

---

## 1. Mức logic và miền điện áp (Logic Levels & Voltage Domains)

Hệ thống phân tách thành hai miền điện áp độc lập:

### 1.1. Miền điện áp điều khiển (Control Domain - 3.3V)
* **Điện áp rail vi điều khiển ($V_{DD}$):** $3.3\text{V DC} \pm 5\%$ (hạ áp từ bus $5\text{V}$ qua module Buck MP1584EN).
* **Mức logic CMOS 3.3V của ESP32-S3:**
  * Điện áp ngõ vào mức thấp ($V_{IL}$): $-0.3\text{V} \le V_{IL} \le 0.825\text{V}$ (tối đa $0.25 \times V_{DD}$).
  * Điện áp ngõ vào mức cao ($V_{IH}$): $2.475\text{V} \le V_{IH} \le 3.6\text{V}$ (tối thiểu $0.75 \times V_{DD}$).
  * Điện áp ngõ ra mức thấp ($V_{OL}$): $\le 0.33\text{V}$ ($0.1 \times V_{DD}$) tại dòng tải danh định $I_{OL} = 10\text{mA}$.
  * Điện áp ngõ ra mức cao ($V_{OH}$): $\ge 2.64\text{V}$ ($0.8 \times V_{DD}$) tại dòng tải danh định $I_{OH} = 10\text{mA}$.
* **Giới hạn dòng GPIO:** Tối đa $20\text{mA}$ trên một chân; tổng dòng cấp/hút qua toàn bộ các chân I/O không vượt quá $150\text{mA}$.

### 1.2. Miền điện áp động lực (Power/Actuator Domain - 5.0V)
* **Điện áp nguồn bơm ($V_{BUS}$):** $5.0\text{V DC} \pm 5\%$ cấp trực tiếp từ củ sạc USB $5\text{V}-2\text{A}$ (tách riêng khỏi rail $3.3\text{V}$ của MCU để chống sụt áp).
* **Cơ chế kích đóng ngắt:** Kích mở chân Gate của N-MOSFET (LR7843 / AO3400) trực tiếp từ mức logic $3.3\text{V}$ của ESP32 ($V_{GS(th)} \le 2.0\text{V}$, mở bão hòa hoàn toàn ở $3.3\text{V}$). Điện trở xả $10\text{k}\Omega$ từ Gate xuống GND chống kích mở trôi khi khởi động.

---

## 2. Đặc tả khối biến đổi tương tự sang số (ADC Specification)

Sử dụng để đo độ ẩm đất từ cảm biến điện dung Capacitive Soil Moisture Sensor v1.2.

* **Kênh ADC phần cứng:** Bắt buộc sử dụng **ADC1_CH3 (GPIO 4)**. Không dùng các chân thuộc ADC2 vì bị vô hiệu hóa khi bật Wi-Fi/MQTT.
* **Độ phân giải (Resolution):** 12-bit ($0 - 4095$ mức lượng tử hóa).
* **Mức suy hao cấu hình (Attenuation):** Cấu hình `ADC_ATTEN_DB_12` (dải điện áp đo hiệu dụng: $0\text{V} - 3.1\text{V}$), bao phủ trọn vẹn dải ngõ ra của cảm biến đất ($1.2\text{V}$ lúc bão hòa nước đến $2.8\text{V}$ khi đất khô kiệt).
* **Hiệu chuẩn ADC (Calibration):** Sử dụng hàm `esp_adc_cal_raw_to_voltage()` kết hợp dữ liệu hiệu chuẩn lưu trong eFuse của chip để khắc phục vùng phi tuyến tính ở hai đầu dải đo.
* **Cơ chế Power Gating:** Chân $V_{CC}$ của cảm biến đất được nối vào **GPIO 5**. GPIO 5 chỉ kéo lên mức HIGH trong đúng $20\text{ms}$ để cấp điện đo ADC, đo xong kéo về LOW ngay để triệt tiêu hiện tượng ăn mòn điện phân trên bản mạch.
* **Lọc nhiễu tín hiệu:** Lấy mẫu liên tiếp $32$ lần trong khoảng thời gian $10\text{ms}$, áp dụng bộ lọc trung vị (Median Filter) loại bỏ mẫu dị biệt, sau đó tính trung bình để ra giá trị điện áp chuẩn trước khi ánh xạ sang $\%$ độ ẩm đất.

---

## 3. Chuẩn giao tiếp phần cứng (Bus Protocols & Interfaces)

* **Giao thức I2C (Inter-Integrated Circuit):**
  * Chân giao tiếp: **SDA** (GPIO 8), **SCL** (GPIO 9).
  * Tốc độ truyền (Bus Speed): Chuẩn Standard-mode $100\text{ kHz}$ (hạn chế suy hao khi kéo dây cảm biến dài $20 - 30\text{cm}$).
  * Điện trở kéo lên (Pull-up): Sử dụng 2 điện trở ngoài $4.7\text{k}\Omega$ kéo lên rail $3.3\text{V}$.
  * Thiết bị trên bus:
    * AHT20 (Nhiệt độ & Độ ẩm không khí): Địa chỉ 7-bit cố định `0x38`.
    * BH1750 (Cường độ ánh sáng Lux): Địa chỉ 7-bit `0x23` (chân ADDR nối GND).
* **Điều biến độ rộng xung PWM (Bơm nước):**
  * Chân phát xung: **GPIO 6**.
  * Cấu hình phần cứng: Module LEDC kênh 0, Timer 0.
  * Tần số sóng mang (Carrier Frequency): $10\text{ kHz}$ (loại bỏ tiếng rít tần số âm thanh của cuộn dây động cơ).
  * Độ phân giải Duty Cycle: 10-bit ($0 - 1023$ mức), điều chỉnh lưu lượng tưới nhỏ giọt mịn từ $30\%$ đến $80\%$.
* **Giao tiếp Digital Input / Output:**
  * Phao từ mực nước (**GPIO 11**): Cấu hình ngõ vào có trở kéo nội `GPIO_PULLUP_ENABLE` ($\approx 45\text{k}\Omega$). Phao nối chân còn lại xuống GND (Logic 0 = Đầy nước, Logic 1 = Cạn nước).
  * LED trạng thái (**GPIO 7** - Xanh, **GPIO 15** - Đỏ): Push-pull ngõ ra, qua trở hạn dòng $330\Omega$ ($I_F \approx 4.5\text{mA}$).
  * Còi báo Active Buzzer (**GPIO 16**): Push-pull kích còi phát âm báo $2.4\text{ kHz}$ khi có sự cố.

---

## 4. Chu kỳ lấy mẫu và định thời hệ thống (Sampling Schedules & Timing)

### 4.1. Chu kỳ vận hành hệ thống
* **Chế độ Hoạt động bình thường (Normal Mode):**
  * Chu kỳ thức: **15 phút / lần**.
  * Trình tự thực thi: Đánh thức từ Light-Sleep $\rightarrow$ Bật GPIO 5 ($20\text{ms}$) $\rightarrow$ Lấy mẫu ADC đất $\rightarrow$ Đọc I2C AHT20 & BH1750 $\rightarrow$ Tắt GPIO 5 $\rightarrow$ Đưa mẫu vào Ring Buffer RAM $\rightarrow$ Chạy suy luận mạng LSTM $\rightarrow$ Kích hoạt bơm (nếu AI yêu cầu) $\rightarrow$ Quay lại Light-Sleep.
  * Tổng thời gian thức (Active Window): $\approx 80\text{ms}$ (nếu không kích hoạt bơm).
* **Chế độ Ban đêm (Night Mode):**
  * Tự kích hoạt khi BH1750 đo ánh sáng $< 10\text{ Lux}$ liên tiếp trong 30 phút.
  * Chu kỳ lấy mẫu dãn ra **60 phút / lần** để tiết kiệm điện và giảm hao mòn cơ khí.
* **Chế độ Vắng nhà (Vacation Mode):**
  * Chu kỳ lấy mẫu cố định **30 phút / lần**, ưu tiên tưới vào sáng sớm hoặc chiều mát dựa theo dự báo của mô hình AI.

### 4.2. Vòng lặp an toàn khi tưới (Watering Safety & Interlocking)
* **Chu kỳ quét phao nước:** Quét mức logic GPIO 11 liên tục mỗi **$50\text{ms}$** trong suốt thời gian bơm chạy. Nếu phát hiện cạn nước, ngắt PWM về $0\%$ ngay lập tức.
* **Khóa thời gian bơm tối đa (Watchdog):** Bơm không được phép chạy liên tục quá **60 giây** trong một lần tưới nhằm ngăn ngập úng nếu tuột ống dẫn.
* **Thời gian thẩm thấu (Soaking Delay):** Sau khi dừng bơm, khóa đọc cảm biến đất trong **5 phút** để nước ngấm đều vào rễ trước khi ghi nhận giá trị ẩm mới.

### 4.3. Cấu trúc dữ liệu đầu vào cho AI (LSTM Input Tensor)
* Mỗi giờ hệ thống tổng hợp 1 vector dữ liệu gồm 4 thuộc tính đã chuẩn hóa: $[T_{\text{air}},\ RH_{\text{air}},\ \text{Lux},\ \text{Moisture}_{\text{soil}}]$.
* Cửa sổ trượt (Time Window): $24$ bước thời gian (24 giờ gần nhất), lưu trữ trong mảng bộ nhớ đệm `float input_buffer[24][4]` trên RAM để nạp vào Tensor Arena khi gọi hàm `invoke()`.
