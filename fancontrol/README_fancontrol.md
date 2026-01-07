# ESP Fan Controller

ESPHome-based fan controller for two 12V PWM fans integrated with Home Assistant.

## Hardware

### Components Used

- **Microcontroller**: ESP32-C3 Super Mini board
- ~~**Fans**: 2x Thermalright TL-C14CW (14cm, 4-pin PWM, max 0.13A each)~~
-- **Fans**: 3x Arctic P14 Slim PWM PST (120-1800 rpm, 0.19A, 12V, 4-pin)
- **Temperature Sensor**: DHT22 (temperature and humidity)
- **Buck Converter**: MP2315 (12V → 5V, 3A, ~95% efficiency)
- **MOSFET**: NDP6020P (P-channel, high-side switch)
- **Transistor**: BC547 (NPN, gate driver)
- **Resistors**:
  - 2kΩ (BC547 base resistor)
  - 10kΩ (MOSFET gate pullup)
- **Power Supply**: 12V 2-4A (sufficient for two fans + ESP32)
- **WiFi Antenna**: 31mm wire soldered to "0" side of onboard antenna (see WiFi notes below)

### TODO

Tacho line low-pass filter:

- Add 10k to 3.3V on the tacho line near the ESP32.
- Add 100nF to GND near the ESP32 (this plus the pull-up makes a low-pass and kills short spikes).

### Why High-Side Switching?

The P-channel MOSFET high-side switch is **essential** for this design. Testing showed that with a low-side (N-channel) switch, the fans would leak enough current through the PWM and Tacho signal lines to keep spinning even when disconnected from ground. The high-side switch completely cuts +12V power to the fans, ensuring true off state.

## Pin Assignment (ESP32-C3 Super Mini)

| GPIO Pin | Function | Connection | Notes |
|----------|----------|------------|-------|
| GPIO0 | Power Enable | BC547 base via 2kΩ resistor | ⚠️ Strapping pin - fans may briefly turn on during boot |
| GPIO1 | PWM Output | Both fans PWM pin (parallel) | 25 kHz PWM signal |
| GPIO2 | DHT22 Data | DHT22 sensor data pin | Strapping pin, safe for DHT22 after boot |
| GPIO3 | Tachometer 1 | Fan 1 tacho wire | Internal pullup enabled |
| GPIO4 | Tachometer 2 | Fan 2 tacho wire | Internal pullup enabled |
| 5V | Power | MP2315 buck converter OUT+ → ESP32‑C3 5V pin | Set to exactly 5.0V |
| GND | Common ground | Fans GND, ESP32‑C3 GND, MP2315 GND, DHT22 GND | Common ground is critical |

## Wiring

### Circuit Diagram

#### Power Switching Circuit (High-Side P-Channel MOSFET)

```
+12V Power Supply
    |
    +----------------+--- NDP6020P Source (pin 3)
    |                |
    |              10kΩ
    |                |
    |                +--- NDP6020P Gate (pin 1)
    |                          |
    |                     BC547 Collector (pin 1)
    |
    +------ NDP6020P Drain (pin 2) --- Both Fans +12V
    
ESP32-C3 GPIO0 --- 2kΩ --- BC547 Base (pin 2)

ESP32-C3 GND --- BC547 Emitter (pin 3)
             |
             +--- Power Supply GND --- Both Fans GND
```

**How the power switch works:**
1. GPIO0 HIGH → BC547 conducts → Gate pulled to GND → P-channel MOSFET ON → Fans powered
2. GPIO0 LOW → BC547 off → Gate pulled to +12V via 10kΩ → P-channel MOSFET OFF → Fans unpowered
3. The BC547 transistor provides level shifting and isolation between ESP32 (3.3V) and 12V rail

### DHT22 Sensor

- **VCC** (Pin 1) → ESP32-C3 3.3V
- **Data** (Pin 2) → GPIO2
- **GND** (Pin 4) → ESP32-C3 GND
- Add a 10kΩ pull-up resistor between VCC and Data (or use DHT22 module with built-in resistor)

