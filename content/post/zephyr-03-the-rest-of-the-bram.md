+++
date = '2026-11-08T13:00:00+11:00'
title = 'Zephyr 3: The rest of the block RAM'
description = "Zephyr's smallest sample did not fit in 8 KB, so the RAM doubled to every block RAM on the chip. The hardware change was easy. The build tool ran out of memory."
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

# The rest of the block RAM

*Part 3 of the Zephyr series. Previously: [It builds, then it does not link]({{< relref "zephyr-02-it-builds-then-it-does-not-link" >}})*

Once the link worked, the first real question was a boring one: how big is it? A kernel has to live somewhere, and this core's only program store is its RAM. Code, constants, data and stacks all share one address space and, as the [von Neumann part]({{< relref "riscv-03-von-neumann-by-choice" >}}) explained, one memory.

The RAM was 8 KB. The stock `samples/basic/minimal` image, the smallest thing Zephyr ships that does anything, came out at **18,480 bytes**. That fits neither 8 KB nor the 16 KB I was about to build.

So this part is about a decision with an obvious answer, and the place where the obvious answer broke.

## What fits

Zephyr is configurable down to a surprising level, and I trimmed the sample hard. The trimmed `minimal` sample fit in 8 KB; the commit message from that day says 7,216 bytes.

That was not the sample I wanted. `minimal` does nothing you can see. What I wanted was `hello`, which prints a line on the UART, and that needs the console, printk and a UART driver. With the same trimming it came to **14,912 bytes**. That is more than 8 KB, and it still left only 1,472 bytes of a 16 KB RAM.

So the RAM had to double. There was nowhere else to put it.

## Why 16 KB is the ceiling

The chip on the Cu is an iCE40 HX8K, and the RAM is built from its block RAMs. The count is easy to remember because it is the whole supply: 32 of them. The 8 KB RAM used 16. The 16 KB RAM uses all 32, and the build report says so (`SB_RAM40_4K 32` in the check script, `32/32 BRAM` in the commit message).

A bigger memory would have to live somewhere else on the board, and I do not have anywhere else. 16 KB is not a choice. It is what the chip has.

## The change that was easy

On the map's side, the change was one number in `memmap.py` (`RAM_BYTES`), which is the whole point of [the previous part]({{< relref "zephyr-01-one-file-for-addresses" >}}). The generated address-decode constants, the simulators and the device tree followed.

The RAM itself is a Verilog file, generated for the block RAMs. The first version I wrote for the larger size was a natural extension of the existing one: four arrays of bytes, one per byte lane, each 4096 deep.

It never got as far as synthesis. Alchitry Labs, the IDE and build tool for this board, ran out of heap and stopped. The cause is in the commit message: Alchitry Labs *unrolls the init loop while parsing*, and it copies the syntax trees as it does. A loop of 4096 words over four lanes is a lot of syntax tree. The old 8 KB tree built fine with the same structure, so what broke was the size, not a mistake in the Verilog.

The fix was in the Verilog: **one 32-bit array of 4096 words, with byte-lane slice writes instead of four byte arrays.** The same behaviour with a quarter of the loop. It built.

The result: 6,326 of 7,680 logic cells, all 32 block RAMs, timing at 32.0 MHz against a 25 MHz target, and no warnings. The 8 KB build it replaced was 6,277 cells and 28.5 MHz.

## Checking I had not broken anything

A RAM rewrite is the kind of change that can fail in a way nothing near it notices, so it gets the full set of checks.

- The Yosys simulation sweep: 36 of 36 cases passed straight after the change.
- `check_main_ram.py`, which synthesises the RAM and compares it against the model. That script takes about 19 minutes. I only re-ran the first three programs on the day. The whole thing came back clean overnight, with the tick image included.
- The Alchitry testbench suite, 57 benches, 0 failures. One bench needed a retry on the first attempt: a UART byte mismatch (`got 30 want 32`) that has shown up before when two simulator instances run side by side. It passed on retry. I record it as a flake I have seen, not as a failure I have explained.

Nothing in that list is the board.

## The sequel: the same tool, the same wall

I would like to say I learned from this. I did not, quite. A few hours later I built the timer test. My first design had two images: one with a 500 ms tick for the board and one with a 1 ms tick for the simulator, plus the old hello. The program is baked into the bitstream as RAM initialisation, which the tool turns into a long list of statements, and every extra image lengthened the list.

At 12,600 RAM initialisation statements Alchitry Labs ran out of memory again, in its expression parser. Setting `JAVA_TOOL_OPTIONS=-Xmx6g` did not help at 9,300, either: the launcher either ignores it or it does not matter. What worked was making the design smaller. There is one image, the tick period comes from the DIP switches, and hello is no longer a separate program in the list. That one built at 6,100 statements.

The more expensive part was a mistake of mine. I read the failed build as "still running". It was not. The Java process had hung and was holding the project lock, and the next build refused to start with "open in another instance of Alchitry Labs". It was a stuck process I had not killed, and it had been that way for hours.

The lesson I took from it, which I now check every time: a backgrounded `alchitry build` that prints an out-of-memory error may not exit, so look for the process before you believe it is working.

## What the RAM bought, and what it did not

With 16 KB, Zephyr's hello fits. It does not leave room for anything else. The tick image, with a timer and sleep, came to 15,696 of 16,384 bytes, 688 bytes of slack, which is not a place to add a driver.

That is also why Zephyr is not a boot-menu entry. The boot menu's programs take about 4.9 KB, and Zephyr takes 15.7 KB. Together they come to roughly 20 KB, and the RAM is 16 KB. So Zephyr is a separate flashed build and the menu is gone from the board while it runs.

Next: [a Zephyr image that says hello]({{< relref "zephyr-04-hello-from-the-uart" >}}) through the UART, first in simulation and then on the board.
