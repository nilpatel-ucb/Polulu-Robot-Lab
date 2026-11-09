# Pololu 3pi+ 2040, programmed in Lingua Franca

Bare-metal robot programs for the [Pololu 3pi+ 2040](https://www.pololu.com/docs/0J86), written in [Lingua Franca](https://www.lf-lang.org/) and compiled to C for the RP2040. The sequence covers GPIO and interrupts, then closed-loop driving: wheel odometry, a square path, bump avoidance, line sensing, and a hill climb that uses the IMU.

This is my record of the [LF Embedded Lab](https://www.lf-lang.org/lf-embedded-lab/). It starts from the course RP2040 template (preserved on `template`) and keeps each milestone on its own branch. `main` is the finished set of programs.

The working copy was cloned from the shared lab repository [`KrithinKamineni/lf-3pi-Krithin-Nil`](https://github.com/KrithinKamineni/lf-3pi-Krithin-Nil). Krithin Kamineni created that repository; the lab programs in this history are my commits.

## What the robot does

The 3pi+ 2040 is a two-wheel robot with an LCD, bumpers, quadrature encoders, a five-sensor reflectance array, and an IMU, all on a Raspberry Pi RP2040. Each exercise is a Lingua Franca program under `src/`. Shared reactors for the display, motors, encoders, line sensors, bumpers, and IMU live in `src/lib/`.

Behavior is expressed as reactor modes. A drive program, for example, starts in `DRIVING`, commands the motors, and switches to `TURNING` when an encoder or the gyro says the current leg is finished.

| Program | What it does |
| --- | --- |
| `SensorsTiltSolution.lf` | Reads the accelerometer and prints roll and pitch on the LCD. |
| `PeripheralsDirectSolution.lf` | Blinks the LED by writing the RP2040 SIO GPIO registers directly. |
| `PeripheralsButtonSolution.lf` | Polls the button by reading the GPIO input register. |
| `InterruptActionSolution.lf` | Reports a button edge and the logical time it happened. |
| `InterruptDeboundedSolution.lf` | Ignores edges closer together than 200 ms. |
| `InterruptModalSolution.lf` | Services the button from a GPIO interrupt and toggles a count-up / count-down mode. |
| `RobotDriveSolution.lf` | Alternates `STOPPED` and `DRIVING`, and shows the mode on the LCD. |
| `RobotEncoderSolution.lf` | Converts each wheel's encoder angle to distance (`Δangle / 360 × π × 3.175 cm`) and displays both sides. |
| `RobotSquareSolution.lf` | Drives 50 cm, pivots 90° using the gyro, and repeats, so the path is a square. |
| `RobotAvoidSolution.lf` | Drives until a bumper hits, then backs up and turns. |
| `HillLineDetectSolution.lf` | Calibrates the reflectance array and labels the left, center, and right sensors as on or off the line. |
| `HillLineAvoidSolution.lf` | Calibrates, drives, backs away when it sees the boundary line, then turns and continues. |
| `HillClimbSolution.lf` | Climbs using pitch and roll from the IMU, and turns around once it is over the crest. |
| `lab5_template.lf` | A software timer on top of one RP2040 hardware alarm: callbacks sit in a time-sorted linked list, with one-shot and repeating timers and cancellation. |
| `ToolsBlinkSolution.lf`, `ToolsLEDSolution.lf`, `ToolsPrintSolution.lf` | Small LED and `printf` programs used to check the toolchain and the timer reactions. |

## Branches

The history is linear. Each lab branch is the commit where that stage was finished, so checking one out shows the code as it stood then, not a rewritten snapshot.

| Branch | Stage |
| --- | --- |
| `template` | Course template, before any lab solutions. |
| `lab1` | Tilt sensing on the LCD. |
| `lab2` | Direct GPIO, polled buttons, debouncing, and interrupt-driven modal control. |
| `lab3` | Driving, odometry, the square, bump avoidance, and line detect / avoid. |
| `lab4` | Hill climb. |
| `lab5` | Virtual timers on the RP2040 alarm. Same robot programs as `main`, without this write-up. |
| `main` | Finished programs and this README. |

## Layout

```
src/                 Lingua Franca programs, one file per exercise
src/lib/             Display, motors, encoders, IMU, line sensors, bumpers
src/test/            Motor tests aimed at the Pico emulator
test/                Wokwi-based Pico emulator and its Node runner
pico-sdk/            Raspberry Pi Pico SDK (submodule)
lf-packages/pololu-3pi-c/   Pololu 3pi support library (submodule)
```

`Lingo.toml` is the package file the Lingua Franca tools read. The default app entry is `HelloPico`; pass a file to `lfc` to build any other program.

## Build and flash

The robot has to be in BOOTSEL mode first: hold the B button while pressing reset, and it mounts as a USB drive.

These commands assume the [Lingua Franca compiler](https://www.lf-lang.org/docs/handbook/getting-started) (`lfc`) and `picotool` are on your path. `nix develop` (or `nix-shell`) provides `picotool`, the Arm toolchain, CMake, OpenOCD, and Node. It does not install `lfc`; install that separately.

```bash
git submodule update --init --recursive
lfc src/RobotSquareSolution.lf
picotool load -x bin/RobotSquareSolution.elf
```

`lfc` writes the ELF and UF2 under `bin/`. `picotool load -x` flashes the ELF and starts it. Loading the UF2 from the BOOTSEL drive also works.

The Nix shell sets `PICO_SDK_PATH`. Outside Nix, export that variable to the `pico-sdk` checkout before compiling.

## Emulator

`test/` runs a [Wokwi](https://wokwi.com) emulator of the Raspberry Pi Pico, which is enough to watch GPIO, UART, and a few other Pico peripherals. It does not model the 3pi+ chassis, so the driving, line, and IMU programs have to run on the robot.

```bash
cd test
npm install
npm start
```

Put the `.hex` file you want to run in `test/`. The runner starts one emulator per hex file and writes a text result beside it. A test is marked failing if it is still running after 10 seconds. A Lingua Franca `timeout` property in the target block stops a program earlier:

```
target C {
  timeout: 5 sec,
}
```

## Credits and license

The RP2040 target, the reactor library layout, and the starter programs come from the [Lingua Franca RP2040 template](https://www.lf-lang.org/lf-embedded-lab/) by Edward A. Lee, Abhi Gundrala, and the Lingua Franca project. The Pico board support is the [Raspberry Pi Pico SDK](https://github.com/raspberrypi/pico-sdk). Robot headers and drivers come from the [Pololu 3pi+ 2040 robot library](https://github.com/pololu/pololu-3pi-2040-robot).

Those upstream pieces stay under their own licenses. This repository as a whole is under the BSD 2-clause license in `LICENSE.txt`, copyright The Regents of the University of California.
