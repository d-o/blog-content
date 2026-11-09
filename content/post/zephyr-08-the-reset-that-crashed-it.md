+++
date = '2026-11-08T18:00:00+11:00'
title = 'Zephyr 8: The reset that crashed it'
description = "Zephyr ticked for hours, and a press of the reset button killed it. Four wrong guesses later, the answer was 36 bytes of initialised data that a restart never reloaded."
author = 'Dean Sellers'
tags = [
    "zephyr",
    "riscv-cpu",
    "riscv",
    "fpga",
    "rtos",
    "embedded",
    "debugging"
]
+++

{{< series "zephyr" >}}

# The reset that crashed it

*Part 8 of the Zephyr series. Previously: [The diet]({{< relref "zephyr-07-the-diet" >}})*

The image was small, it ticked at 500 ms, and a power cycle brought it back every time. Then I pressed the Cu's reset button, and nothing came back.

The question for this part: **why does a reset not put the machine back where it started?** The answer is not about hardware. It was a fact about how Zephyr starts, which I had not looked into until now.

## The symptom

After the DIPs went to zero and the reset was pressed twice, the serial port gave no bytes for minutes. The display showed the program counter at 0000 and the Cu's LEDs in a pattern I read as "held in reset". A power cycle restored the ticks.

That reading was wrong, and it was the first of several. The guesses, in order, are most of the content of this post.

## Four wrong guesses

**1. The fast clock is off.** The machine has a slow mode for single-stepping, and in slow mode a boot takes minutes. I put the silence down to the clock having been left slow. It had not: fast was on, and I had blamed the wrong thing.

**2. The core is stuck in reset.** A later, cleaner capture killed this one. I did a single press with the capture already open. The ticks ran at 500 ms up to tick 1212, then, about seven seconds after the last one, the greeting `tick up on alchitry_cu_io/riscitry` appeared, and then nothing for 68 seconds. The core had restarted and printed. It was not stuck in reset. And the display showed the program counter *flickering* around 0x1888. A core in reset does not flicker. A core running a loop does.

**3. It was something I had done to the image.** By now I had shrunk the image, written a new timer driver and changed the tick rate. Any of those could have broken a restart, and it was the guess I tested hardest.

**4. The restart wipes its own stack.** Zephyr zeroes its uninitialised data on start, and I wondered whether that zeroing was erasing the stack it was running on. It was not. The stacks sit outside the zeroed region, which I checked in the linker map.

There were also two hardware suspects that I never ruled out directly. The symptom, an illegal instruction at address zero, looked as if the first instruction had been fetched wrongly. A reset in the middle of a RAM write could corrupt address 0, or the first fetch after a reset could misread the block RAM, which I knew has a bad first few microseconds after power-up. The real cause made both unnecessary.

## Reproducing it without the board

The first useful step was to take the board out of it. Another tool of mine, `vsim`, runs the real Verilog with a window, and it can press reset at a chosen moment. At the same reset time that crashed the board, the trimmed image reproduced it. The greeting was cut short at `tick up on alc`, followed by Zephyr's own fatal-error dump:

```text
mcause: 2, Illegal instruction
mepc: 0
mstatus: 1800
```

Illegal instruction, at address zero. And the pre-trim image at that reset time restarted cleanly. That looked like guess 3 confirmed.

I then bisected, adding one change at a time to the old image. The seven size options restarted fine. The smaller stacks restarted fine. The 1,000 ticks-per-second setting with the old driver crashed straight after the reset, with no greeting at all. So one of the three was guilty, and I had a suspect.

## The control that took it away

I did not trust that, because I had only tried one reset moment per image. The trimmed image and the old one differ in where their code sits, which could change which instant of a reset is dangerous.

So I took the **pre-trim image, which was known good**, and moved the reset. At 0.2 s, 0.20013 s and 0.20037 s, it restarted. At 0.20089 s, 0.2011 s, 0.20137 s and 0.2016 s, it died with the same fatal dump.

That is decisive, and it is an uncomfortable result: *any* tick image could die on a reset that landed at the wrong instant. My changes had not caused it. They had only moved which instants were bad. The smaller image probably looked broken because my one test moment happened to fall in a bad part.

