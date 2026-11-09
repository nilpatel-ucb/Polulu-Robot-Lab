# Lab notes

What each lab asked for, and what we did on the robot. The code for a lab is the branch of the same name. Programs were checked by flashing them onto the Pololu: hold B and press reset to enter BOOTSEL, build with `lfc`, then run `picotool load -x bin/<program>.elf`.

## Lab 1 — LED reactor, display, and tilt

Branch: `lab1`

The goal was to make a reusable LED reactor, drive the LCD, calibrate the accelerometer, and show the robot's pitch and roll.

### Reusable LED

`src/Blink.lf` blinks the onboard LED, but the GPIO code sits in the `main` reactor. A `main` reactor is the program entry point. Other Lingua Franca files cannot import it, so that blink logic cannot be reused.

We split it into a reactor with one input. `src/LED.lf` defines `LED` with `input set: bool`. On startup it configures `PICO_DEFAULT_LED_PIN` as an output. When `set` arrives `true`, the reaction turns the LED on with `gpio_put`. When `set` arrives `false`, it turns the LED off.

`src/ToolsLEDSolution.lf` imports that reactor. Two timers drive the input: one writes `true` every 250 ms, the other writes `false` every 400 ms, so the LED blinks. The checkoff was to show the program running and walk through those two reactions.

Every one of these files starts from the same target block:

```
target C {
  platform: {
    name: "rp2040",
    board: "pololu_3pi_2040_robot"
  },
  single-threaded: true,
}
```

`lfc` generates C. The platform selects the Raspberry Pi RP2040 runtime and the Pololu board. `single-threaded: true` means the target is bare metal: no operating system and no thread library, so reactions run one at a time.

The LED reactions call Pico SDK functions declared in `hardware/gpio.h`. That header is pulled in here:

```
preamble {=
  #include <hardware/gpio.h>
=}
```

Dropping the preamble makes the C compile fail, because `gpio_init`, `gpio_set_dir`, `gpio_put`, and `PICO_DEFAULT_LED_PIN` are never declared. Those calls write the memory-mapped GPIO registers. On this board, `PICO_DEFAULT_LED_PIN` is the LED on the robot.

### The display

`src/lib/Display.lf` is the LCD reactor. It has four string inputs, `line0` through `line3`. On startup it initializes the display once, selects the 8×16 font, and clears the frame buffer. When a line input arrives, the reaction clears that 16-pixel-tall row, draws the string, and calls `display_show()`. Callers format text with `snprintf` into a 17-byte buffer (16 characters plus a null terminator), which matches the width of that font.

### Calibrating the accelerometer

The IMU is an ST LSM6DSO. `src/lib/IMU.lf` samples it and reports acceleration in g's. A raw sample is not the true acceleration. Over the range we used, the relationship is

```
a_measured = sensitivity * a_true + bias
```

so the value we display is

```
a_true = (a_measured - bias) / sensitivity
```

Gravity is the known reference, 1 g. To find the bias on one axis, we tilt the robot until that axis is perpendicular to gravity. The true acceleration on the axis is then 0, because the components of gravity on it cancel, and whatever the sensor still reports is the bias.

To find the sensitivity, we tilt that same axis until it points straight up, where the true specific force is +1 g. After subtracting the bias, the remaining reading is the sensitivity.

The constants we measured are applied in `src/AccelerometerDisplay.lf` before the numbers are written to the LCD:

```
x_true = (x - 0.01365) / 0.99885
y_true = (y + 0.052356) / 0.9904
z_true = (z - 0.0165) / 1.0035
```

The y equation adds 0.052356 because the y bias we measured was negative, −0.052356. Adding it is the same subtraction as in the general formula.

### What the accelerometer shows on a flat table

Sitting level, x and y read near 0 g and z reads near +1 g.

x and y point horizontally, so gravity has almost no component along them. z points up. The chip reports specific force, the acceleration from everything except gravity. The table pushes up on the robot at 1 g to keep it from falling, and that upward force is what the z axis records. The reading is near +1 g because the support force points up along z.

Yaw is invisible to this sensor. Rotating the robot about the vertical axis leaves the gravity vector in the same place relative to x, y, and z. A magnetometer would be needed to measure yaw against magnetic north.

### The four reactions

`src/AccelerometerDisplay.lf` is the program we flashed for the axis checkoff. Sampled every 250 ms, the path from timer to LCD is four reactions:

1. The timer reaction in `main` fires every 250 ms and sets `Accelerometer.trigger`.
2. The accelerometer reaction handles that trigger, reads the LSM6DSO, and outputs x, y, and z in g's.
3. The reaction on those three outputs applies the bias and sensitivity equations above and writes one string per axis.
4. The `Display` reaction takes those strings and paints lines 0, 1, and 2 of the LCD.

`single-threaded: true` matters for more than the missing OS. The IMU and display reactors keep one shared device each and do not take locks. Reactions have to run to completion without another reaction touching the same device. That holds only because the runtime is single-threaded.

### Pitch and roll

The lab asked for inclination: how far the surface under the robot is from level. Pitch is the tilt of a line pointing out the front of the robot. Roll is the tilt of a line through the wheels. We only computed those two. The robot stays right-side up, and the angles of interest are small deviations from flat.

`src/lib/Tilt.lf` takes the accelerometer's x, y, and z and outputs the angles in degrees, using the geometry from Fisher, *Using an Accelerometer for Inclination Sensing*:

```
row   = atan( x / sqrt(y² + z²) ) * 180 / π
pitch = atan( y / sqrt(x² + z²) ) * 180 / π
```

On the 3pi+ the accelerometer x axis points forward and the y axis points to the left, so the `row` equation is the front-to-back tilt and the `pitch` equation is the tilt through the wheels. The lab's names for those two tilts are pitch and roll. The ports in our reactor are `row` and `pitch`, and the LCD prints those port names: `row` on line 0, `pitch` on line 1. `src/SensorsTiltSolution.lf` wires `Accelerometer` into `Tilt`. We kept `Tilt.lf` for the later labs that need inclination, including the hill climb.