### Fan Connections (Thermalright TL-C14CW)

Each fan has 4 wires:
1. **Black** - Ground (GND) → Power supply GND (common ground with ESP32)
2. **Yellow** - +12V → NDP6020P Drain (pin 2) - switched power
3. **Green** - Tachometer signal → GPIO3 (Fan 1) or GPIO4 (Fan 2)
4. **Blue** - PWM control → GPIO1 (both fans connected in parallel)

### Power Supply

The system is powered from a single 12V power supply:

```
12V Power Supply (2-4A recommended)
    ├─[+12V]─→ NDP6020P Source → Fans (switched)
    ├─[+12V]─→ MP2315 Buck Converter IN+
    └─[GND]──→ MP2315 GND, BC547 Emitter, Fans GND, ESP32-C3 GND, DHT22 GND
                    ↓
              MP2315 Buck Converter (12V → 5V)
                    ↓
              [5V OUT]─→ ESP32-C3 Super Mini 5V pin
              [GND]────→ ESP32-C3 GND
```

**Important Power Notes:**
- **Use MP2315 buck converter** for high efficiency and low heat dissipation
- Set buck converter output to exactly **5.0V** before connecting ESP32-C3 (verify with multimeter)
- Connect 5V to ESP32-C3's **5V pin**, never to the 3.3V pin (it's an output!)
- **Common ground is critical** - connect all GND points together
- ESP32-C3 power consumption: ~80-150mA typical, peaks to ~350mA during WiFi
- Do NOT connect USB and 5V pin power simultaneously
- Total system power: ~0.5-1.5A (two TL-C14CW fans at 0.13A each + ESP32-C3)

**MP2315 Buck Converter Setup:**
1. Connect 12V input to MP2315 IN+ and GND
2. Adjust output potentiometer to exactly 5.0V (measure with multimeter at OUT+ and GND)
3. Disconnect 12V input
4. Connect 5V output to ESP32-C3 Super Mini 5V pin
5. Reconnect 12V input and verify voltage is still 5.0V

**Power Supply Sizing:**
- Thermalright TL-C14CW fans: 0.13A each at full speed (0.26A total)
- ESP32-C3: ~0.15A average
- Recommended: 12V 2A power supply minimum
- MP2315 can handle up to 3A output at 5V

### Circuit Notes

- The tachometer signals are open-collector outputs that pull to ground
- Internal pullups are enabled on GPIO3 and GPIO4
- PWM frequency is set to 25kHz (standard for 4-pin PWM fans)
- Both fans share the same PWM signal, so they run at the same speed
- The 10kΩ pullup resistor keeps the MOSFET OFF during ESP32-C3 boot/reset
- GPIO0 is a strapping pin; fans may briefly turn on during boot (expected behavior)
- TL‑C14CW fans use 2 pulses per revolution → `rpm_multiplier: "0.5"`

### WiFi Antenna Fix

**Problem**: The ESP32-C3 Super Mini's onboard PCB antenna has poor WiFi connectivity, especially when enclosed or near metal.

**Solution**: Solder a 31mm piece of wire to the "0" side of the onboard antenna connector. This wire acts as an external antenna and significantly improves signal strength.

**Instructions:**
1. Use a thin, solid-core wire (22-26 AWG)
2. Cut to exactly 31mm length (quarter-wave for 2.4 GHz)
3. Solder to the "0" pad on the antenna connector (the side closer to the board edge)
4. Do NOT connect a loop to the other side - single wire only
5. Position the wire vertically or at a 45° angle for best performance

**Result**: After adding the wire antenna, WiFi connectivity was stable and reliable.

## Home Assistant Integration

Once flashed and connected, the following entities will be available:

