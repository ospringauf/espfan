# ESP Fan Controller - Hardware Configuration

## Components Used

- **Microcontroller**: ESP32 Dev Board
- **Fans**: 2x 4-pin PWM fans (max 0.13A each)
- **MOSFET**: NDP6020P (P-channel)
- **Transistor**: BC547 (NPN)
- **Resistors**: 
  - 2kΩ (base resistor)
  - 10kΩ (pullup resistor)

## Circuit Diagram

### Power Switching Circuit (High-Side with P-Channel MOSFET)

```
+12V Power Supply
    |
    +----------------+--- NDP6020P Source (pin 3)
    |                |
    |              10k
    |                |
    |                +--- NDP6020P Gate (pin 1)
    |                          |
    |                     BC547 Collector (pin 1)
    |
    +------ NDP6020P Drain (pin 2) --- Both Fans +12V
    
ESP32 GPIO33 --- 2kΩ --- BC547 Base (pin 2)

ESP32 GND --- BC547 Emitter (pin 3)
          |
          +--- Power Supply GND --- Both Fans GND
```

## Wiring Connections

### ESP32 to Circuit
| ESP32 Pin | Connection | Notes |
|-----------|------------|-------|
| GPIO33 | BC547 Base via 2kΩ resistor | Power enable control |
| GPIO25 | Both fans PWM pin (direct) | 25 kHz PWM signal |
| GPIO26 | Fan 1 Tacho pin | RPM sensing with internal pullup |
| GPIO27 | Fan 2 Tacho pin | RPM sensing with internal pullup |
| GND | BC547 Emitter + common ground | |

### Power Circuit
| Component | Pin | Connection |
|-----------|-----|------------|
| NDP6020P | Source (pin 3) | +12V power supply |
| NDP6020P | Drain (pin 2) | Both fans +12V line |
| NDP6020P | Gate (pin 1) | BC547 Collector + 10kΩ pullup to +12V |
| BC547 | Base (pin 2) | ESP32 GPIO33 via 2kΩ resistor |
| BC547 | Collector (pin 1) | NDP6020P Gate |
| BC547 | Emitter (pin 3) | Common GND |

### Fan Connections (per fan)
| Fan Pin | Wire Color | Connection |
|---------|------------|------------|
| 1 | Black | Common GND |
| 2 | Yellow | +12V via NDP6020P |
| 3 | Green | ESP32 GPIO25 (PWM) |
| 4 | Blue | ESP32 GPIO26/27 (Tacho) |

## How It Works

1. **Power Control**: 
   - When ESP32 GPIO33 is HIGH, BC547 conducts
   - BC547 pulls NDP6020P gate to GND
   - P-channel MOSFET turns ON, connecting +12V to fans
   - Fans receive power and respond to PWM signal

2. **PWM Speed Control**:
   - GPIO25 outputs 25 kHz PWM signal directly to both fans
   - PWM duty cycle controls fan speed (20-100%)
   - Fans only respond to PWM when powered via MOSFET

3. **RPM Monitoring**:
   - Each fan's tacho pin connects to dedicated GPIO (26/27)
   - Internal pullup enabled on ESP32
   - Pulse counter measures fan speed
   - Multiplier of 0.5 converts pulses to RPM

## Design Notes

- **P-channel high-side switching** ensures clean fan GND for tacho signals
- **BC547 level shifter** provides isolation and proper 12V gate drive
- **10kΩ pullup** keeps MOSFET OFF during ESP32 boot/reset
- **2kΩ base resistor** limits BC547 base current to ~1.5mA
- **Power supply component** in ESPHome automatically controls GPIO33

## Troubleshooting

- If fans don't turn off completely, verify 10kΩ pullup is connected
- If fans don't turn on, check BC547 wiring (Collector to Gate, Emitter to GND)
- If RPM reading is wrong, adjust `rpm_multiplier` substitution in config
- If fans buzz at low speeds, increase `fan_min_power` substitution
