+++
date = '2026-11-08T11:00:00+11:00'
title = 'Zephyr 1: One file for addresses'
description = "A memory map defined once in a Python file feeds the hardware, the simulator, the test programs, the C build and now Zephyr's device tree. What that does and does not guarantee."
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

# One file for addresses

*Part 1 of the Zephyr series. Previously: [What is a port?]({{< relref "zephyr-00-what-is-a-port" >}})*

Zephyr wants to be told where every device lives, in a text file, as numbers. That sounds like the easiest part of a port. You write down the UART's address and you move on.

The trouble is that by this point the UART's address already lived in several places, and the device tree would have been another one. This part is about why that is dangerous, what I did instead, and, because it is easy to oversell, what the fix does *not* cover.

## The same number, five times

An address is information the whole project shares. The hardware decodes it, the simulator models it, the test programs use it, the C build checks against it, and now Zephyr needs it. If each of those holds its own copy, you get the worst kind of bug: everything passes, because every tool agrees with *itself*, and the board does something else.

So early on, before Zephyr was anywhere near, I moved the addresses into one Python file, `tools/memmap.py`. It begins with a docstring that says what it is for:

> The memory map: the one place addresses are defined.

and then it is just constants:

```python
RAM_BASE = 0x0000_0000
RAM_BYTES = 16384             # 4096 words: all 32 BRAMs (main_ram), code + data

UART_BASE = 0x8000_0000       # 16-byte window
UART_TXDATA = UART_BASE + 0   # write: send byte lane 0 (bits 7:0); dropped if STATUS.busy
UART_STATUS = UART_BASE + 4   # read: bit 0 = tx busy, bit 1 = rx valid, bit 2 = rx overrun
UART_RXDATA = UART_BASE + 8   # read: bits 7:0 = the received byte, bit 8 = valid; the read pops it
UART_BYTES = 16
```

Nothing clever. The register layout is in the comments next to the numbers, which is where I look for it anyway. (Comments trimmed here.) The script also asserts that each window is aligned to its own size, so a typo that would put a device across a boundary stops the generator.

Five tools read this file:

| Reader | What it does with the map |
|---|---|
| `gen_cpu_tb.py` | writes `source/computer/memory_map.luc`, the constants the hardware decodes |
| `rv32sim.py` | the instruction-level simulator: which addresses are RAM, which are devices, everything else raises |
| `programs.py` | the test programs, which use the map's addresses (the end of the RAM, for one) |
| `cbuild.py` | refuses a C image that does not fit in the RAM |
| `gen_dts.py` | writes the Zephyr device tree include |

The last row is the new one. Writing it took about 130 lines, most of them the template for the devices.

## What the hardware sees

The generated Lucid file is short. The RAM line is derived, not typed:

```text
const RAM_BASE = 32h00000000
const RAM_ADDR_BITS = 14       // byte address bits inside the RAM
const RAM_WORDS = 4096
const UART_BASE = 32h80000000    // TXDATA at +0, STATUS at +4, RXDATA at +8
const UART_ADDR_BITS = 4
```

`RAM_ADDR_BITS` and `RAM_WORDS` come from `RAM_BYTES`. The RAM was 1 KB when I wrote the map and is 16 KB now. On the map's side each of those changes was one number in one file, and the decode, the simulators and the device tree followed. (Making the RAM *hardware* that big was a different story, and it has [a part of its own]({{< relref "zephyr-03-the-rest-of-the-bram" >}}).)

## What Zephyr sees

`gen_dts.py` turns the same constants into a device tree include. This is the UART node as it comes out:

```text
/* TXDATA +0, STATUS +4, RXDATA +8 (the read pops). RX interrupt = platform line 0. */
uart0: serial@80000000 {
	compatible = "riscitry,uart";
	reg = <0x80000000 0x10>;
	interrupts-extended = <&cpu0_intc 16>;
	status = "disabled";
};
```

`0x80000000` is `UART_BASE` and `0x10` is `UART_BYTES`, run through a format string. The file begins with a banner saying it is generated and not to edit it, and `gen_dts.py --check` exits non-zero if the file on disk is not what the script would write now. I ran it today and it says `up to date`.

The same trick pays off in a less obvious place. Later in the series the RAM gets split in two, a fake "flash" for code and a RAM for data (that is [a whole part of its own]({{< relref "zephyr-08-the-reset-that-crashed-it" >}})). The split is a single number, `ROM_BYTES`, and the RAM node is computed from it:

```text
ram: memory@2800 {
	device_type = "memory";
	reg = <0x2800 0x1800>;
};
```

`0x2800` is `RAM_BASE + ROM_BYTES` and `0x1800` is `RAM_BYTES - ROM_BYTES`. Nobody had to subtract anything by hand, which is the sort of arithmetic I get wrong at midnight.

## What "cannot drift" does not mean

I wrote in my notes that the point of the generator is that "hardware and devicetree cannot drift". Having written the generator, I would now say that more carefully, because it is two claims.

**The device tree matches `memmap.py`.** That one is true by construction, and `--check` guards it.

**`memmap.py` matches the hardware.** That one is only as good as the other link: the Lucid constants are generated from the same file, so the decode and the map agree, and the simulators run the real generated Verilog and agree on the program's output. It is a strong argument but not a proof; the proof has to come from testing. The check that actually closes the loop is the boring one: the Zephyr hello image writes to the UART at the address in the generated file, and the bytes come out. They did in simulation, and they did on the board ([Part 4]({{< relref "zephyr-04-hello-from-the-uart" >}}) and [Part 5]({{< relref "zephyr-05-on-the-board" >}})).

There are also two things the single file does *not* hold:

- **Interrupt lines.** Which device raises which interrupt is a small table at the top of `gen_dts.py`: the timer is 7, the external interrupt 11, the UART receiver 16, the counter 17. That is copied by hand from the hardware (`computer.luc` and `csr_file.luc`). It is the one place where two files have to agree because a person keeps them in step. The script's own docstring says so. It is the obvious next candidate for moving into `memmap.py`.
- **The bindings.** Zephyr wants a small YAML file per device type describing its properties. Those are hand written. They describe *fields*, not addresses, so they do not drift in the same way, but they are not generated either.

## Why the map has no hole

One thing about this file is worth a mention, because it is the reason the map is shaped the way it is. For its first day it did not cover the whole machine: it described the *data bus* only, and the ROM sat on its own instruction bus, so address 0 for the instruction fetch and address 0 for a load were different places. That was an accident of how the first ROM was written, not a decision. I only noticed when I asked what looked like an obvious question, "so the mem map contains no ROM?", and the honest answer was that it did not. The [part about von Neumann by choice]({{< relref "riscv-03-von-neumann-by-choice" >}}) is the long version. The short version is that there is now one address space for code and data, and the docstring at the top of `memmap.py` says so. That is also why the map needs no special case for where the code lives, and why the device tree can describe one RAM for both.

Next: [the build]({{< relref "zephyr-02-it-builds-then-it-does-not-link" >}}) that went through the configure step, compiled every file, and then refused to link.
