# Lab 4 — Edge sensing and hill climbing

The goal was to drive the Pololu up a ramp, turn around on the flat top, and drive back down. The work splits in two. Section 8.2 uses the five downward infrared sensors to see the dark tape on the edges of the ramp and to turn away from it. Section 8.3 points the robot up and down the slope with the accelerometer. The roll angle is the share of gravity on the side-to-side axis, and the heading correction drives that angle back toward zero at every sample.

Programs were checked by flashing them onto the Pololu: hold B and press reset to enter BOOTSEL, build with `lfc`, then run `picotool load -x bin/<program>.elf`. The motors run only when batteries are installed and the power button is on. Every program uses the C target, the RP2040 on the `pololu_3pi_2040_robot` board, and `single-threaded: true`.

## 8.2 Line sensing

The line sensors are five infrared LEDs and phototransistors aimed at the ground, read through GPIO. `src/lib/Line.lf` documents the result: each reading is an integer from 0 to 1024, and a larger number means less reflected light. On this ramp the edges are covered with dark tape and the driving surface is light, so the tape returns a high reading and the bare ramp returns a low one. That difference is the edge detector. The same sensors can see a real drop-off, because a cliff returns little infrared, but ambient infrared makes that reading unreliable, so this lab uses the tape.

The reactor starts in calibration. Each trigger calls `line_sensors_calibrate()`, which records the brightest and darkest value each sensor has seen. A `false` on the `calibrate` input ends that phase, and later triggers call `line_sensors_read_calibrated()`. The stored limits are lost every time a new program is flashed, so the robot has to be rolled over the light surface and the tape again after every upload. Section 6.5 of the Pololu 3pi+ 2040 user's guide says the bump sensors and the line sensors cannot be used together, so these programs use only the line sensors. Index 0 is the leftmost sensor and index 4 is the rightmost.

`src/LineDisplay.lf` is the provided viewer. A 100 ms timer triggers the sensors. For the first 10 seconds the LCD says to roll the robot over light and dark, and the sensors stay in calibration. After that the display prints the five calibrated numbers, two per line. Watching those numbers on the ramp is how the dark threshold was chosen: a sensor sitting on the tape reads well above a sensor sitting on the ramp, and 900 is high enough to mean "this sensor is on the tape."

### Detecting an edge

`src/HillLineDetectSolution.lf` keeps that 10 second calibration, then labels each sample on the LCD. A reading of 900 or more is dark.

```
sensor 0 >= 900              line 0:  LEFT T    else LEFT F
sensor 1 or 2 >= 900         line 1:  CENTER T  else CENTER F
sensor 4 >= 900              line 2:  RIGHT T   else RIGHT F
```

The three lines are independent, so the left and the center can both read true when the robot meets a corner of the tape. Pushing the front of the robot onto an edge marker by hand is the checkoff: the matching line flips to `T` while that sensor is over the tape, and back to `F` when it is over the light surface again.

### Driving away from an edge

`src/HillLineAvoidSolution.lf` calibrates for the same 10 seconds, displaying `CALIBRATING`, then commands both motors to `0.2` and enters `DRIVING`. The line sensors are sampled every 100 ms. The gyroscope is sampled every 2 ms and integrated by `GyroAngle`, the same trapezoidal integrator the square used, so `gyro.z` is yaw in degrees since the first sample.

```
CALI    -- 10 s elapsed              --> DRIVING
DRIVING -- any sensor sees the tape  --> BACKING
BACKING -- 0.5 s of reverse          --> TURNING
TURNING -- yaw has changed by 45°    --> DRIVING
```

In `DRIVING`, a dark sensor stores the logical time and reverses both motors to `-0.2`. Which sensor fired chooses the later turn, and the leftmost sensor wins if it is dark at the same time as another:

```
sensor 0 dark                  hitting = 2    turn right
sensor 1, 2, 3, or 4 dark      hitting = 1    turn left
```

`BACKING` waits until logical time is 500,000,000 ns past that stamp. It then snapshots the yaw and starts the spin. A left-edge hit (`hitting == 2`) commands the left motor to `0.2` and the right motor to `-0.2`, which swings the nose to the right, off the tape. Any other dark sensor commands the opposite spin, left motor `-0.2` and right motor `0.2`. `TURNING` holds that spin until the yaw has moved 45° from the snapshot, then sets both motors back to `0.2` and returns to `DRIVING`.