(One correction on the way. Two of my runs looked as if they had printed nothing. They had not. I was counting lines with a tool that treats the simulator's output as binary data because of a stray byte in the greeting, and it reported blanks. Use `grep -a`. The conclusion did not change, but I nearly threw out good data.)

## The real cause

The clue was a hunch, and it came from the other half of the project: the boot menu's programs restart fine after a reset, so it had to be something about Zephyr.

The hunch was right. Here is why the menu's C programs survive. A C program has initialised data, such as `int x = 5;`, which lives in the program image. Mine start with a small piece of start-up code, `crt0`, that copies a *pristine* version of that data over the top before `main` runs. After a reset the RAM holds the old run's values, and `crt0` overwrites them.

Zephyr in my build did not do that. Zephyr zeroes its uninitialised data on start. It also copies its initialised data from the image, but **only if it is built to run in place from flash**. Mine was built to run from RAM, so the copy never happened. Thirty-six bytes of initialised data kept their values from the previous run. One of them was `timeout_list`, the head of the kernel's list of pending timeouts.

Now follow a reset. The main thread is almost always asleep in `k_msleep`, because that is what it does. So the pending-timeouts list points at the sleeping thread's timeout node. The core restarts. Zephyr zeroes its uninitialised data, which includes that node. The initialised pointer to it survives. The first time the kernel walks the list, it follows the pointer into zeroed memory and ends up at address 0 with an illegal-instruction trap: `mcause 2`, `mepc 0`. I have not traced exactly why a jump to zero traps, rather than simply running the start-up code again. The fix removes the problem, and the simulations below back the cause.

That also explains the whole pattern of symptoms:

- Why a reset only worked at some instants: it worked when no timeout was pending, which is while the thread is printing or in an interrupt. Roughly one reset time in three, in the control scan.
- Why the first screen showed `pc 0000`: a restart loop, not a held reset.
- Why bisecting the size changes found nothing: the code layout changed which instants hit.
- Why the earlier hello reset worked: one sample at one moment.

## The fix

The fix uses Zephyr's own mechanism. Tell Zephyr the image runs in place from flash, and it will copy the initial data on every start.

There is no flash. But the device tree can lie, and I generate the device tree. So the generator splits the 16 KB: the bottom 10 KB is a fake `flash0` holding code, constants and the pristine copy of the data; the rest is `ram`, for data, zeroed data and stacks. A single number in the memory map file, `ROM_BYTES`, sets the split (the [first part]({{< relref "zephyr-01-one-file-for-addresses" >}}) showed the RAM node computed from it). The cost in memory is 36 bytes. The image fits with 608 bytes to spare in its half.

## Did it work?

I checked it three ways, and gave each a way to fail.

- **Simulation, 16 reset moments**, including every moment that had crashed before: all restarted, with the greeting twice and the ticks from 1.
- **A negative control.** I wrote a scan that resets at 24 different moments. The fixed image: 24 of 24 clean. The pre-fix image: 16 of 24 died. A test that gave the same answer for both would have been worth nothing.
- **The board.** I flashed it and pressed reset ten times. Each press was followed by a greeting and then `tick 1` about half a second later, with no fault dump. The presses were about 0.9 seconds apart, so each reset landed while `main` was asleep in `k_msleep`, the exact condition that killed the old image.

The same scan also confirmed the hunch from the other side: the boot menu's programs restart cleanly at every reset moment I tried, on the entries that print to the serial port.

## What I would not claim

- It is one image on one board. Ten resets is a good sign and not a proof.
- The boot menu entries that only drive LEDs and the display have not been scanned, because the scan reads serial output.
- Two of my first captures "went silent" because of my own listener, not the board. I nearly concluded the board had stopped ticking. The lesson was to check the capture is alive before reading silence as the board's.
- The fix has a price. The ROM half is a fixed ceiling: if the code grows past 10 KB the link fails and I raise the constant. The binary now covers only the ROM half, so everything above it relies on the RAM being zero at power-up, which was already true.

There were other ways to do it, and I ranked six of them. This was the cheapest and used what Zephyr already had.

## What this taught me

A reset is a promise that the machine goes back to a known state. For a program that is only true if the program itself restores whatever the reset did not. The boot menu's C programs do that by design. Zephyr does it only in one configuration, and I had picked the other.

Next: [the same port, on a newer Zephyr]({{< relref "zephyr-09-the-upgrade-that-grew-516-bytes" >}}).
