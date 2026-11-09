+++
date = '2026-11-08T19:00:00+11:00'
title = 'Zephyr 9: The upgrade that grew 516 bytes'
description = "Zephyr 4.5.0 built against the port with no changes and ran on the board. The image was 516 bytes bigger, and bisecting 11,492 commits found where they went."
author = 'Dean Sellers'
tags = [
    "zephyr",
    "riscv-cpu",
    "riscv",
    "fpga",
    "rtos",
    "embedded",
    "bisect"
]
+++

{{< series "zephyr" >}}

# The upgrade that grew 516 bytes

*Part 9 of the Zephyr series. Previously: [The reset that crashed it]({{< relref "zephyr-08-the-reset-that-crashed-it" >}})*

Zephyr 4.5.0 came out while I was writing this series. The port had been built against a development snapshot, 4.4.99, so the obvious question was whether it still worked on the release, and on the current development head, called `main`.

The question for this part has two halves. The first has a short answer. **Does an out-of-tree port survive an upgrade?** The second is the one I enjoyed: **where did 516 bytes come from?**

## The port, unchanged

I built the same tick sample against 4.5.0 and against upstream `main` without changing a line of the port. To keep my working copy of Zephyr untouched I used two throwaway copies, each a shared clone with a two-line west manifest.

Both built. Both linked. They ran in the Verilog simulator, with the 36-byte greeting and the 60-byte three-ticks case, and on the board for five minutes each at 500 ms:

| Build | Lines | Resets | Gaps | Stalls | Mean tick |
|---|---|---|---|---|---|
| 4.5.0 | 599 | 0 | 0 | 0 | 500.993 ms |
| main | 600 | 0 | 0 | 0 | 500.995 ms |

As a control I rebuilt the 4.4.99 image the same way. It was byte-for-byte the committed image, which is what tells me the method reproduces what I shipped, rather than something that merely looks similar.

The port's only complaint is one warning: my timer driver includes `<zephyr/sys_clock.h>`, which has been deprecated in favour of `<zephyr/sys/clock.h>`. That is a one-line change that I have not made.

So the answer to the first half is yes, for this port, for this sample.

## The bill

The ROM part of the image has a hard ceiling. The XIP split from [the last part]({{< relref "zephyr-08-the-reset-that-crashed-it" >}}) gives it 10,240 bytes, and the build fails if the code outgrows that.

| | 4.4.99 | 4.5.0 | main |
|---|---|---|---|
| ROM | 9,632 B | 10,148 B (+516) | 10,196 B (+564) |
| RAM part | 1,376 B | 1,424 B | 1,424 B |
| Headroom | 608 B | 92 B | 44 B |

The image grew by 516 bytes on the release and 564 on `main`, and the headroom fell from 608 bytes to as little as 44. Nothing in my port changed, so all the growth is Zephyr's.

## Where to look

I started with a symbol-by-symbol comparison of the two builds, using `nm`, to see which functions grew. The scheduler's ready and unready paths were bigger. The fatal-error handlers were bigger, with `z_fatal_error` up 160 bytes and the RISC-V one up 80. There was a new print helper. And `k_sleep` had been restructured into a different function.

That is a list of suspects, not an answer. The release is 11,492 commits past the point my snapshot branched from, and I wanted to know *which* of them cost bytes.

## The sweep

A real `git bisect` wants a good/bad question and a single culprit. Size is neither. It grows in several steps, and I wanted all of them.

So I built the tick image at 25 evenly spaced points along the history, one at a time, and recorded the ROM size at each. Two details made it work. The first is a check at the starting point: the sample at the common ancestor read 9,632 bytes, equal to my control image, so the method is sound. The second is the ordering. Author dates on this history are not in order, so I went by position in the first-parent list, not by timestamp.

One sample failed to build, and it is worth saying why, because it looked like a find. At that commit my timer driver no longer compiled: `conflicting types for 'sys_clock_set_timeout'`. Upstream had briefly dropped one of the function's arguments, then reverted the whole series. The release and `main` both keep the original form, and my driver builds on both. It was not my bug, and it was not a broken upstream commit either. It was a moment when the timer API was in flux.

From the 25 samples I bisected each interval, assuming each one grew in a single step. The steps added up to the measured +516.

| Step | Bytes | What changed |
|---|---|---|
| A | +368 | the fatal-error handler switched from `LOG_ERR` to `EXCEPTION_DUMP` |
| B | +44 | a mutex priority-inheritance fix |
| C | +96 | a restructure of `printk` |
| D, E | +16 each | not bisected; noise level |
| F | −24 | a device is left uninitialised when its init fails |

Step A is 71% of the growth. Before it, fatal-error messages went through the logging system's `LOG_ERR`, which the compiler throws away completely when logging is off, as it is here. After it, the handler uses an exception-dump facility whose plain `printk` fallback is compiled in whenever logging is off. Same words, new path, 368 bytes.

Step B is a correctness fix, and I would not want it back out. C is a printk restructure, which is the same function I decided to keep in [the diet part]({{< relref "zephyr-07-the-diet" >}}).

## A switch, and a trade-off

The facility behind step A has a Kconfig option, `CONFIG_EXCEPTION_DUMP`, default on, and its help text describes size-constrained builds with logging off. I built `main` with it off:

**ROM 8,644 bytes**, from 10,196. That is 988 bytes *below* the 4.4.99 image and about 1.6 KB of headroom, with the RAM part unchanged.

It is not free. Turning it off removes the register and fault dump that the 4.4.99 image had. That dump is the output that showed `mcause: 2` and `mepc: 0` in the simulator during the reset hunt in the last part, and told me where to look.

So the choice is between a smaller image and a crash that explains itself. I have not made it. For now the port stays on 4.4.99.

## What I did not do

- I did not run the reset checks on 4.5.0 or `main`. I have not checked whether the reset fix holds there.
- The soaks were five minutes, not an hour.
- I did not capture the greeting on the board for either new image, because my capture opened after boot. The simulator has it.
- The bisect assumes each interval grew in one step. The steps do sum to the measured total, which is a check, not a proof.

## What this taught me

An upgrade is a good test of how much of a port is yours. The whole port carried over without a change, and the only thing that moved was a margin I had fixed in advance. The surprise was how a headroom figure I had treated as comfortable, 608 bytes, could lose most of its margin to a change in a handler I do not call.

That is the case for making the size ceiling fail loudly, as the link does. The build did not quietly grow past it. It would have stopped.

What is left is a decision about whether to move at all.

Next: [the scanner that started the whole series]({{< relref "zephyr-10-larson-from-the-main-thread" >}}), on top of Zephyr.