The turn is a change in yaw, taken from the value saved at the end of the backup. It is a fixed 45° so the robot leaves the edge and then drives, and the side that saw the tape decides which way. The checkoff is the robot driving on the ramp, backing up when the front reaches the tape, turning away from that side, and continuing.

## 8.3 Climbing with roll feedback

`src/HillClimbSolution.lf` is the climb. The heading correction is the roll term below, recomputed every time a new accelerometer sample arrives. The line sensors are the edge detectors of Section 8.2; this program samples the accelerometer and the gyroscope.

`Tilt` in `src/lib/Tilt.lf` turns the three accelerometer channels, in g, into two angles in degrees. π is taken as 3.14:

```
x_angle = atan(x / sqrt(y² + z²)) * 180 / π
y_angle = atan(y / sqrt(x² + z²)) * 180 / π
```

Each angle is how far that axis has tilted out of the horizontal. On level ground the vertical axis carries the whole 1 g and both angles are near zero. As the robot tilts, a larger share of gravity falls on the tilted axis, the fraction inside the `atan` grows, and the angle grows with it. `Tilt` writes the x angle to its `row` output and the y angle to its `pitch` output.

`main` crosses those two wires:

```
tilt.row   -> robot.pitch
tilt.pitch -> robot.row
```

Inside `Robot`, `row` is the steering error and `pitch` is the slope measurement. A 20 ms timer triggers the accelerometer and `GyroAngle` together. `Tilt` publishes both angles from that accelerometer sample, and the driving reaction runs off `row`, reading `pitch` and the integrated yaw on the same tick.

Pointed straight up the ramp, the two sides of the robot are at the same height, so the side-to-side angle is about zero and gravity shows up as pitch, the steepness of the ramp. Yaw the robot across the slope and one side sits lower than the other, so part of gravity appears on the lateral axis and `row` leaves zero. Holding `row` near zero holds the nose on the fall line, uphill on the way up and downhill on the way down. That is the feedback loop from Section 2.4 of Lee and Seshia: the measurement is `row`, the reference is 0, and the actuator command is the difference between the two wheel powers. Because the correction is proportional to the error, it shrinks as the robot gets closer to straight up or straight down the ramp.

`direction` starts true, which means uphill, and `seen_slope` starts false. On each sample the driving reaction does three things.

A pitch whose magnitude passes 8° latches `seen_slope`. The floor at the bottom is flat, so the latch is what makes the later flat reading mean "this is the top" after the robot has actually been on the ramp.

While still going up, a pitch inside 2° and a roll inside 3°, after the slope has been seen, is the plateau. The reaction saves the current yaw, commands a spin in place (left motor `0.1`, right motor `-0.1`), displays `TURNING`, and enters `TURNING`.

Otherwise it applies the proportional law. Uphill the base power is `0.2` and the gain on roll is `0.01` per degree:

```
left  = 0.2 - 0.01 * row
right = 0.2 + 0.01 * row
```

A positive `row` slows the left wheel and speeds the right wheel. A negative `row` does the opposite. Ten degrees of roll is a 0.1 difference on each wheel, so the powers stay near the base command while the heading is close, and they separate further while the robot is crossing the slope.

`TURNING` waits until the integrated yaw has changed by 180° from the saved angle (`fabsf` of the difference). It then sets `direction` false, displays `GOING DOWN`, and returns to `DRIVING`. The spin command is left in place until the next roll sample overwrites it. The plateau test also requires `direction` to still be true, so the flat top does not start a second turn once the robot is committed to going down, and the flat floor at the bottom does not start one either.

Downhill the same gain is applied with the opposite sign, and the base power is cut in half:

```
left  = 0.1 + 0.01 * row
right = 0.1 - 0.01 * row
```

The sign flips because the robot is facing the other way: the wheel that has to speed up to cancel a given body-frame roll is the other wheel. The smaller base power is the pitch problem the lab points at. The same motor command produces a very different speed once gravity is helping, so the downhill command is a lower open-loop power and the roll term only trims the heading.

`src/lib/MotorsWithFeedback.lf` is the other way to handle that pitch dependence. It takes a desired speed in meters per second and the encoder counts, estimates each wheel's speed from the change in counts, and runs a proportional-integral controller (default gains 0.5 and 0.25) so the delivered power tracks the requested speed. The wheel model there is a diameter of 0.032 m and 360 counts per revolution. The climb program commands power on `Motors` directly, with the two base powers above, and uses the roll loop for direction.

The checkoff is the robot driving to the top under the roll correction, turning 180° once the pitch has fallen back toward level, and driving down the ramp with the slower, sign-flipped command.
