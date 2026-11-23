# Ashen Firmware

This repository contains ashen tamper sensor firmware, based on [embassy-rs](https://embassy.dev/). This firmware is
mean to run on a microcontroller connected to the `ashen-linux` system to monitor connected physical tamper sensors such
as tamper-sensing meshes, and alert the linux system when a tampering attempt is detected. The linux system and this
firmware communicate through a noise-based [noise](https://noiseprotocol.org/)-based heartbeat protocol.

The linux kernel side of this protocol can be found in the [ashen-linux](https://github.com/yasec-de/ashen-linux)
repository.

On top of the basic heartbeat protocol and at this point simple, GPIO-based tamper sensing, this firmware embeds
[WolfTPM](https://github.com/yasec-de/ashen-linux), a firmware TPM 2.0 (fTPM) implementation. TPM messages are tunneled
to the host besides the heartbeat traffic through the encrypted heartbeat link. On an alarm, `ashen-firmware` notifies
the host, and then deletes the fTPM's root secrets along with the heartbeat link's encryption and authentication keys.

Intende use of the fTPM feature is easy disk encryption on the host without requiring additional drivers or tools apart
from the ashen driver.

## Building and Flashing

### Prerequisites

Software: Install nix and enable flakes
Hardware: A single arduino Uno Q that will run both `ashen-linux` on its linux SoC and `ashen-firmware` on its on-board
STM32 microcontroller.

### Build

```bash
nix build # cross-compiles firmware ELF to result/bin/ashen-fw
```

You can run the pinned rust toolchain manually through a nix dev shell:

```bash
nix develop -c cargo build --release --features embedded --target thumbv7em-none-eabihf
```

### Flash

The easiest way to flash the on-board STM32 on the uno q is through gdb running on your local computer, connected to the
uno q through either adb or ssh. First, you need to start the `arduino-debug` tool that is part of the uno q's stock OS.
This tool wraps [`openocd`](https://openocd.org/) and on launch will expose a `gdb` protocol interface for debugging and
programming on `localhost`. You then forward this gdb port to your development machine that will later be running
openocd.

> [!TIP]
> We aren't using arduino's Zephyr-based core runtime on the STM32, so the arduino-flash script that's shipped by
> arduino with the uno q won't work.


Using SSH to access the uno q:

```shell
$ ssh root@[your uno q] -L 3333:localhost:3333 arduino-debug
```

Using ADB over USB instead:

```shell
$ adb forward tcp:3333 tcp:3333 && adb shell arduino-debug
```

You then call gdb with instructions to connect to that port, and to load the firmware elf binary. 

> [!NOTE]
> On your system, the ARM gdb binary might have a different name. If you can't find an `arm-none-eabi-gdb`, one built
> for `arm-linux-gnueabi` will probably also work.

```shell
$ arm-none-eabi-gdb result/bin/ashen-fw \
  -ex 'target extended-remote localhost:3333' \
  -ex "monitor reset halt" \
  -ex "load" \
  -ex "monitor reset run" \
  -ex "detach"
```

### Development and testing

> [!NOTE]
> The code in this repo was written using an LLM. Most of the tests were LLM-generated with minimal human supervision,
> while the remainder of the firmware was LLM-generated with close human supervision. Proceed reading this code with
> caution. If you find parts of it hard to read, or inelegant, please open an issue [on
> github](https://github.com/yasec-de/ashen-issues/issues). We consider the smells of LLM-generated code such as
> overcomplicated logic, useless comments, and excessive exception handling code issues that we want to correct.
>
> Like in other parts of ashen, all standalone documentation (so docs outside some comments inside the source files) is
> fully human-written. See [our AI policy in our docs](https://ashen.yasec.de/docs/ai-policy/).

```bash
nix develop  # Enter the development shell with all tools, then:

cargo build --release --features embedded --target thumbv7em-none-eabihf
```

You can use gdb as described above not just for flashing but also for full debugging. The MCU has support for hardware
breakpoints and data watchpoints.

For automatic environment activation, use direnv:
```bash
echo "use flake" > .envrc
direnv allow
```

#### Secure Link Tests

A unit test that tests the [noise](https://noiseprotocol.org/)-based heartbeat protocol on the host can be run like
this:

```bash
nix develop -c cargo test
```

#### Formal Verification

The heartbeat link uses [Consistent Overhead Byte Stuffing
(COBS)](https://en.wikipedia.org/wiki/Consistent_Overhead_Byte_Stuffing) as its framing protocol. Since we wrote this
critical part of the protocol machinery ourselves, it has been verified using a formal proof in
[Verus](https://github.com/verus-lang/verus). The same file is compiled into the firmware and checked by the verifier.

The verified properties are:

- `encode` never emits a `0x00` byte, so `0x00` is an unambiguous frame
  delimiter;
- `decode` never panics, never overflows, and never indexes out of bounds on
  zero-free input (the bytes it sees off the wire);
- Decoding and encoding are correct: `decode(encode(s)) == Ok(s)` for every input `s`.

Running the proof:

```bash
nix develop -c verus --crate-type=lib src/cobs.rs
```

## License

GPL-3.0

## AI Policy

See [our AI policy in our docs](https://ashen.yasec.de/docs/ai-policy/).

## Contact

You can find contact information for the project at [https://ashen.yasec.de/](https://ashen.yasec.de/). For
security-related issues, you can contact project lead jaseg at [ashen@yasec.de](mailto:ashen@yasec.de).
