# DIY Breadboard Guide: ESP32-S3 XiaoZhi AI Voice Assistant

This guide provides step-by-step instructions to build the **Compact Breadboard (`bread-compact-wifi`)** edition of the XiaoZhi AI Voice Assistant using an **ESP32-S3**, an **INMP441** I2S microphone, a **MAX98357A** I2S audio amplifier, and an **SSD1306/SH1106** I2C OLED display.

---

## 1. Hardware Bill of Materials (BOM)

| Component | Specification / Model | Purpose | Quantity |
| :--- | :--- | :--- | :--- |
| **Microcontroller** | ESP32-S3-DevKitC-1 (N8R8 or N16R8, 8MB/16MB Flash, 8MB PSRAM recommended) | Main controller running XiaoZhi firmware | 1 |
| **I2S Microphone** | INMP441 (or ICS-43434) MEMS module | High-fidelity voice input / wake word detection | 1 |
| **I2S DAC / Amplifier** | MAX98357A (Class D 3.2W mono amp) | Audio decoding and speaker output | 1 |
| **Speaker** | 4Ω or 8Ω, 2W–3W (40mm / 31mm miniature speaker) | Voice playback | 1 |
| **Display** | 0.96" or 0.91" I2C OLED (SSD1306 128x64 or 128x32, or SH1106) | Shows status, Banglish/Bengali text, and face emotes | 1 |
| **Tactile Buttons** | 6x6mm momentary push buttons (or TTP223 capacitive touch module) | Push-to-talk, Volume Up, Volume Down | 3 |
| **Breadboard & Wires** | Standard 830-point solderless breadboard + Dupont jumper wires | Circuit prototyping | 1 kit |
| **Power Supply** | USB-C 5V 2A power adapter / reliable USB port | Power source | 1 |
| *(Optional)* **Lamp / Relay** | 5V Relay module or 5mm LED + 330Ω resistor | MCP smart home test (`LAMP_GPIO`) | 1 |

---

## 2. Complete Pinout & Wiring Table

The pinout corresponds directly to [`main/boards/bread-compact-wifi/config.h`](../main/boards/bread-compact-wifi/config.h).

### A. I2S Microphone (INMP441)
> [!NOTE]
> The firmware uses **Simplex I2S** mode, giving the microphone and speaker dedicated clocks to prevent sample rate collisions.

| INMP441 Pin | ESP32-S3 Pin | Function / Description |
| :--- | :--- | :--- |
| **VDD** | `3.3V` | Power supply (3.3V only, do NOT connect to 5V) |
| **GND** | `GND` | Ground |
| **SD** | `GPIO 6` | Serial Data out (`AUDIO_I2S_MIC_GPIO_DIN`) |
| **WS** | `GPIO 4` | Word Select / Left-Right Clock (`AUDIO_I2S_MIC_GPIO_WS`) |
| **SCK** | `GPIO 5` | Continuous Serial Clock (`AUDIO_I2S_MIC_GPIO_SCK`) |
| **L/R** | `GND` | Channel select: connect to GND for Left channel |

---

### B. I2S Audio Amplifier & DAC (MAX98357A)

| MAX98357A Pin | ESP32-S3 Pin | Function / Description |
| :--- | :--- | :--- |
| **VIN** | `5V` (or `3.3V`) | Power supply (**5V** from USB pin is recommended for best audio volume without brownouts) |
| **GND** | `GND` | Ground (must share common GND with ESP32) |
| **DIN** | `GPIO 7` | Data Input (`AUDIO_I2S_SPK_GPIO_DOUT`) |
| **BCLK** | `GPIO 15` | Bit Clock (`AUDIO_I2S_SPK_GPIO_BCLK`) |
| **LRC** | `GPIO 16` | Left/Right Clock (`AUDIO_I2S_SPK_GPIO_LRCK`) |
| **GAIN** | `GND` / Unconnected | Connect to GND for 9dB gain (or leave floating for 12dB gain) |
| **SD_MODE** | Unconnected / `100kΩ to VDD` | Mixes (L + R) / 2 to mono output by default |
| **+/- Speaker Terminals** | Speaker leads | Connect to 4Ω/8Ω speaker terminals |

---

### C. I2C OLED Display (SSD1306 / SH1106)

| OLED Pin | ESP32-S3 Pin | Function / Description |
| :--- | :--- | :--- |
| **VCC** | `3.3V` | 3.3V Power |
| **GND** | `GND` | Common Ground |
| **SCL** | `GPIO 42` | I2C Clock (`DISPLAY_SCL_PIN`) |
| **SDA** | `GPIO 41` | I2C Data (`DISPLAY_SDA_PIN`) |

*Default I2C address: `0x3C`.*

---

### D. Buttons & Interactions

All buttons use internal pull-ups; one side connects to the GPIO and the other side connects to **GND**.

| Button | GPIO | Default Action | Alternate Action |
| :--- | :--- | :--- | :--- |
| **Boot Button** | `GPIO 0` (Onboard) | Single click: Toggle chat state | Click during startup: Enter Wi-Fi config mode |
| **Touch / PTT Button** | `GPIO 47` | Press down: Start listening | Release: Stop listening & process reply |
| **Volume Up** | `GPIO 40` | Single click: Volume +10% | Long press: Max volume (100%) |
| **Volume Down** | `GPIO 39` | Single click: Volume -10% | Long press: Mute (0%) |

