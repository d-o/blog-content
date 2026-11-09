+++
date = '2026-11-08T14:00:00+11:00'
title = 'Zephyr 4: Hello from the UART'
description = "A 78-line serial driver, a one-line sample, and the question of how to trust a simulated 'Hello' when the simulator that checks it is the one that predicted it."
author = 'Dean Sellers'
tags = [
    "zephyr",
    "riscv-cpu",
    "riscv",
    "fpga",
    "rtos",
    "embedded",
    "testing"
]
+++

{{< series "zephyr" >}}

# Hello from the UART

*Part 4 of the Zephyr series. Previously: [The rest of the block RAM]({{< relref "zephyr-03-the-rest-of-the-bram" >}})*

With a build that links and a RAM that holds it, the kernel needs a way to say something. Everything else in the port is invisible. A UART is the first part you can *see*.

This part has two halves. The first is the driver, which is small. The second is the harder question: when a simulation prints "Hello", why should you believe it?

## The driver

My UART is the one from the main series. It has three registers:

| Offset | Register | |
|---|---|---|
| +0 | TXDATA | write a byte to send it; dropped if the transmitter is busy |
| +4 | STATUS | bit 0 transmit busy, bit 1 receive valid, bit 2 receive overrun |
| +8 | RXDATA | the received byte, with bit 8 as a valid flag; the read pops it |

The Zephyr driver is `uart_riscitry.c`, 78 lines, and it implements the three functions of Zephyr's polling UART API. Sending is the whole point of this part:

```c
static void uart_riscitry_poll_out(const struct device *dev, unsigned char c)
{
	const struct uart_riscitry_config *cfg = dev->config;

	while (sys_read32(cfg->base + UART_STATUS) & STATUS_TX_BUSY) {
	}
	sys_write32(c, cfg->base + UART_TXDATA);
}
```

Wait until the transmitter will take a byte, then write it. The rule that makes this correct is in the hardware: a write while busy is *dropped*, not queued, so the loop is not an optimisation. Without it, bytes written back to back would be lost.

The other two functions, `poll_in` and `err_check`, read the receive register and clear the overrun bit. The driver does not use interrupts. That is a later job, and the console does not need one.

Where the driver gets its address is the part of the port I described in [Part 1]({{< relref "zephyr-01-one-file-for-addresses" >}}): `DT_INST_REG_ADDR(n)` reads it from the generated device tree. There is no address in the C file.

## The sample

The program is the shortest I have ever written for this machine:

```c
int main(void)
{
	printk("Hello from Zephyr on %s\n", CONFIG_BOARD_TARGET);
	return 0;
}
```

Zephyr sends `printk` to whichever device the board names as `zephyr,console`, which Part 0 showed is the UART. The board name comes from the build, so the output is:

```text
Hello from Zephyr on alchitry_cu_io/riscitry
```

That is 44 characters, plus the carriage return and line feed, which makes **46 bytes** on the wire.

The image uses 14,912 bytes of the 16 KB RAM. Getting it to fit is the story of [the last part]({{< relref "zephyr-03-the-rest-of-the-bram" >}}), and of [a later part]({{< relref "zephyr-07-the-diet" >}}) about making it smaller.

## What actually runs

Before `main`, Zephyr does a lot: it sets up the C runtime, zeros its uninitialised memory, initialises the kernel and its devices, and starts a main thread. When `main` returns, there is nothing left to do, so the kernel idles. The idle thread executes `WFI`, wait for interrupt, and with nothing enabled it waits forever. That is allowed by the RISC-V spec, and the core implements it faithfully: it stops.

In my instruction-level simulator, `rv32sim`, the whole thing from reset to the idle `WFI` takes **5,504 instructions**. That is a small number for an operating system booting.

## Why you should not believe the first "Hello"

Here is the problem. I have two simulators for this core. `rv32sim` is a Python model of the instruction set. The other runs the *actual Verilog* that Alchitry generates from my Lucid, through Yosys, one clock at a time. They are independent implementations, and most of my tests make them agree.

Agreement is also the weakness. If both are wrong in the same way, they still agree.

So I did three things.

**1. The expected string is fixed text.** For this program the check compares the output against a fixed string in the test table, not against whatever the model produced. The 46 bytes have to be exactly those 46 bytes.

**2. The Verilog run is the one that counts.** `xsim_computer.py` runs the generated Verilog, at two settings of its `--tick-div` option (1 and 5). Both gave the 46 bytes.

**3. A mutation.** I changed the string in a scratch copy of the built design, so `Hello` became `Iello`, and ran the check again. It failed. A test that can fail is worth something. A test that has never been seen to fail is a hope.

That is a reasonable level of confidence for a simulated result, and it is where I stopped before the hardware. One more thing I will own: the hello image was later removed from the board's list of built-in programs, because every program in that list is baked into the bitstream and the build tool does not cope with too many (that is also [from the last part]({{< relref "zephyr-03-the-rest-of-the-bram" >}})). The tick image that replaced it prints its own greeting through the same driver, and the same kinds of checks apply to it.

## What this does and does not show

It shows that a stock Zephyr kernel, with a trimmed configuration and my port files, boots on my core, reaches `main`, and writes the right bytes to the right address in the right order.

It does not show anything about a real board. All of this is simulation.

Next: [flashing it]({{< relref "zephyr-05-on-the-board" >}}).
