# EMBEDDED SYSTEM HARDWARE SPECIFICATION (AIoT NODE)
* Project: Smart Adaptive Irrigation Node (ESP32-S3 + LSTM)
* Role: Member A (Hardware & Firmware Core)
* Milestone: Week 1 Deliverable

---

## 1. Power Budget & Supply Architecture
* **Primary Power Source:** 2S Li-ion Battery pack (2x 18650, 7.4V nominal, 8.4V max, 2500mAh) or external 9V-12V/2A DC Adapter.
* **Step-Down Regulation:** DC-DC Buck Converter MP1584EN (Adjustable to 3.3V, switching frequency 1.5MHz, up to 3A peak / 1.5A continuous, efficiency ~92%).
* **Load Current Profile:**
  - ESP32-S3 Peak: 350mA (Wi-Fi TX burst + dual-core active).
  - ESP32-S3 Sleep: 15mA (Light-sleep) / <50uA (Deep-sleep).
  - R385 Motor Pump: Nominal 0.5A, Inrush peak current up to 1.8A (7.4V).
  - Sensor Suite: AHT20 (~1mA), BH1750 (~0.2mA), Capacitive Soil Sensor (~5mA).
* **Anti-Brownout & Noise Filtering:**
  - High-side isolation Diode (SS34) separating motor power rail from logic buck input.
  - 1000uF/16V Low-ESR electrolytic capacitor in parallel with 100nF ceramic cap at MP1584 input terminal to absorb motor inrush inductive dips.
  - Trim potentiometer on MP1584 calibrated strictly to 3.30V before connecting to ESP32-S3.

---

## 2. Sensor Interfacing & Power Gating
* **Capacitive Soil Moisture Sensor v1.2:**
  - Operating Voltage: 3.3V.
  - Analog Out: 1.2V (Saturated wet) to 2.8V (Bone dry).
  - Assigned Pin: GPIO 4 (Dedicated to internal ADC1_CH3 to prevent Wi-Fi coexistence conflicts).
  - **Power Gating:** VCC tied to GPIO 5. Sensor is powered only for 20ms during sampling cycles to eliminate continuous 5mA drain.
* **Micro-Climate Monitoring (I2C Bus):**
  - AHT20: Ambient Temperature & Relative Humidity (I2C Addr: `0x38`).
  - BH1750: Ambient Solar Irradiance (I2C Addr: `0x23`).
  - Bus Pull-ups: Dual 4.7kOhm external resistors pulled to 3.3V.
* **Reservoir Safety:**
  - Mini Vertical PP Float Switch connected to GPIO 11 with internal pull-up.
  - Trigger Logic: HIGH = Normal water level; LOW = Low water alert (Pump locked).

---

## 3. Actuator Driver & Inductive Protection
* **Switching Element:** N-Channel Logic-Level MOSFET (IRLZ44N or AO3400).
* **Isolation:** PC817 Optocoupler isolating ESP32-S3 GPIO from inductive switching noise and ground bounce.
* **Flyback Protection:** Fast recovery Schottky Diode SS34 mounted antiparallel across motor terminals to suppress reverse EMF ($V = -L \frac{di}{dt}$).
* **Gate Discharge:** 10kOhm resistor from Gate to GND preventing spurious turn-on during MCU boot.

---

## 4. ESP32-S3 Pinout Allocation Map
| Pin | Label | Type | Electrical Interface | Target Device |
|:---|:---|:---|:---|:---|
| GPIO 4 | SOIL_ADC | Analog In | ADC1_CH3 | Soil Moisture Analog Out |
| GPIO 5 | SOIL_PWR | Digital Out | Push-Pull | Sensor VCC Power Gate |
| GPIO 8 | I2C_SDA | Open-Drain | External 4.7k Pull-up | AHT20 & BH1750 SDA |
| GPIO 9 | I2C_SCL | Open-Drain | External 4.7k Pull-up | AHT20 & BH1750 SCL |
| GPIO 10| PUMP_EN | Digital Out | Via 220R to PC817 Anode | Motor Driver Trigger |
| GPIO 11| FLOAT_SW | Digital In | Internal Pull-up | Reservoir Float Switch |
| GPIO 12| LED_FAIL | Digital Out | Via 330R to Red LED | Fault / Failsafe Status |
| GPIO 13| LED_AWAY | Digital Out | Via 330R to Green LED | Away Mode Indicator |