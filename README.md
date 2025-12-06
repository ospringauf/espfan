# ESP Fan Controller

ESPHome-based fan controller for two 12V PWM fans integrated with Home Assistant.

!!! Disclaimer: this is work in progress, untested, mostly AI generated !!!

## Hardware

- ESP32 Dev Board (generic)
- Two 12V PWM fans with tachometer output
- Temperature sensor (choose one):
  - DHT22 temperature and humidity sensor (use `espfan.yaml`)
  - BME280 temperature, humidity and pressure sensor (use `espfan_bme280.yaml`)
- 12V power supply (sufficient current for both fans, typically 2-4A)
- Buck converter module (12V → 5V):
  - **Best**: MP2315 buck converter (3A, ~95% efficiency, low ripple)
  - **Recommended**: XL4015 buck converter (5A, high current capability)
  - **Good**: LM2596 buck converter (3A, widely available)

## Pin Assignment

### DHT22 Variant (espfan.yaml)

| GPIO Pin | Function | Connection |
|----------|----------|------------|
| GPIO22 | DHT22 Data | Connect to DHT22 temperature sensor data pin |
| GPIO25 | PWM Output | Connect to PWM input on both fans (parallel) |
| GPIO26 | Tachometer 1 | Connect to tachometer wire of Fan 1 |
| GPIO27 | Tachometer 2 | Connect to tachometer wire of Fan 2 |

### BME280 Variant (espfan_bme280.yaml)

| GPIO Pin | Function | Connection |
|----------|----------|------------|
| GPIO21 | I2C SDA | Connect to BME280 SDA pin |
| GPIO22 | I2C SCL | Connect to BME280 SCL pin |
| GPIO25 | PWM Output | Connect to PWM input on both fans (parallel) |
| GPIO26 | Tachometer 1 | Connect to tachometer wire of Fan 1 |
| GPIO27 | Tachometer 2 | Connect to tachometer wire of Fan 2 |

## Wiring

### DHT22 Sensor (espfan.yaml)

- **VCC** (Pin 1) → ESP32 3.3V
- **Data** (Pin 2) → GPIO22
- **GND** (Pin 4) → ESP32 GND
- Add a 10kΩ pull-up resistor between VCC and Data (or use DHT22 module with built-in resistor)

### BME280 Sensor (espfan_bme280.yaml)

- **VCC** → ESP32 3.3V
- **GND** → ESP32 GND
- **SDA** → GPIO21
- **SCL** → GPIO22
- Note: Most BME280 modules have pull-up resistors already included
- Default I2C address is 0x76 (some modules use 0x77)

### Fan Connections (typical 4-pin PWM fans)

Each fan has 4 wires:
1. **Black** (black) - Ground (GND) → Connect to 12V power supply GND and ESP32 GND
2. **Red** (yellow) - +12V → Connect to 12V power supply positive
3. **Yellow** (green) - Tachometer signal → Connect to GPIO26 (Fan 1) or GPIO27 (Fan 2)
4. **Blue** (blue) - PWM control → Connect both fans' PWM wires together to GPIO25

### Power Supply

The system is powered from a single 12V power supply:

```
12V Power Supply (2-4A recommended)
    ├─[+12V]─→ Fan 1 & Fan 2 (Red wires)
    ├─[+12V]─→ Buck Converter IN+
    └─[GND]──→ Buck Converter GND, Fans (Black), ESP32 GND, Sensors GND
                    ↓
              Buck Converter (12V → 5V, MP2315/XL4015/LM2596)
                    ↓
              [5V OUT]─→ ESP32 VIN/5V pin (NOT 3.3V pin!)
              [GND]────→ ESP32 GND
```

**Important Power Notes:**
- **Use a buck converter** (not linear regulator) for efficiency and low heat
- Set buck converter output to exactly **5.0V** before connecting ESP32 (verify with multimeter)
- Connect 5V to ESP32's **VIN or 5V pin**, never to the 3.3V pin (it's an output!)
- **Common ground is critical** - connect all GND points together
- ESP32 power consumption: ~100-200mA typical, peaks to ~500mA during WiFi
- Do NOT connect USB and VIN power simultaneously
- Total system power: ~1-3A depending on fan load

