+++
date = '2026-11-08T16:00:00+11:00'
title = 'Zephyr 6: Ticks and sleep'
description = "k_msleep(500) needs a counter, a compare, an interrupt and a way to wait. The core had all four, and the board says the sleep is 501 milliseconds, not 500."
author = 'Dean Sellers'
tags = [
    "zephyr",
    "riscv-cpu",
    "riscv",
    "fpga",
    "rtos",
    "embedded",
    "interrupts"
]
+++

{{< series "zephyr" >}}

# Ticks and sleep

*Part 6 of the Zephyr series. Previously: [On the board]({{< relref "zephyr-05-on-the-board" >}})*

A kernel that cannot sleep is a loop with ambitions. The first thing that makes Zephyr an operating system rather than a library is that you can tell a thread to wait 500 milliseconds and the CPU goes and does something else.

So the question for this part: **what does `k_msleep(500)` need from the hardware?**

## Four things, all already built

It needs a counter that goes up at a known rate, a compare register that raises an interrupt when the counter reaches it, a CPU that takes that interrupt, and a way for the CPU to do nothing until then.

I built all four in the main series, for their own sake: the machine timer in [Time and interrupts]({{< relref "riscv-06-time-and-interrupts" >}}), and `WFI`, which stops the core until an interrupt arrives, in [Sleeping]({{< relref "riscv-08-sleeping" >}}). RISC-V names the counter `mtime` and the compare `mtimecmp`. Mine count at 1 MHz.

Zephyr runs *tickless* here, which means it does not interrupt every millisecond just to check. It works out when the next thing is due and asks the timer driver for exactly that wake-up. The driver writes the time into `mtimecmp`. With nothing else to run, the kernel's idle thread executes `WFI`. When `mtime` reaches `mtimecmp`, the core wakes, takes the timer interrupt (cause 7), and the driver tells the kernel how many ticks have passed.

There is nothing between the timer and the CPU. No interrupt controller routes it, because this core does not have one, and [Part 0]({{< relref "zephyr-00-what-is-a-port" >}}) explained why that is a smaller job than it sounds. The timer interrupt is a bit in a CPU register.

## Sleeping, as a test

The sample is short:

```c
int main(void)
{
	uint32_t ms = sys_read32(SWITCHES) & 0xFF;

	if (ms == 0) {
		ms = 500;
	}
	printk("tick up on %s\n", CONFIG_BOARD_TARGET);
	for (unsigned int n = 1; ; n++) {
		k_msleep(ms);
		printk("tick %u\n", n);
	}
}
```

The period is set by the middle bank of DIP switches, in milliseconds, and 500 when they read zero. The switches are there for the simulator's sake. Simulating half a second of a 25 MHz core takes a very long time, so the test asks for 1 ms instead.

The code reads the switches **once**, at the start of `main`, so changing one has no effect until the core restarts.

## In simulation

The instruction-level simulator gave four ticks at about 500 ms spacing before it gave up. It gives up on any `WFI` longer than a million clocks, so the 500 ms case there only checks the greeting. That is a limit of the simulator, not a fault in the core, and I do not count it as a check of the 500 ms path.

The Verilog run does the 1 ms case: the greeting, then `tick 1`, `tick 2`, `tick 3`, 60 bytes in all. To confirm it can fail, I changed `tick` to `uick` in a scratch copy of the built design. All four lines failed.

## On the board

The first board run was not at 500 ms. The lines came about every 31 ms: the DIPs were not at zero, and the middle bank happened to read about 31. So the first board result was that the switch worked.

I let that run as a soak. Three hours, the host reading every line:

- 345,020 lines, 4,485,262 bytes;
- 0 resets, 0 stalls;
- a mean of 31.300 ms between ticks, steady to within 0.03 ms from minute to minute.

There were nine `gap` lines in the log, all within the same two seconds, and they were my doing: I ran a quick self-test of the logger on the serial port while the soak was reading it. Two readers on one port split the bytes between them. The board never missed a tick, and that is why I now never open the port twice.

The 31.3 ms is not 31. This run used Zephyr's stock timer driver, which ticks at 10 kHz, and the section below on 501 ms explains why that matters.

Then I set the DIPs to zero, power-cycled, and watched the real thing. 120 ticks in 60 seconds, a mean of **500.98 ms**, none missed. The seven-segment display showed the program counter sitting at 0x3F4, the idle `WFI`, and the sleep and execute lights flashed at the tick rate. That was the part I enjoyed: you can *see* the CPU asleep, with the same front panel that began this whole series.

Two one-hour soaks at 500 ms followed, the second with the host kept awake so it watched every line. That one finished with 7,187 lines, 0 resets, 0 gaps, 0 stalls, and an overall mean of **501.004 ms**.

## Why 501, not 500

It is consistent, so it is not drift. The mean is the same from soak to soak, within a few hundredths of a millisecond, and the tick count tracks the wall clock for the whole hour. The source explains it. In `kernel/timeout.c`, when a thread sleeps, the kernel works out the wake-up time like this:

```c
dticks = timeout.ticks + ticks_elapsed +
	 (this_cpu_announcing() ? 0 : 1);
```

The `+ 1` is the round-up: a relative sleep is rounded up to the next tick so that you never wake early. My `main` is an ordinary thread, not the timer interrupt, so it gets the extra tick every time. At 1,000 ticks per second that is 1 ms, and the period is 501 ticks.

The other half of my first guess, the time to print, turns out not to add anything. The wake-up is aligned to the tick grid, and the line takes about 0.1 ms to send at 1 Mbaud, which finishes inside the same 1 ms tick. The next sleep starts from the same place every time, so the print is absorbed. The measured 501.004 ms against a predicted 501.000 supports that. This is my reading of the source, checked against the numbers, not a separate measurement.

It also explains the earlier 31.3 ms. At 10,000 ticks per second a tick is 0.1 ms, about the time the print takes, so the print spans a tick boundary and adds one or two ticks on top of the round-up. That fits the roughly 0.3 ms of extra per period I saw with the stock driver, and its disappearance after the move to 1 ms ticks. I did not record the exact DIP value for that run, so I can state the fit but not prove it.

## What I would not claim

- The first soak was supposed to run an hour, but the host suspended after about 15 minutes. The board's own tick counter shows it ticked on for about 99 minutes with no reset. That is a statement about the board, inferred from a counter, not something I watched.
- The soaks have no resets in them, and I have not soaked a short period on the newer timer driver. Resets turned out to matter.

Next: [why the image got smaller]({{< relref "zephyr-07-the-diet" >}}), and then [why a reset made it crash]({{< relref "zephyr-08-the-reset-that-crashed-it" >}}).
