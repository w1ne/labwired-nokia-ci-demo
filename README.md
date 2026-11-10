# Nokia 5110 Invaders — firmware validated in CI, no hardware

[![Firmware Tests (LabWired)](https://github.com/w1ne/labwired-nokia-ci-demo/actions/workflows/firmware-test.yml/badge.svg)](https://github.com/w1ne/labwired-nokia-ci-demo/actions/workflows/firmware-test.yml)

A tiny but complete example of **testing embedded firmware in CI against a
deterministic simulator** — no physical board, no hardware-in-the-loop rig.

The firmware is a small "Space Invaders / Breakout" game for an **STM32L476**
(NUCLEO-L476RG) driving a **Nokia 5110 (PCD8544) 84×48 LCD** over SPI, with an
**HC-SR04** ultrasonic sensor steering the paddle. On every push, GitHub Actions
builds the firmware and runs it inside [LabWired](https://app.labwired.com),
which executes Cortex-M4 instructions and models the peripheral registers, then
asserts on what the firmware actually did.

## What the CI proves

The firmware renders into a 504-byte PCD8544 framebuffer kept in RAM at
`0x20000000`. At boot it draws a deterministic splash frame with the player
paddle centred on the bottom three rows. The test
([`tests/paddle.lwtest.yaml`](tests/paddle.lwtest.yaml)) asserts the paddle
pixels are actually lit in the framebuffer:

```yaml
limits:
  max_cycles: 5000000
  max_steps: 5000000
  stop_when_assertions_pass: true
  stop_when_assertions_pass_settle_steps: 100000
assertions:
  - memory_value: { address: 0x200001C9, expected_value: 0xE0, size: 8 } # paddle col 37
  - memory_value: { address: 0x200001CD, expected_value: 0xE0, size: 8 } # paddle col 41
  - memory_value: { address: 0x200001D1, expected_value: 0xE0, size: 8 } # paddle col 45
  - expected_stop_reason: assertions_passed
```

`0xE0` = bits 5, 6, 7 set = rows 45–47 lit (the paddle). If the firmware — or
the simulated silicon — failed to draw it, these bytes would be `0x00` and CI
goes red. That's a real regression guard: this exact assertion would catch a
CPU instruction-decode bug that once dropped the paddle in the simulator.

The test stops after all splash assertions remain true through a 100,000-step
settling window. The five-million-step and cycle limits bound failure. They
do not select the sample point: a fixed sample at that point can land during
a later game redraw, when the framebuffer has been cleared. With Rust 1.95,
the rebuilt firmware passes all four checks at 133,242 steps on both the
released LabWired v0.25.0 CLI and the current Core build.

## Run it locally

```bash
# 1. build the LabWired CLI from the public engine repo
git clone --depth 1 https://github.com/w1ne/labwired-core
toolchain_host="$(rustc -vV | sed -n 's/^host: //p')"
cargo build --release -p labwired-cli --manifest-path labwired-core/Cargo.toml \
  --target "$toolchain_host"
export PATH="$PWD/labwired-core/target/$toolchain_host/release:$PATH"

# 2. build the firmware for the Cortex-M4F target
rustup target add thumbv7em-none-eabihf
cargo build --release

# 3. run the firmware in the simulator and check the assertions
labwired test --script tests/paddle.lwtest.yaml --output-dir out/
```

A passing run prints `status: pass` and writes `out/result.json`,
`out/uart.log`, and `out/junit.xml`.

## Layout

| Path | What it is |
| --- | --- |
| `src/main.rs` | The firmware: PCD8544 driver, HC-SR04 read, game loop |
| `system.yaml` | The virtual board: L476 + PCD8544 on SPI1 + HC-SR04 on GPIO |
| `stm32l476.yaml` | The chip descriptor (memory map + peripherals) |
| `tests/paddle.lwtest.yaml` | The LabWired test script (inputs, limits, assertions) |
| `.github/workflows/firmware-test.yml` | Build firmware → run in sim → JUnit report |

## How it works

LabWired is a deterministic firmware simulator: it executes the real ELF on a
modelled Cortex-M core and models the peripherals (here the SPI-attached
PCD8544 LCD and the GPIO-wired HC-SR04) in software. Because it's deterministic,
the same firmware produces the same result every run — so assertions on memory,
UART, or stop reason are stable in CI. No board, no flashing, no probe.

Learn more: <https://app.labwired.com>

## License

MIT — see [LICENSE](LICENSE).