### Sensors
- **Temperature**: Current temperature reading from DHT22 sensor
- **Humidity**: Current humidity reading from DHT22 sensor
- **Fan 1 RPM**: Real-time RPM sensor for first fan (via GPIO3 tacho)
- **Fan 2 RPM**: Real-time RPM sensor for second fan (via GPIO4 tacho)

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
        Includes 2°C hysteresis to prevent rapid on/off cycling:
          - When the fan is already running, the "turn-off" threshold is lowered to `temp_min - temp_hysteresis`.
          - Example with defaults: `temp_min=30°C`, `temp_hysteresis=2°C` → fan turns on above 30°C, but will only turn off when temperature falls below 28°C.
- Manual fan controls are overridden

## Configuration

All configurations can be adjusted via the `substitutions` section in `espfan.yaml`:

```yaml
substitutions:
  temp_min: "30.0"          # Temperature where fan starts (°C)
  temp_max: "45.0"          # Temperature where fan reaches 100% (°C)
  temp_hysteresis: "2.0"    # Prevents rapid on/off cycling (°C)
  rpm_multiplier: "0.5"     # RPM calculation factor (0.5 for 2 pulses/rev)
  fan_min_power: "0.20"     # Minimum PWM duty (20% - prevents buzzing)
  update_interval: "10s"    # Sensor polling interval
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

1. **Hardware Assembly**:
   - Build the power switching circuit (see Wiring section above)
   - Add 31mm wire antenna to ESP32-C3 Super Mini
   - Connect MP2315 buck converter and set to 5.0V
   - Wire DHT22 sensor, fans, and tacho signals

2. **Software Configuration**:
   - Edit `secrets.yaml` with your WiFi credentials and passwords
   - (Optional) Adjust temperature thresholds in `espfan.yaml` substitutions

3. **Flash ESP32-C3** using ESPHome:
   ```bash
   esphome run espfan.yaml
   ```
   - First flash requires USB connection
   - Subsequent updates can use OTA (Over-The-Air)

4. **Add to Home Assistant**:
   - Device should be auto-discovered via API integration
   - If not, manually add ESPHome integration with device IP


## Troubleshooting

### WiFi connectivity issues
- **Solution**: Add 31mm wire antenna to ESP32-C3 (see WiFi Antenna Fix section)
- Check WiFi signal strength in ESPHome logs
- Ensure router supports 2.4 GHz (ESP32-C3 doesn't support 5 GHz)
- Try reducing distance to WiFi router during initial setup

### Fans don't turn off completely
- Verify 10kΩ pullup resistor is connected between MOSFET gate and +12V
- Check BC547 transistor wiring (Collector to Gate, Emitter to GND)
- Measure gate voltage when GPIO0 is LOW (should be ~12V)

### Fans briefly turn on during boot
- **Expected behavior**: GPIO0 is a strapping pin and may briefly go HIGH during boot
- The 10kΩ pullup minimizes this, but cannot eliminate it completely
- Fans will properly turn off once ESPHome initializes (1-2 seconds)

### Fan RPM reads incorrectly
- Adjust `rpm_multiplier` substitution (see Configuration section)
- Thermalright TL-C14CW uses 2 pulses/revolution, so multiplier should be 0.5
- Verify tacho wires connected to GPIO3 and GPIO4
- Check that internal pullups are enabled in config

### DHT22 not responding
- Verify 10kΩ pull-up resistor is present between VCC and Data
- Check sensor power supply (3.3V)
- GPIO2 is a strapping pin but safe for DHT22 after boot
- Try different DHT22 module (some are defective)

### Fan doesn't turn off in auto mode
- Check that temperature is below `temp_min - temp_hysteresis`
- Example: With defaults (temp_min=30°C, hysteresis=2°C), fan turns off below 28°C
- Verify Auto Mode switch is ON in Home Assistant
- Check ESPHome logs for temperature control decisions

### ESP32-C3 won't flash or boot
- Ensure MP2315 output is exactly 5.0V (measure with multimeter)
- Check USB cable supports data (not just power)
- Hold BOOT button while connecting USB to enter bootloader mode
- Verify common ground between power supply and ESP32-C3
