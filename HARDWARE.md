# ESP Fan Controller - Hardware Reference

**Note**: This document contains detailed hardware schematics and reference information. For complete setup instructions, wiring diagrams, and configuration, see [README_fancontrol.md](README_fancontrol.md).

## Components Used

- **Microcontroller**: ESP32-C3 Super Mini board
- **Fans**: 2x Thermalright TL-C14CW (14cm, 4-pin PWM, max 0.13A each)
- **Temperature Sensor**: DHT22 (temperature and humidity)
- **Buck Converter**: MP2315 (12V → 5V, 3A, ~95% efficiency)
- **MOSFET**: NDP6020P (P-channel, high-side switch)
- **Transistor**: BC547 (NPN, gate driver)
- **Resistors**: 
  - 2kΩ (BC547 base resistor)
  - 10kΩ (MOSFET gate pullup)
- **WiFi Antenna**: 31mm wire soldered to onboard antenna

## Circuit Diagram

### Power Switching Circuit (High-Side with P-Channel MOSFET)

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

## Wiring Connections

### ESP32-C3 Super Mini to Circuit
| ESP32-C3 Pin | Connection | Notes |
|--------------|------------|-------|
| GPIO0 | BC547 Base via 2kΩ resistor | Power enable control (⚠️ strapping pin) |
| GPIO1 | Both fans PWM pin (direct) | 25 kHz PWM signal |
| GPIO2 | DHT22 Data pin | Temperature/humidity sensor |
| GPIO3 | Fan 1 Tacho pin | RPM sensing with internal pullup |
| GPIO4 | Fan 2 Tacho pin | RPM sensing with internal pullup |
| 5V | MP2315 buck converter output | 5.0V regulated power |
| GND | BC547 Emitter + common ground | Common ground for all components |

### Power Circuit
| Component | Pin | Connection |
|-----------|-----|------------|
| NDP6020P | Source (pin 3) | +12V power supply |
| NDP6020P | Drain (pin 2) | Both fans +12V line |
| NDP6020P | Gate (pin 1) | BC547 Collector + 10kΩ pullup to +12V |
| BC547 | Base (pin 2) | ESP32-C3 GPIO0 via 2kΩ resistor |
| BC547 | Collector (pin 1) | NDP6020P Gate |
| BC547 | Emitter (pin 3) | Common GND |
| MP2315 | IN+ | +12V power supply |
| MP2315 | OUT+ | ESP32-C3 5V pin (5.0V regulated) |
| MP2315 | GND | Common GND |

### Fan Connections (Thermalright TL-C14CW)
| Fan Pin | Wire Color | Connection |
|---------|------------|------------|
| 1 | Black | Common GND |
| 2 | Yellow | +12V via NDP6020P Drain |
| 3 | Green | ESP32-C3 GPIO1 (PWM, both fans parallel) |
| 4 | Blue | ESP32-C3 GPIO3 (Fan 1) or GPIO4 (Fan 2) |

## How It Works

1. **Power Control**: 
   - When ESP32-C3 GPIO0 is HIGH, BC547 conducts
   - BC547 pulls NDP6020P gate to GND
   - P-channel MOSFET turns ON, connecting +12V to fans
   - Fans receive power and respond to PWM signal
   - **Why high-side switching?** Low-side switching doesn't work - fans leak enough current through PWM/Tacho lines to keep spinning

2. **PWM Speed Control**:
   - GPIO1 outputs 25 kHz PWM signal directly to both fans (parallel)
   - PWM duty cycle controls fan speed (20-100%)
   - Fans only respond to PWM when powered via MOSFET
   - Minimum 20% duty prevents buzzing at low speeds

3. **RPM Monitoring**:
   - Each fan's tacho pin connects to dedicated GPIO (3 and 4)
   - Internal pullup enabled on ESP32-C3
   - Pulse counter measures fan speed every 10 seconds
   - Multiplier of 0.5 converts pulses to RPM (Thermalright TL-C14CW uses 2 pulses/rev)

4. **Temperature Sensing**:
   - DHT22 connected to GPIO2 reads temperature and humidity
   - 10 second update interval
   - Temperature controls fan speed in Auto Mode

## Design Notes

- **P-channel high-side switching** is essential - low-side doesn't work due to current leakage through signal lines
- **BC547 level shifter** provides isolation between 3.3V ESP32-C3 and 12V MOSFET gate
- **10kΩ pullup** keeps MOSFET OFF during ESP32-C3 boot/reset
- **2kΩ base resistor** limits BC547 base current to ~1.5mA (safe for GPIO0)
- **GPIO0 is a strapping pin** - fans may briefly turn on during boot (unavoidable)
- **MP2315 buck converter** efficiently steps down 12V to 5V for ESP32-C3
- **31mm wire antenna** required for reliable WiFi on ESP32-C3 Super Mini
- **ESPHome output component** automatically controls GPIO0 when fan turns on/off

## WiFi Antenna Modification

**Problem**: The ESP32-C3 Super Mini's onboard PCB antenna provides poor WiFi connectivity.

**Solution**: Add a 31mm external wire antenna:

1. Cut a piece of solid-core wire to exactly **31mm** (quarter-wavelength for 2.4 GHz)
2. Solder it to the pad next to "0" on the antenna connector 
3. Do NOT connect to the other pad - single wire only. The "loop" solution suggested by others did not work for me
4. Position wire vertically or at 45° angle for best performance

**Result**: Significantly improved WiFi signal strength and stability.

## Troubleshooting

### Fans don't turn off completely
- Verify 10kΩ pullup resistor between MOSFET gate and +12V
- Check BC547 wiring: Collector to Gate, Emitter to GND, Base to GPIO0 via 2kΩ
- Measure gate voltage when GPIO0 is LOW (should be ~12V = MOSFET off)
- Confirm NDP6020P orientation (Source to +12V, Drain to fans)

### Fans briefly power on during boot
- **This is expected behavior** - GPIO0 is a strapping pin
- The 10kΩ pullup minimizes this effect
- Fans will turn off within 1-2 seconds once ESPHome initializes

### Fans don't turn on
- Check BC547 transistor orientation and connections
- Verify 2kΩ base resistor is present
- Measure GPIO0 voltage when fan should be on (should be ~3.3V)
- Check MP2315 output is 5.0V

### RPM reading is wrong
- Thermalright TL-C14CW uses 2 pulses/revolution → `rpm_multiplier: "0.5"`
- Verify tacho wires connected to GPIO3 and GPIO4
- Check that internal pullups are enabled in config

### Fans buzz at low speeds
- Increase `fan_min_power` in config (try 0.25 or 0.30)
- Some fans won't run smoothly below 30% duty cycle

### WiFi connectivity issues
- Add 31mm wire antenna (see WiFi Antenna Modification section)
- Check antenna solder joint
- Ensure router supports 2.4 GHz (ESP32-C3 doesn't support 5 GHz)

### ESP32-C3 won't boot or flash
- Verify MP2315 output is exactly 5.0V (use multimeter)
- Check USB cable supports data transfer (not power-only)
- Hold BOOT button while connecting USB to enter bootloader
- Ensure common ground between 12V supply and ESP32-C3
