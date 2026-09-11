# Stage 04 — Raspberry Pi Pico, Electronics & Drive Base

## Goal

Build the low-level controller that safely drives the real robot.

At the end of this stage the Pico should:

- boot reliably;
- control both motor directions;
- control both motor speeds with PWM;
- stop immediately on command;
- communicate over USB serial;
- run a non-blocking firmware loop;
- include a command watchdog;
- leave clean interfaces for encoders and sensors in later stages.

Do this on the bench before installing everything into the chassis.

---

## 1. Understand the Pico's Role

The Pico is **not** running Nav2, SLAM or heavy ROS software.

It is responsible for time-sensitive hardware work:

```text
motor PWM
motor direction
encoder counting
MPU6050 reading
ultrasonic timing
IR reading
vacuum/brush switching
battery measurement
serial telemetry
command timeout / emergency stop
```

The laptop remains responsible for:

```text
ROS 2
static map
sensor fusion
Nav2
coverage planning
high-level decisions
```

This division keeps the system simple and robust.

---

## 2. Recommended Firmware Choice

### Final firmware: Pico SDK C/C++

Use the official C/C++ SDK for the project firmware because it gives direct control over:

- PWM;
- interrupts;
- timers;
- USB/UART;
- I2C;
- PIO if we need it later.

Official resources:

- Raspberry Pi Pico documentation: https://www.raspberrypi.com/documentation/microcontrollers/pico-series.html
- Pico C/C++ SDK: https://www.raspberrypi.com/documentation/microcontrollers/c_sdk.html
- Pico SDK API: https://www.raspberrypi.com/documentation/pico-sdk/
- Pico examples repository: https://github.com/raspberrypi/pico-examples

### Optional learning/prototyping: MicroPython

MicroPython is fine for testing a sensor quickly, but do not create half the final firmware in MicroPython and half in C unless there is a clear reason.

A very good beginner Pico video course:

- Core Electronics Pico beginner course: https://www.youtube.com/watch?v=Ic4ExTusoTw

Prioritize GPIO, PWM, ADC, I2C and UART/serial sections.

---

## 3. Electrical Architecture

A safe simplified power tree is:

```text
Battery pack
   |
   +---- BMS / fuse / main switch
   |
   +---- motor driver VM --------> wheel motors
   |
   +---- vacuum power -----------> vacuum motor through MOSFET/driver
   |
   +---- buck converter ---------> regulated logic supply
                                    |
                                    +--> Pico
                                    +--> compatible sensors
```

### Critical rules

1. Never power a DC motor directly from a Pico GPIO.
2. All logic/control grounds must share a common reference unless intentionally isolated.
3. Confirm sensor output voltages before connecting them to 3.3 V GPIO.
4. Put a fuse near the battery source.
5. Keep high-current vacuum/motor wiring physically away from IMU/signal wiring where possible.
6. Add decoupling near motor driver/logic boards.
7. Do not assume the battery cells are safe for 3S just because they physically fit.

Battery pack assembly is a safety-critical part of the project. Use matched cells, a suitable BMS and charger, and do not charge an improvised pack unattended.

---

## 4. Motor Driver Selection

Our cheap baseline is **TB6612FNG** if the existing motors fit its current capability.

Typical module capability is around:

```text
~1.2 A continuous per channel
higher short peak capability
```

Reference:

- https://www.sparkfun.com/products/14451
- https://lite.digilog.pk/products/tb6612fng-motor-driver-module-for-arduino-in-pakistan

### Before buying/using it

Find or measure the motor's **stall current**.

Stall current is the worst case when the shaft cannot rotate. A motor that normally uses 300 mA may demand much more when stalled.

If the motor stall current is too high, use a stronger H-bridge instead of hoping the TB6612 survives.

Do not choose L298N just because it is common in old tutorials. It works for prototypes, but it has much larger voltage losses and is inefficient compared with modern MOSFET H-bridges.

---

## 5. Example Pico Pin Plan

This is a starting point, not a permanent law.

```text
GP0   I2C SDA -> MPU6050
GP1   I2C SCL -> MPU6050

GP2   TB6612 AIN1
GP3   TB6612 AIN2
GP4   TB6612 PWMA

GP6   TB6612 BIN1
GP7   TB6612 BIN2
GP8   TB6612 PWMB
GP9   TB6612 STBY

GP10  left encoder
GP11  right encoder

GP12  ultrasonic trigger
GP13  ultrasonic echo (through safe level conversion if needed)

GP14-18  IR array inputs if digital and suitable
```

Keep some GPIO free for:

- extra ultrasonic sensors;
- vacuum/brush control;
- battery ADC;
- additional encoder channel if quadrature encoders are used.

If the IR array has analog outputs, the pin plan must change because not every Pico GPIO is an ADC input.

---

## 6. HC-SR04 / Ultrasonic Voltage Warning

Many common HC-SR04 modules use 5 V and can output approximately 5 V on `ECHO`.

The Pico GPIO is 3.3 V logic.

Use a voltage divider or proper level shifter for a 5 V echo signal.

Example resistor-divider concept:

```text
HC-SR04 ECHO ---- resistor ----+---- Pico GPIO
                               |
                            resistor
                               |
                              GND
```

Choose resistor values that bring the maximum echo voltage safely below the Pico GPIO limit.

Do not copy a circuit without understanding the voltage ratio.

