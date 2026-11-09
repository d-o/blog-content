+++
date = '2026-11-08T10:00:00+11:00'
title = 'Zephyr 0: What is a port?'
description = "What an RTOS actually needs to know about a CPU it has never met: a SoC, a board, and a third thing Zephyr never sees, in 141 lines of description files."
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

# What is a port?

*Part 0 of the Zephyr series, which follows the [RISC-V soft core series]({{< relref "riscv-00-why-not-mano" >}}). The core is described in [Measuring the board]({{< relref "riscv-14-measuring-the-board" >}}).*

Why would I choose Zephyr as an operating system for this core? It's a fair question with a simple answer: because I could.

In real life you would almost certainly not bother with an operating system for this sort of core. It has limited RAM, not many peripherals, and a use case that went away once *Knight Rider* was cancelled.

Another question is why [Zephyr](https://zephyrproject.org/), or an RTOS at all? The benefit of using any OS is that application code becomes more portable, and low level APIs like threading, events and IPC are already implemented and tested. There are other options: FreeRTOS, NuttX and more. Compared to FreeRTOS, Zephyr is more *complete*, in the sense that it has a unified driver API built on a device tree, built-in network stacks, and a large, active community. The price, of course, is its footprint: all of that comes at a resource cost. For this exploration I am only interested in whether it is technically possible. Whether it is practical is another question.

My soft core had run C I wrote and assembly I wrote. What I wanted to know was whether it could run someone else's operating system, one I had not bent to fit. That is where "port" comes in, and I had used the word for years without being able to say what is in one.

So here is the question this part answers: **what does Zephyr need from a core it has never met?**

I expected the answer to be about the CPU. Instructions, registers, traps. Some of it is. Most of it is not, and the size of the answer surprised me: the hand-written description of the SoC and the board is 13 files and 141 lines. The generated memory map is another 73, and the two drivers, which are the only real C, add about 190.

## Three nouns, not two

Zephyr splits the hardware description into a **SoC** and a **board**. Most ports I had looked at treat those as obvious. For a soft core on an FPGA they are a decision, because the design has a layer that is neither.

| Zephyr | In this project | What it is |
|---|---|---|
| SoC `soc/riscitry/riscitry` | `computer` + `cpu` | the core, its memory, its on-chip peripherals |
| Board `boards/riscitry/alchitry_cu_io` | the Cu + Io top (`riscitry.luc`) | which pins go where, the UART's baud rate, what is the console |
| (nothing) | `machine` | the run/step sequencer and front panel |

The third row is the odd one. `machine` is what lets me stop the CPU, single-step it with a button, slow the clock to a crawl and show the program counter on the seven-segment display. It is what the early parts of the series were about. Zephyr has no concept of it and does not need one. From Zephyr's side there is a processor that is running, a bank of RAM, and a handful of devices at fixed addresses. Whether a human can freeze it is not Zephyr's business.

That turned out to be a useful rule for deciding where things live: **if the kernel would never ask about it, it is not in the port.**

## What Zephyr asks, and where the answer goes

An out-of-tree port is a *module*: a directory with a `zephyr/module.yml` that tells the build where to find your boards, SoCs and devicetree files. Mine is eight lines and three of them are the same idea three times (`board_root`, `soc_root`, `dts_root` all point at `zephyr`).

After that the questions come in a fixed order, and each goes to a particular file.

**What instruction set is this?** `Kconfig.soc` answers it, by selecting options:

```text
config SOC_RISCITRY
	bool
	select RISCV
	select RISCV_ISA_RV32E
	select RISCV_ISA_EXT_ZICSR
	select RISCV_ISA_EXT_ZIFENCEI
	select RISCV_PRIVILEGED
	select INCLUDE_RESET_VECTOR
```

RV32E (the 16-register embedded base), the CSR instructions, machine mode only, no multiply, no atomics, no compressed instructions. The `ZIFENCEI` line is the one I will have to come back to: the core traps `FENCE.I` as an illegal instruction, yet the SoC claims to support it, because the toolchain would not link without it. That is the subject of [Part 2]({{< relref "zephyr-02-it-builds-then-it-does-not-link" >}}).

**What does the CPU look like to the device tree?** The `cpu0` node:

```text
cpu0: cpu@0 {
	device_type = "cpu";
	compatible = "riscv";
	reg = <0>;
	riscv,isa-base = "rv32e";
	riscv,isa-extensions = "zicsr", "zifencei";
	clock-frequency = <25000000>;

	/* mie/mip: timer 7, external 11, platform lines 16 (UART RX)
	 * and 17 (counter). No PLIC. */
	cpu0_intc: interrupt-controller {
		compatible = "riscv,cpu-intc";
		...
	};
};
```

The interesting part is the comment. Most RISC-V SoCs have a PLIC, a separate interrupt controller that routes dozens of sources to the CPU. This core does not. It has the interrupt bits in `mie` and `mip` that the privileged spec defines, plus two local ones I added at 16 and 17. So the interrupt controller *is* the CPU, and the SoC says `NUM_IRQS = 18` and stops there. That is about as small as an interrupt story gets.

**How fast is time?** Two lines in `Kconfig.defconfig` say the machine timer counts at 1 MHz and how many kernel ticks there are per second. Zephyr's whole sense of time rests on those numbers, so a wrong value does not crash anything; it just makes every sleep the wrong length.

**Where is everything?** The memory map lives in a generated file, `riscitry-memmap.dtsi`: the UART, the timer, the counter, the buttons, the switches, the LED dimmer, the display and the RAM, each at its address. I did not write it by hand, and the next part is about why.

**What is the console?** The board file picks the UART and says how fast to run it:

```text
chosen {
	zephyr,sram = &ram;
	zephyr,flash = &flash0;
	zephyr,console = &uart0;
	zephyr,shell-uart = &uart0;
};

&uart0 {
	current-speed = <1000000>;
	status = "okay";
};
```

Notice what the board did *not* have to say. It did not describe the UART's registers, because those belong to the SoC. A board is a choice about which parts of the SoC to use, and how.

## What it takes beyond the files

Description files are not code. Two things are code, and they are small:

- `soc.c` is seven lines and contains no functions. It is a hook kept in case the interrupt vector alignment needs handling, and it has not: the comment says there is nothing to set up, because the RAM is preloaded and the clocks belong to the board.
- The drivers. The UART driver is polling only. The timer is the other one: Zephyr's own RISC-V machine timer driver was enough at first, and I wrote a replacement later for reasons that belong to a later part. These are the real C, and they are the only parts of the port that *do* anything instead of *describe* something.

That is the shape of it. A port is mostly a set of answers to questions, some in the device tree, some in Kconfig, a little in C, and the kernel does the rest.

## The wrong turns, in order

The port did not build the first time. The errors are a fair picture of how Zephyr thinks about what a port is, so here they are in the order I met them:

1. The module root must hold `zephyr/module.yml`. Without it the build does not know your boards exist.
2. `board.yml` needs a `boards:` list. A bare name is not enough.
3. A binding description containing `: ` broke the YAML. The text was fine in English and not in YAML.
4. The `cpu` node requires `riscv,isa-base`. The tool will not guess your instruction set.
5. A vendor prefix has to be registered before a compatible string can use it.
6. `SOC_LINKER_SCRIPT` was unset. The SoC has to say which linker script to use, even if the answer is "the generic RISC-V one".

None of these is about the CPU. They are all about Zephyr not being willing to guess, which is a good property in an RTOS and an annoying one on the first day.

The seventh error was different. It was a *link* failure, and it was about the CPU, not about configuration. That is [Part 2]({{< relref "zephyr-02-it-builds-then-it-does-not-link" >}}).

## What this says about the core

Reading the port back, the core has to answer only a short list of questions, and it already had answers to nearly all of them before I started:

- an instruction set the toolchain can target;
- a way to take a trap and return from it (`mtvec`, `mepc`, `mret`);
- a timer that can raise an interrupt;
- RAM, and a place for the reset vector;
- one device to print on.

Each of those had been built and tested for its own sake in the main series. The one that had to change was the RAM, which had to double in size, and that is [a later part]({{< relref "zephyr-03-the-rest-of-the-bram" >}}). The rest worked as they were, which is encouraging, and I will try not to read too much into it. It says the core was **complete enough for this kernel to ask its questions**. It does not say the answers were *right*, and the rest of the series is about the places they were not.

Next: [how the memory map got into the device tree]({{< relref "zephyr-01-one-file-for-addresses" >}}) without anyone typing an address twice.
