# EMBEDDED SYSTEM HARDWARE SPECIFICATION (AIoT NODE)
* **Project:** Smart Adaptive Irrigation Node (ESP32-S3 + On-Device TinyML LSTM)
* **Subsystem:** Hardware & Firmware Core (Member A)
* **Document Version:** v1.1 | Milestone: Week 1 Deliverable

---

## 1. Logic Levels & Voltage Domains

The system operates across two separate voltage domains decoupled via semiconductor switching:

### 1.1. Control Domain (3.3V Rail)
* **Nominal Supply Voltage ($V_{DD}$):** $3.3\text{V DC} \pm 5\%$ stepped down from the $5\text{V}$ bus via an MP1584EN buck converter[cite: 2].
* **ESP32-S3 3.3V CMOS Logic Thresholds:**
  * Input Low Voltage ($V_{IL}$): $-0.3\text{V} \le V_{IL} \le 0.825\text{V}$ (maximum $0.25 \times V_{DD}$).
  * Input High Voltage ($V_{IH}$): $2.475\text{V} \le V_{IH} \le 3.6\text{V}$ (minimum $0.75 \times V_{DD}$).
  * Output Low Voltage ($V_{OL}$): $\le 0.33\text{V}$ ($0.1 \times V_{DD}$) at nominal sink current $I_{OL} = 10\text{mA}$.
  * Output High Voltage ($V_{OH}$): $\ge 2.64\text{V}$ ($0.8 \times V_{DD}$) at nominal source current $I_{OH} = 10\text{mA}$.
* **Current Limitations:** Maximum $20\text{mA}$ per individual GPIO pin; total cumulative source/sink current across all I/O pins must not exceed $150\text{mA}$.

### 1.2. Actuator / Power Domain (5.0V Rail)
* **Actuator Supply Voltage ($V_{BUS}$):** $5.0\text{V DC} \pm 5\%$ sourced directly from an external $5\text{V}/2\text{A}$ USB adapter (bypassing the MCU buck converter to avoid brownout transients).
* **Gate Drive Configuration:** Direct $3.3\text{V}$ logic switching of a logic-level N-MOSFET (e.g., LR7843, AO3400, or IRLZ44N)[cite: 2]. Selected MOSFETs feature a gate threshold voltage $V_{GS(th)} \le 2.0\text{V}$, ensuring full channel saturation at $3.3\text{V}$.
* **Gate Pull-Down:** A $10\text{ k}\Omega$ resistor connected from Gate to GND to prevent floating states and spurious conduction during MCU boot/reset[cite: 2].

---

## 2. Analog-to-Digital Converter (ADC) Specification

Dedicated to acquiring analog moisture data from the Capacitive Soil Moisture Sensor v1.2[cite: 2].

* **Hardware Channel:** Assigned strictly to **ADC1_CH3 (GPIO 4)** to avoid co-existence lockouts with the ESP32-S3 Wi-Fi/BLE subsystem[cite: 2].
* **Resolution:** 12-bit ($0 - 4095$ quantization levels).
* **Attenuation Setting:** Configured to `ADC_ATTEN_DB_12` (effective measurable range: $0\text{V} - 3.1\text{V}$), fully spanning the sensor output profile of $1.2\text{V}$ (saturated wet) to $2.8\text{V}$ (bone dry)[cite: 2].
* **Hardware Calibration:** Utilizes factory characterization data stored in chip eFuse via `esp_adc_cal_raw_to_voltage()` to linearize readings near rail extremes.
* **Power Gating Mechanism:**
  * Sensor $V_{CC}$ is powered from **GPIO 5**[cite: 2].
  * Driven HIGH for only $20\text{ms}$ during active sampling[cite: 2], then pulled LOW immediately after ADC acquisition to prevent continuous standby draw and eliminate galvanic corrosion on the sensor pads[cite: 2].
* **Signal Conditioning:** Burst sampling with $N = 32$ iterations across a $10\text{ms}$ window; filtered using a 1D median filter to discard impulse noise, followed by a moving average filter before mapping to soil moisture percentage.

---

## 3. Bus Protocols & Hardware Interfaces