---

### E. Status LED & MCP Test Actuator

| Component | GPIO | Description |
| :--- | :--- | :--- |
| **Built-in LED** | `GPIO 48` | System state indicator LED (`BUILTIN_LED_GPIO`) |
| **MCP Lamp / Relay** | `GPIO 18` | Actuator controlled by AI via MCP tool `LAMP_GPIO` |

---

## 3. Wiring Diagram Overview

```
                      +-------------------+
                      |   ESP32-S3 DevKit |
                      +-------------------+
                      |  3V3          GND |----> Common Ground Rail
  OLED VCC <----------|  3V3          5V  |----> MAX98357A VIN
  INMP441 VDD <-------|  3V3              |
                      |                   |
  OLED SCL <----------|  GPIO 42          |
  OLED SDA <----------|  GPIO 41          |
                      |                   |
  INMP441 WS <--------|  GPIO 4   GPIO 16 |---> MAX98357A LRC
  INMP441 SCK <-------|  GPIO 5   GPIO 15 |---> MAX98357A BCLK
  INMP441 SD <--------|  GPIO 6   GPIO 7  |---> MAX98357A DIN
                      |                   |
  Vol- Button <-------|  GPIO 39  GPIO 18 |---> (Optional) MCP Lamp Relay
  Vol+ Button <-------|  GPIO 40  GPIO 48 |---> Builtin LED
  PTT / Touch <-------|  GPIO 47          |
  Boot Button <-------|  GPIO 0           |
                      +-------------------+
```

---

## 4. Audio Quality & Noise Reduction Tips

1. **Clean Power Routing**:
   - The MAX98357A draws high peak currents when producing loud speech. Power the amplifier module directly from the `5V` (USB) pin rather than the regulated `3.3V` pin.
   - Place a `100µF` to `470µF` electrolytic capacitor across `5V` and `GND` near the amplifier to eliminate clicking or popping sounds.
2. **Microphone Isolation**:
   - Keep the INMP441 microphone physically separated from the speaker cone to prevent acoustic feedback and wake word false triggers.
   - Connect the INMP441 `L/R` pin directly to `GND`.
3. **Keep Wire Runs Short**:
   - High-speed I2S clock lines (`GPIO 5`, `GPIO 15`, `GPIO 16`) should be kept as short as possible to avoid radio frequency interference on the breadboard.

---

## 5. Firmware Build & Flashing

### Prerequisites
- ESP-IDF v5.4 or higher installed and configured in your environment.
- Python 3.10+.

### Step 1: Clone and Set Up
```bash
git clone https://github.com/Eftyqhar/Esp32s3-xiaozhi-ai-voice-assistant.git
cd Esp32s3-xiaozhi-ai-voice-assistant
```

### Step 2: Build the Breadboard Target

To build for an **SSD1306 128x64** OLED:
```bash
python scripts/release.py --build bread-compact-wifi-128x64
```

To build for an **SSD1306 128x32** OLED:
```bash
python scripts/release.py --build bread-compact-wifi
```

Alternatively, build using standard ESP-IDF commands:
```bash
idf.py set-target esp32s3
idf.py build
```

### Step 3: Flash to ESP32-S3
Connect your ESP32-S3 via the **UART / USB** port and run:
```bash
idf.py -p COMx flash monitor
# (Replace COMx with your port, e.g. COM3 or /dev/ttyUSB0)
```

---

## 6. First Boot & Wi-Fi Configuration

1. **Power On**: On first boot, the OLED display will initialize and show the startup animation.
2. **Wi-Fi Configuration**:
   - If no network is saved, click the **Boot Button** (`GPIO 0`) at startup to enter Wi-Fi Config Mode.
   - Connect your phone or PC to the Wi-Fi AP broadcasted by the ESP32 (e.g. `XiaoZhi-xxxx`).
   - Open your browser at `http://192.168.4.1` and select your home Wi-Fi SSID and password.
3. **Connecting to Server**:
   - Once connected to Wi-Fi, the device registers with the XiaoZhi cloud backend (`xiaozhi.me` or your self-hosted WebSocket server).
   - An activation code or status will appear on the OLED display.

---

## 7. Bengali Localization & Banglish Display

This build includes support for **Bengali (bn-BD)**:
- System messages and prompts are localized in Bengali (`main/assets/locales/bn-BD/language.json`).
- Because monochrome OLED displays have limited memory and font rendering constraints for complex conjuncts (যুক্তবর্ণ), incoming Bengali responses are automatically transliterated into readable phonetic **Banglish** in real-time via the built-in `ToBanglish()` engine in `display.cc`.

---

## 8. Testing MCP Smart Home Control

The firmware comes with a preconfigured Model Context Protocol (MCP) actuator tool on **`GPIO 18`**:
- Say to the assistant: *"Turn on the lamp"* or *"Turn off the light"*.
- The LLM will trigger the local MCP tool `self.lamp.set_state`, which toggles `GPIO 18` HIGH or LOW automatically.