---

## 7. Bench Bring-Up Order

Never wire the entire robot and then switch it on for the first time.

### Test 1 — Pico only

Flash a simple LED/serial program.

Confirm:

```text
Pico boots
USB appears on laptop
serial output is stable
```

### Test 2 — Motor driver without motors

Verify logic pins and standby behavior.

### Test 3 — One motor

Lift the wheel/motor off the table.

Test:

```text
stop
forward 20% PWM
forward 50%
reverse 20%
reverse 50%
stop
```

### Test 4 — Second motor

Repeat independently.

### Test 5 — Both motors

Run both at low PWM and confirm no reset/brownout.

If the Pico resets when motors start, investigate power integrity before continuing.

---

## 8. Motor Control API

Do not scatter GPIO writes throughout the firmware.

Create a clear interface such as:

```c
void motor_left_set(float command);
void motor_right_set(float command);
void motors_stop(void);
```

where command is normalized:

```text
-1.0 = full reverse
 0.0 = stop
+1.0 = full forward
```

Later Stage 5 will replace direct command PWM with closed-loop velocity targets.

### Direction logic

Conceptually:

```text
command > 0 -> forward direction + PWM(abs(command))
command < 0 -> reverse direction + PWM(abs(command))
command = 0 -> stop/brake policy
```

Do not reverse from full forward to full reverse instantly. Ramp or stop first.

---

## 9. PWM

Understand:

```text
PWM frequency
PWM duty cycle
motor dead-zone
minimum useful duty
```

A motor often will not move at 5% PWM because static friction is too high.

Measure for each motor:

```text
minimum PWM where motor reliably starts
minimum PWM where motor keeps turning
```

Store these values. Stage 5 PID tuning will need them.

---

## 10. Firmware Timing Architecture

Avoid this style:

```c
while (true) {
    read_everything();
    sleep_ms(1000);
}
```

A robot needs different tasks at different rates.

Good initial target:

```text
motor/PID loop      50–100 Hz
encoder processing  continuous / interrupt-driven
IMU                 50–100 Hz
ultrasonic           10–20 Hz per sensor
IR                   20–50 Hz
serial receive      continuous/non-blocking
telemetry            20–50 Hz
watchdog             every loop
```

Do not trigger several ultrasonic sensors simultaneously because they can hear each other's echoes. Stagger them.

---

## 11. Command Watchdog

This feature is mandatory.

The Pico should remember when it last received a valid motion command.

Concept:

```c
if (time_since_last_command > 500_ms) {
    target_left = 0;
    target_right = 0;
    motors_stop();
}
```

The exact timeout can be tuned, but the principle is non-negotiable.

Test the watchdog by physically unplugging USB while wheels are raised. They must stop automatically.

---

## 12. USB Serial

For the first version, use simple USB serial between Pico and laptop.

We will define the full protocol in Stage 7.

For now make the Pico support commands like:

```text
M,0.25,0.25
M,0.20,-0.20
STOP
```

and respond with:

```text
OK
```

Use newline-terminated messages so parsing is simple.

Do not use `printf()` debugging in a way that corrupts the machine-readable telemetry stream later. Consider a debug mode or message prefix.

---

## 13. Noise and Brownout Testing

DC motors generate electrical noise.

Test these cases:

1. motors start together;
2. one motor rapidly changes speed;
3. both motors reverse at low power;
4. vacuum motor starts;
5. motors stall briefly under controlled/safe test conditions.

Watch for:

```text
Pico USB disconnects
Pico resets
MPU6050 corrupt readings
serial garbage
motor driver overheating
buck converter voltage drop
```

Fix electrical instability now. ROS cannot fix a bad power system.

---

## 14. Suggested Firmware Structure

```text
firmware/
├── CMakeLists.txt
├── pico_sdk_import.cmake
├── src/
│   ├── main.c
│   ├── motor.c
│   ├── motor.h
│   ├── serial_protocol.c
│   ├── serial_protocol.h
│   ├── safety.c
│   └── safety.h
└── README.md
```

Encoders and sensor modules are added in later stages.

Keep hardware-specific operations in modules instead of a 1000-line `main.c`.

---

## Stage Acceptance Checklist

- [ ] Pico flashes and connects over USB reliably.
- [ ] Both motors can move independently forward/reverse.
- [ ] PWM changes motor speed.
- [ ] Motor driver remains within safe temperature/current limits.
- [ ] Pico does not reset when motors start.
- [ ] USB serial commands can control the bench motors.
- [ ] A watchdog stops motors when commands disappear.
- [ ] Emergency `STOP` works immediately.
- [ ] High-current and logic power architecture is documented.
- [ ] Encoder GPIO pins remain available for Stage 5.

---

## Best References

- Raspberry Pi Pico documentation: https://www.raspberrypi.com/documentation/microcontrollers/pico-series.html
- Pico C/C++ SDK: https://www.raspberrypi.com/documentation/microcontrollers/c_sdk.html
- Pico SDK examples: https://github.com/raspberrypi/pico-examples
- Core Electronics Pico course: https://www.youtube.com/watch?v=Ic4ExTusoTw
- TB6612FNG guide/datasheet links: https://www.sparkfun.com/products/14451

**Next:** [Stage 05 — Encoders, PID & Wheel Odometry](./05-encoders-pid-odometry.md)