| Protocol / Interface | ESP32-S3 Pin | Connected Peripheral | Technical Parameters & Settings |
| :--- | :--- | :--- | :--- |
| **I2C Bus** | **SDA:** GPIO 8<br>**SCL:** GPIO 9[cite: 2] | - AHT20 (Ambient Temp & Humidity)[cite: 2]<br>- BH1750 (Irradiance / Lux)[cite: 2]<br>- *(Optional)* SSD1306 OLED | - Standard-mode ($100\text{ kHz}$) to prevent signal degradation over $20 - 30\text{cm}$ sensor leads.<br>- External $4.7\text{ k}\Omega$ pull-up resistors to the $3.3\text{V}$ rail[cite: 2].<br>- 7-bit Hardware Addresses:<br>  + AHT20: `0x38`[cite: 2]<br>  + BH1750: `0x23` (ADDR grounded)[cite: 2]<br>  + SSD1306: `0x3C` |
| **ADC1 Channel** | **Signal:** GPIO 4[cite: 2] | Capacitive Soil Sensor v1.2[cite: 2] | Analog input $0 - 3.0\text{V}$, high-impedance mode, decoupled with a $100\text{nF}$ ceramic capacitor. |
| **GPIO Output** | **Power Gate:** GPIO 5[cite: 2] | Soil Sensor $V_{CC}$[cite: 2] | Digital Push-Pull, sourcing up to $12\text{mA}$ during the active $20\text{ms}$ window[cite: 2]. |
| **GPIO Input** | **Float Switch:** GPIO 11[cite: 2] | Vertical PP Float Switch[cite: 2] | Internal pull-up enabled (`GPIO_PULLUP_ENABLE` $\approx 45\text{ k}\Omega$)[cite: 2]. Switch to GND:<br>- Full: Circuit closed to GND (Logic 0)[cite: 2].<br>- Empty: Circuit open, pulled to $3.3\text{V}$ (Logic 1) $\rightarrow$ Trigger pump lockout[cite: 2]. |
| **PWM (LEDC)** | **Pump PWM:** GPIO 6 | N-MOSFET Gate Driver Module | LEDC Channel 0, Timer 0:<br>- Carrier Frequency: $10\text{ kHz}$ (eliminates acoustic coil whine).<br>- Duty Cycle Resolution: 10-bit ($0 - 1023$ steps), providing micro-drip flow regulation between $30\%$ and $80\%$. |
| **GPIO Output** | **Status LEDs:**<br>- GPIO 7 (Green)<br>- GPIO 15 (Red) | 5mm Status Indicator LEDs | Push-pull outputs via $330\ \Omega$ series resistors ($I_F \approx 4.5\text{mA}$):<br>- Green: $1\text{Hz}$ blink during normal telemetry.<br>- Red: Solid state on reservoir empty or sensor disconnect. |
| **GPIO Output** | **Buzzer:** GPIO 16 | 5V Active Buzzer Module | Push-pull driving an audible $2.4\text{ kHz}$ intermittent alarm on failsafe states. |

---

## 4. Sampling Schedules & Timing Architecture
### 4.1. Operating Modes & Sampling Intervals
* **Normal Active Mode:**
  * Wake-up Interval: **Every 15 minutes**.
  * Sequence: Wake from Light-Sleep $\rightarrow$ Assert GPIO 5 HIGH ($20\text{ms}$ settling)[cite: 2] $\rightarrow$ Sample Soil ADC $\rightarrow$ Poll I2C sensors (AHT20 & BH1750)[cite: 2] $\rightarrow$ De-assert GPIO 5[cite: 2] $\rightarrow$ Push vector to RAM Ring Buffer $\rightarrow$ Run TinyML LSTM inference $\rightarrow$ Activate PWM pumping (if commanded by AI) $\rightarrow$ Return to Light-Sleep.
  * Active Time Window: $\approx 80\text{ms}$ (excluding pumping duration).
* **Night Mode:**
  * Auto-engaged when BH1750 reads $< 10\text{ Lux}$ continuously for 30 minutes.
  * Sampling Interval: **Every 60 minutes** (transpiration and soil dry-down are minimal without sunlight).
* **Vacation / Away Mode:**
  * Fixed **30-minute** interval with AI prioritizing early morning/late evening watering to maximize moisture retention and sustain the 5L reservoir for 20+ days.

### 4.2. In-Watering Safety Interlocking
* **Float Switch Polling Rate:** Polled every **$50\text{ms}$** while the pump is active. If water depletion is detected, the PWM output is immediately forced to $0\%$ within $100\text{ms}$.
* **Max Pumping Watchdog:** Hard cut-off limit of **60 seconds** continuous run per event to prevent waterlogging in case of line detachment.
* **Moisture Settling Delay (Soaking Delay):** Soil moisture ADC readings are suppressed for **5 minutes** after pump shut-off to allow root-zone percolation and prevent localized bias.

### 4.3. LSTM Input Tensor Formulation
* Every hour, the system generates one consolidated 4-variable feature vector:
