# Lab 3 — Motors, encoders, and modal navigation

The goal was to drive the Pololu from Lingua Franca modes, turn wheel rotation into a distance, and use the gyroscope and the bump sensors as the guards that switch those modes.

Programs were checked by flashing them onto the Pololu: hold B and press reset to enter BOOTSEL, build with `lfc`, then run `picotool load -x bin/<program>.elf`. The motors run only when batteries are installed and the power button is on, so the blue LED is lit. USB power alone does not feed the motor drivers.

Every program uses the same target as the earlier labs: the C target, the RP2040 on the `pololu_3pi_2040_robot` board, and `single-threaded: true`. A reaction body is C. The Lingua Franca text says when that C runs; the C says what it writes to a motor, the display, or the mode.

## 7.2 Motors and PWM

`src/lib/Motors.lf` takes a power in the range −1 to 1 on `left_power` and `right_power`. The sign is the direction. The magnitude is the duty cycle. Each input reaction calls `set_power`, which clamps the magnitude to 1 and converts it to a 12-bit level:

```
duty_cycle = (uint16_t)(fabsf(power) * MAX_MOTOR_POWER);
motors_set_power(duty_cycle, forward, left);
```

`MAX_MOTOR_POWER` is `0x0FFF` (4095). A command of `0.1f` is a duty of about 410, roughly 10% of full scale.

`motors_init(1)` is called once at startup. The `1` is the PWM clock divider. The function claims the four motor pins from prelab question 3:

```
GP10   right direction     DRV8838 PHASE
GP11   left direction      DRV8838 PHASE
GP14   right speed         PWM7 A, DRV8838 ENABLE
GP15   left speed          PWM7 B, DRV8838 ENABLE
```

GP14 and GP15 share one PWM slice. The wrap value is 4095 and the divider is 1, so the period is 125 MHz / 4096, about 30.5 kHz. `motors_set_power` clamps the level to 4095, drives the direction pin low for forward (`gpio_put(pin, !forward)`), and stores the level with `pwm_set_gpio_level`. The fraction `level / 4095` is the fraction of each period that ENABLE is high. The GPIO pin carries that logic waveform. The DRV8838 switches battery current into the motor. That is the PWM scheme from the prelab: the hardware block on those pins sets the duty cycle, and the driver, not the pin, supplies the power.

`src/RobotDriveSolution.lf` is `RobotTemplate.lf` with those powers wired in. A 2 second timer toggles a `drive` boolean. The `Robot` reactor owns the mode. Its outputs are the display string and the two power values, and `main` connects the powers to `Motors`.

```
startup                 INIT,   both motors 0
STOPPED  -- drive -->   command 0.1; if drive is true,  enter DRIVING
DRIVING  -- drive -->   command 0;   if drive is false, enter STOPPED
```

Startup writes `INIT` and zero power. The `STOPPED` reaction writes `0.1f` on both motors, then enters `DRIVING` and displays `DRIVING` when `drive` is true. The `DRIVING` reaction writes `0.0f` on both motors, then enters `STOPPED` and displays `STOPPED` when `drive` is false. The transition uses `reset`, so a timer that lived inside the mode being entered would start over. The checkoff is the LCD moving from `INIT` to `DRIVING` and `STOPPED`, with the wheels turning only in `DRIVING`, together with the `motors_set_power` path above.

## 7.3 Encoders

`src/EncoderDisplay.lf` prints the raw encoder counts. The library treats each count as one degree: about 12 counts per motor-shaft revolution, times the 30:1 gear ratio, which is about 358 counts per wheel revolution and is taken as 360 degrees. The count increases as the wheel rolls forward. It is a signed 32-bit value, so it wraps at 2,147,483,647 degrees.

The wheel diameter is 3.175 cm, so one revolution travels the circumference:

```
distance = (degrees / 360) * π * diameter
```

