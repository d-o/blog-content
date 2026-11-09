+++
date = '2026-11-08T15:00:00+11:00'
title = 'Zephyr 5: On the board'
description = "The Hello ran on the real board at the first try. Whether it boots from power-up was a separate question, and the answer needed care because of what block RAM does in its first microseconds."
author = 'Dean Sellers'
tags = [
    "zephyr",
    "riscv-cpu",
    "riscv",
    "fpga",
    "rtos",
    "embedded"
]
+++

{{< series "zephyr" >}}

# On the board

*Part 5 of the Zephyr series. Previously: [Hello from the UART]({{< relref "zephyr-04-hello-from-the-uart" >}})*

The last part ended in simulation, which proves what simulation proves. This one is the board.

The question I expected to be the hard one was "does it print?". It was not. The harder question turned out to be a different one: **does it start when the power comes on?**

## Getting it on there

The image is part of the bitstream. There is no loader and no second memory: the program is the RAM's initial contents, written into the FPGA when it is configured. So "run Zephyr" means "build a bitstream whose RAM starts holding Zephyr", and "stop running Zephyr" means "build another".

I made a temporary edit to the board top-level, selecting the Zephyr hello as the program instead of the boot menu, built it, and tried to load it. There is a wrinkle with the Cu. The `alchitry load` command refuses to load to RAM on this board ("Lattice chips only support programming the Flash"), so the image has to go to the flash chip, and the FPGA reads it from there at start-up.

That has a consequence, and it is the one I mentioned in [the RAM part]({{< relref "zephyr-03-the-rest-of-the-bram" >}}). Flashing replaces the boot menu. There is no way to have both on the board at once, and I had already decided, for size, that Zephyr would not be a menu entry. So for as long as Zephyr was on the board, the boot menu was gone, and bringing it back was a rebuild and a flash.

## The first run

I read the serial port at 1,000,000 baud, straight after the flash. What came back was 46 bytes:

```text
Hello from Zephyr on alchitry_cu_io/riscitry
```

followed by a carriage return and line feed. That is byte-for-byte the string from the simulation. It was the first time any of Zephyr had run on silicon, and it matched.

## Starting from cold

Flash-and-run is not the same thing as a power-up. The board is already warm and the serial port is already open. What a user does is plug it in.

This matters because of something I found in [the part about the board arriving]({{< relref "riscv-13-the-board-arrives" >}}): the block RAM on this chip has an output register that is undefined for a short time after configuration, so a read in the first couple of microseconds can return junk. No simulator has a model of it. The main series fixed it by holding the CPU in reset for a fixed count of clocks after start-up, long enough to cover the measured window with a margin, and that fix is in the design Zephyr runs on. The same part found two awkward facts that shape what I can test: a reflash does not clear that register, and the reset button never reconfigures the chip. **Only a power cycle exercises the window.**

So here is what I tried, and what each try can and cannot tell me.

| Test | Result | What it shows |
|---|---|---|
| Reset button, pressed repeatedly | full 46 bytes each time (gaps from 3.4 s down to 0.4 s) | not a boot loop; restarts cleanly. **Does not test the window.** |
| Reconfigure by flashing | full 46 bytes | the image boots after configuration. **Does not test the window.** |
| USB unplug and replug, machine at full speed | nothing seen | **inconclusive**: the port reopened 4 s after the cable went back, and the greeting was probably already printed |
| USB unplug and replug, machine in slow mode, then switched to fast | full 46 bytes | a power-up from USB reaches the greeting |

Look at the last row carefully, because I nearly overstated it. In slow mode the CPU clock is about 24 Hz, so the boot crawls, and the CPU starts fetching a long time after the block RAM has settled. That row shows a power-up reaches the greeting. It does **not** show the fast-start case, where the CPU begins as soon as the reset hold ends.

The row I would have liked, a full-speed power-up with the port already open, I do not have. It is a gap, not a pass.

## The mtvec worry

One thing I checked by reading rather than running. The core wants its trap vector base aligned to 128 bytes, and Zephyr's trap entry sits at address 0x14. That looks like a conflict.

It is not. The 128-byte rule is for *vectored* mode, where the core adds the cause number to the base. Zephyr runs in direct mode, one entry for everything, and the core passes the address through unchanged in that mode. That is an argument from the source, not a test, and [the part about ticks]({{< relref "zephyr-06-ticks-and-sleep" >}}) is where it was exercised for real.

## What this part says

Simulation told me Zephyr booted on the Verilog. The board told me it booted on the real chip, from reset and from reconfiguration, and reached the greeting from a slow-mode power-up. The table above is the honest extent of what it has told me about a cold start.

The lesson is about what a test covers. "It printed once" and "it starts every time" are different claims, and the second needs a table.

Next: [a clock]({{< relref "zephyr-06-ticks-and-sleep" >}}), so Zephyr can wait.
