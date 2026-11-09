+++
date = '2026-11-08T17:00:00+11:00'
title = 'Zephyr 7: The diet'
description = "A Zephyr image went from 15,696 to 10,880 bytes of RAM. Most of it came from reading the memory report, and the biggest single saving was from one 64-bit divide."
author = 'Dean Sellers'
tags = [
    "zephyr",
    "riscv-cpu",
    "riscv",
    "fpga",
    "rtos",
    "embedded",
    "optimisation"
]
+++

{{< series "zephyr" >}}

# The diet

*Part 7 of the Zephyr series. Previously: [Ticks and sleep]({{< relref "zephyr-06-ticks-and-sleep" >}})*

The tick image worked and it used 15,696 of the 16,384 bytes of RAM. That left 688 bytes. A kernel with no room for a driver is a demo.

The question for this part: **where do the bytes go, and which of them can go away?** The biggest saving turned out to be in code I had not written.

## Read the report first

Zephyr will tell you. A build can print a RAM report and a ROM report, one line per symbol. I read both before I touched a setting. The image was 13,476 bytes of text, 40 of initialised data and 2,176 of zeroed data. The stacks account for most of the last number: 1,792 bytes in three of them.

The biggest named items were these:

| Symbol | Bytes |
|---|---|
| `__udivdi3` | 1,620 |
| `__m_vfprintf` | 1,140 |
| `z_add_timeout` | 504 |
| `z_thread_halt` | 468 |
| `z_riscv_fatal_error` | 408 |
| `__clz_tab` | 256 |

The first line is the one that stood out. `__udivdi3` is a libgcc routine for dividing two 64-bit numbers. This core has no hardware divider, and no 32-bit one either, so every division is a function call. A 64-bit one is a large function. And **nothing in my own code divides anything**. Something in the kernel did, and I had to find it.

## The cheap round: seven options

First I tried the obvious: switch off what the image plainly did not use. Seven settings, such as a 64-bit timeout type, the stack-trace code on exceptions, and the heap. I put them in an overlay and built.

The image fell from 15,696 to **13,936** bytes. That is 1,760 bytes for little thought. I did not test the options one by one, so I cannot say which paid.

I had not yet run that image in the simulator or on the board, so at that point it was a size, not a result.

## The divide hunt

Three ideas had a chance of removing the 64-bit divide and the printf. Here is what each did.

**More ticks, or fewer?** Zephyr's tick rate decides how it converts milliseconds to ticks. At the 10,000 ticks per second I had, converting a sleep in milliseconds needs a divide. At 1,000 per second, the conversion is the identity and the divide goes. I set it to 1,000 and expected a big drop. It dropped by **48 bytes**.

The divide was still there. A different caller kept it alive: the stock timer driver. Its init routine does `mtime() / CYC_PER_TICK`, a 64-bit divide by a constant, and on a 32-bit core with no divider the compiler turns that into the library call. I tried 15,625 ticks per second as well, because 64 cycles per tick is a power of two and a shift is free. It moved the divide into `k_sleep` and the size did not change.

The only fix was to stop using the stock driver.

**Replace the C library?** The other 1.1 KB is `vprintk`, Zephyr's own print routine, and it calls the C library's `vfprintf`. I tried a smaller C library, the minimal one. It was **worse**: 15,488 bytes against 13,936, because the print code it pulls in is bigger. It did make a warning about thread-local storage go away, which suggests the thing selecting that storage is the standard library.

**Shrink the stacks.** The stacks are 1,792 bytes of fixed allowances. I measured how much each actually used, running the image in the instruction simulator for 198 ticks and looking for how far down each stack had been written. The main stack used 212 of 1,024 bytes, the interrupt stack 144 of 512, and the idle stack 124 of 256. I set main to 512 and the interrupt stack to 256, and left idle alone. The image went to **13,168** bytes and still counted the same 198 ticks.

Two honest limits on that. The method looks for the lowest word that is not zero, so a pushed zero could hide a few words. And I did not exercise the fatal-error path, which is the one that uses the most stack. 512 is about 2.4 times what the main stack has used so far, which is not a lot, and future drivers will want more.

## The driver

The stock RISC-V timer driver is general code. Mine only needs one set of registers, so I wrote a replacement in the port: `timer_riscitry.c`. It is derived from the stock one, with all the arithmetic kept to 32 bits. The 64-bit parts that are left are addition, subtraction and compare, which are inline and cost nothing to call.

There is a limit, and the file says so: no wait is longer than three-quarters of 2^32 cycles, about 53 minutes at 1 MHz. A longer sleep wakes early and the kernel re-arms it.

A board-wide setting makes 1,000 ticks per second the default. Together the two changes took the image to **10,880 bytes, 66.4% of the RAM**. I checked with `objdump` that no 64-bit helper is linked at all. Compared with where the series began, that is 4,816 bytes less, and 5.5 KB free. (The next part splits this figure into a ROM half and a RAM half, which is why you will see different numbers there.)

The sleep granularity is now 1 ms. A `k_sleep` of less than that rounds up.

## What I did not do

Printing, the 1,140 bytes, stays. I had an idea for a tiny replacement that handled just strings and decimals, which would save about 1.1 KB. I dropped it. I use `printk` to debug this thing, a smaller one "gets complicated", and 5.5 KB is enough headroom for now. That is a decision, not a technical limit.

## Does it still work?

The trimmed image ran in the Verilog simulator (the 1 ms ticks and the greeting) and failed the `tick` to `uick` mutation as it should. It ran on the board at 500 ms and at about 32 ms. The one-hour soaks in the last part were on a later build of the same image, after the split described in the next part. The 500 ms path was not covered in simulation, because of the simulator's limit on a long `WFI`.

## What this taught me

Looking at the numbers first, instead of toggling settings, is what pointed at the divide. The list of options Zephyr offers is enormous, and *none of the seven I switched off was the answer*. The answer was a division in a function I had not written and had not read.

Next: [a reset, and a stale pointer]({{< relref "zephyr-08-the-reset-that-crashed-it" >}}).