**Buck Converter Setup:**
1. Connect 12V input to buck converter
2. Adjust output potentiometer to exactly 5.0V (measure with multimeter)
3. Disconnect 12V input
4. Connect output to ESP32 VIN pin
5. Reconnect 12V input and verify voltage is still 5.0V

**Power Supply Sizing:**
- Each 12V fan typically draws 0.1-0.5A at full speed
- ESP32 draws ~0.2A average
- Recommended minimum: 12V 2A power supply
- For two high-power fans: 12V 3-4A power supply

- **12V Power Supply**: Powers both fans (connect Red and Black wires)
- **ESP32**: Powered via USB or separate 5V supply
- **Important**: Connect ESP32 GND to 12V power supply GND for common ground

### Circuit Notes

- The tachometer signals are typically open-collector outputs that pull to ground
- Internal pullups are enabled on GPIO26 and GPIO27
- PWM frequency is set to 25kHz (standard for PC fans)
- Both fans share the same PWM signal, so they run at the same speed

## Home Assistant Integration

Once flashed and connected, the following entities will be available:

### Sensors
- **Temperature**: Current temperature reading from sensor
- **Humidity**: Current humidity reading from sensor
- **Pressure**: Current pressure reading (BME280 variant only)
- **Fan 1 RPM**: Real-time RPM sensor for first fan
- **Fan 2 RPM**: Real-time RPM sensor for second fan

### Controls
- **Fan**: On/Off switch and speed control (0-100%) - used in Manual mode
- **Auto Mode**: Switch to toggle between Auto and Manual mode

### Operating Modes

**Manual Mode** (Auto Mode switch OFF):
- Full manual control via Home Assistant
- Use the Fan entity to control on/off and speed (0-100%)

**Auto Mode** (Auto Mode switch ON):
- Fan speed automatically controlled by temperature
- Temperature-based control curve:
  - ≤30°C: Fan off (0%)
  - 30°C - 45°C: Linear increase (0% to 100%)
  - ≥45°C: Full speed (100%)
- Includes 2°C hysteresis to prevent rapid on/off cycling
- Manual fan controls are overridden

## Configuration

All configurations can be adjusted via the `substitutions` section in the YAML files:

```yaml
substitutions:
  temp_min: "30.0"          # Temperature where fan starts (°C)
  temp_max: "45.0"          # Temperature where fan reaches 100% (°C)
  temp_hysteresis: "2.0"    # Prevents rapid on/off cycling (°C)
  rpm_multiplier: "0.5"     # RPM calculation factor
```

### RPM Multiplier

The `rpm_multiplier` depends on your fan's tachometer signal:
- **2 pulses per revolution**: Use `0.5` (most common)
- **1 pulse per revolution**: Use `1.0`
- **4 pulses per revolution**: Use `0.25`

To find your fan's pulse rate:
1. Set `rpm_multiplier: "1.0"`
2. Note the RPM reading in Home Assistant
3. Compare with the fan's rated RPM (usually printed on the fan)
4. Adjust multiplier: `correct_multiplier = rated_rpm / measured_rpm`

## Setup

1. Choose the appropriate YAML file:
   - `espfan.yaml` for DHT22 sensor
   - `espfan_bme280.yaml` for BME280 sensor
2. Edit `secrets.yaml` with your WiFi credentials and passwords
3. (Optional) Adjust temperature thresholds and other settings in substitutions
4. Flash the ESP32 using ESPHome:
   ```bash
   esphome run espfan.yaml
   # or
   esphome run espfan_bme280.yaml
   ```
5. Add the device to Home Assistant via the API integration (should be auto-discovered)

## Troubleshooting

### Fan RPM reads incorrectly
- Adjust the `rpm_multiplier` substitution (see Configuration section)
- Verify tachometer wire is connected correctly
- Some fans may need a stronger pull-up resistor

### BME280 not detected
- Check I2C wiring (SDA/SCL)
- Try changing address from `0x76` to `0x77` in the YAML file
- Enable I2C scanning in logs to see detected addresses

### Temperature sensor not responding
- **DHT22**: Verify pull-up resistor is present (10kΩ)
- **BME280**: Confirm I2C address and wiring
- Check sensor power supply (3.3V)

### Fan doesn't turn off in auto mode
- Check that temperature is below `temp_min` minus `temp_hysteresis`
- Verify Auto Mode switch is ON in Home Assistant