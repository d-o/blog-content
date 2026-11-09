+++
date = '2026-11-08T12:00:00+11:00'
title = 'Zephyr 2: It builds, then it does not link'
description = "Every file compiled, then the linker said 'i' and 'e' do not mix. The fix was one line that claims an instruction the core traps, and the question is what that line is worth."
author = 'Dean Sellers'
tags = [
    "zephyr",
    "riscv-cpu",
    "riscv",
    "fpga",
    "rtos",
    "embedded",
    "toolchain"
]
+++

{{< series "zephyr" >}}

# It builds, then it does not link

*Part 2 of the Zephyr series. Previously: [One file for addresses]({{< relref "zephyr-01-one-file-for-addresses" >}})*

The first six errors of the port were all configuration. Zephyr was unwilling to guess, I filled in what it wanted, and each time the build got a little further. The [first part]({{< relref "zephyr-00-what-is-a-port" >}}) lists them.

The seventh error was different, because by then everything had compiled. Ninety-odd build steps went past, my own drivers among them, and then the first link failed. A link failure is not a message about *my* files. It is the toolchain telling me that two pieces it was asked to join do not agree about what machine they are for.

So this part answers: **why would a CPU's own libraries refuse to link into its own kernel?**

## The message

I reproduced it for this post by taking the port as it is today and removing the line that fixes it, and its twin in the device tree. These are the errors, as the linker printed them:

```text
ld.bfd: error: .../riscv64-zephyr-elf/lib/libc.a(memcpy.c.o)
  use 16-byte stack aligned but the output use 4-byte stack aligned
ld.bfd: error: .../riscv64-zephyr-elf/lib/libc.a(memcpy.c.o):
  mis-matched ISA string to merge 'i' and 'e'
```

There is one "mis-matched ISA string" and 32 "stack aligned" lines, one for each library object that got pulled in: `memcpy`, `strlen`, the printf and scanf code, and libgcc's divide and bit-count helpers.

Read them as a pair. The *output*, my kernel, is built for RV32E with the `ilp32e` ABI. That ABI keeps the stack 4-byte aligned. The library objects were built for plain RV32I, whose ABI keeps it 16-byte aligned. They cannot be merged, and the linker says so.

The odd part is that I never asked for the 'i' libraries. The linker picked them without telling me.

## Where the libraries come from

A GCC toolchain for RISC-V ships many copies of its libraries, one per combination of architecture string and ABI. These are called multilibs. When you compile with `-march=rv32e_zicsr -mabi=ilp32e`, GCC looks through that list for an exact match and links the library from that directory. If it finds none, it falls back to the default set, which is the one for plain RV32I. The fallback is silent.

So I asked the toolchain what it has. The SDK is 1.0.1, GCC 14.3. These are its RV32E entries:

```text
rv32e_zicsr_zifencei/ilp32e
rv32em_zicsr_zifencei/ilp32e
rv32emc_zicsr_zifencei/ilp32e
rv32emc_zicsr/ilp32e
...
```

The only RV32E entry without the multiply and compressed extensions ends in `zifencei`. The entries that lack `zifencei` carry the compressed extension, which this core does not have, so they cannot match. The one my core needs, `rv32e_zicsr`, is not on the list. Zephyr assembled the architecture string from what the SoC claimed (`rv32e_zicsr`), found no directory called that, and the linker fell back to the wrong library.

`zifencei` is the extension that defines `FENCE.I`, the instruction that tells a core to re-read instructions it may already have fetched. It is what you need after you write code into memory and then want to run it.

## The one-line fix

The SoC's Kconfig selects the extension:

```text
select RISCV_ISA_EXT_ZIFENCEI
```

With that, Zephyr's build adds `_zifencei` to the architecture string, the toolchain finds `rv32e_zicsr_zifencei/ilp32e`, and the link goes through. The device tree says the same thing, `riscv,isa-extensions = "zicsr", "zifencei"`, so the two agree.

That is the whole fix, and it deserves some suspicion, because **my core does not implement `FENCE.I`**.

## What the core does with it

`FENCE` and `FENCE.I` share an opcode. The decoder treats the first as a no-op, since this is an in-order core with no caches and nothing to order. The second is listed among the illegal encodings and traps. There is a test for it: it is one of the illegal-instruction cases that has to land in the trap handler with the right cause.

The decoder has had it on the illegal list since its first testbench. I did not think about it again until the toolchain made me.

So the port says "this core has Zifencei" in two files, and the hardware disagrees.

## Is that a lie?

Sure, you could call it fake news to placate a fussy build system and move on. The other framing is that the architecture string is doing two jobs. It is a statement about what the core can run, and it is also a *key for choosing a library directory*. For `zifencei` I use it only for the second.

That is only safe if nothing ever executes the instruction. I can check that rather than assume it. I disassembled three of the linked images, and none has a `fence.i` in it:

- an earlier tick build;
- the control build on the Zephyr this port was written against (4.4.99);
- a build against upstream main.

All three show the architecture attribute `rv32e2p0_zicsr2p0_zifencei2p0` and zero `fence.i` instructions. That is a check on three builds, not a promise about the next one.

Zephyr does contain `fence.i`. A search of the arch, include, kernel, lib, subsys and drivers directories finds it in a handful of files: cache code for T-Head and VexRiscv cores, one flash driver, and an Andes cache header. None is in this build. A driver, a bootloader or a module that loaded code at run time might add one.

## The real fix

There is another answer, and it is almost certainly better in the long run: make the core accept `FENCE.I` as a no-op. I had already said so: the list of limits in [where it breaks]({{< relref "riscv-11-where-it-breaks" >}}) has "`FENCE.I` traps" and says it would be correct as a no-op. The [von Neumann part]({{< relref "riscv-03-von-neumann-by-choice" >}}) explains why. With no instruction cache, every fetch reads memory fresh, so the instruction has nothing to do and doing nothing is correct. A small decoder change and an edited test would make the claim in the device tree *true* instead of tolerated.

I have not made that change. It touches the hardware and the test suite for the sake of a fussy toolchain, and for now the cheaper answer works. If I ever add code that gets loaded and then run, I need it anyway.

There is a smaller guard that costs nothing: a check on every image that `fence.i` appears nowhere in the disassembly.

## What this one taught me

I started this series expecting the CPU to be the hard part and found that a port is mostly paperwork. This is the first place where the paperwork and the hardware pulled in different directions.

- A toolchain's idea of a CPU is a name in a table. If your core is not in the table you either match an existing entry, with the lies that implies, or build your own libraries.
- The silent fallback is the dangerous part. The failure was loud only because the ABI differs between E and I. For two variants that differed in a way the linker could not see, I would have got a binary that linked and then misbehaved.
- "The core does not need it" and "the core says it has it" are different statements. The port is where you find out which one you made.

Next: [the part where]({{< relref "zephyr-03-the-rest-of-the-bram" >}}) Zephyr needed 18 KB of RAM and I had 8.