With the diameter in meters, `0.03175`, the distance is in meters and one revolution is about 0.100 m. `AngleToDistance` in `src/RobotEncoderSolution.lf` uses the diameter in centimeters (`3.175`) and `3.14` for π, so the number it produces is centimeters. The first sample is stored as the zero point and later samples subtract it, which is the change since the program started. The LCD title is `Distance:`, and the two lines are `L:` and `R:` in cm. Rolling the robot straight for 50 cm should put both lines near 50.

The signed count overflows after 2,147,483,647 / 360 revolutions. Times the circumference, π × 0.03175 m, that is about 595 km. These labs travel meters, so wraparound is not a concern.

## 7.4 A square

`GyroAngle` in `src/lib/IMU.lf` integrates the gyroscope with the trapezoidal rule. The output is degrees since the first sample. On a flat floor the yaw is the z axis, and that is the input the square uses.

`src/RobotSquareSolution.lf` is two modes. Distance comes from the right wheel only, in the centimeters produced by `AngleToDistance`. A 2 ms timer triggers the encoder and the gyro.

```
DRIVING -- right wheel has gained 50 cm --> TURNING
TURNING -- yaw >= 90° × turn count     --> DRIVING
```

`DRIVING` starts as the initial mode. Startup commands both motors to `0.1f` and displays `DRIVING`. Each time the right-wheel distance has grown by 50 cm since `start_distance`, the reaction increments `turns`, commands a left spin (left motor `-0.1f`, right motor `0.1f`), displays `TURNING`, and enters `TURNING` with `reset`. `TURNING` waits until `gyro.z` reaches `90 * turns`. The first corner waits for 90°, the next for 180°, then 270°, and so on, because the integrator never wraps back to zero. On the way out it snapshots `start_distance`, commands both motors forward, and returns to `DRIVING`.

The square is uneven for a few independent reasons:

- The straight leg is open loop. Equal duty cycles do not make equal wheel speeds, so the leg is a shallow arc. Only the right wheel is measured, so that arc can still count as 50 cm.
- The gyro bias is integrated the whole time, and the trapezoidal sum only sees a sample every 2 ms. A drifting zero shifts every corner, because the target is an absolute angle.
- The guard is checked on a sample, and the robot is still moving when it becomes true, so each side and each corner overshoots.
- The 3.175 cm diameter and the 360-count approximation scale the 50 cm legs.
- The caster and the floor add drag that a fixed duty cycle does not account for.

The checkoff is the robot tracing the square and the mode diagram above.

## 7.5 Obstacle avoidance

`src/BumpDisplay.lf` instantiates `Bump`. That reactor calibrates the front IR bumpers at startup and reads them every 100 ms. On a sample where the left bumper is pressed it produces `left`; the right bumper produces `right`. A released bumper produces no output on that tick. The display program writes `Left Bumped!` or `Right Bumped!` and schedules a logical action 2 seconds later to clear the line.

`src/RobotAvoidSolution.lf` keeps the square and adds a `BUMP` mode. The bump outputs are inputs of `Robot`. A press is handled from `DRIVING`.

```
DRIVING -- right wheel has gained 50 cm --> TURNING -- yaw has reached the next 90° --> DRIVING
DRIVING -- left or right bumper        --> BUMP
BUMP    -- 0.5 s of reverse            --> TURNING
```

The bump reaction commands both motors to `-0.1f`, displays `BUMP`, records `lf_time_logical_elapsed()` in `back_start`, and enters `BUMP`. `BUMP` reacts to new gyro samples. When logical time is more than 500,000,000 ns (half a second) past `back_start`, it increments `turns`, commands the same left spin as a normal corner, and enters `TURNING`. From there the square's turn reaction runs: it spins until yaw reaches the new multiple of 90°, then drives again. Backing up for half a second pulls the robot off the obstacle, and the following turn points it away before the next 50 cm leg. The bumper that closed does not pick the spin direction; every avoidance turn spins left, the same way the square corners do.

The checkoff is the robot leaving an obstacle and the three-mode diagram above.
