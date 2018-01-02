---
layout: page
title:  Part 2 - Getting Something to Boot
date:   2017-12-29 11:07:26 -0500
---

As with any new project, the best way to get started is to copy a bunch of code from somewhere and get something working, then go back and try to understand the code.  I
pulled this first batch of code from [the OSDev wiki](http://wiki.osdev.org/Raspberry_Pi_Bare_Bones), but I am going to post it here and explain each piece.

## boot.S - The kernel entry point

boot.S is going to be the first thing that the hardware executes in our kernel.  This must be done in assembly.  When the hardware loadsthe kernel, it does not set up a C
runtime environment.  It does not even know what the C runtime environment looks like!  This assembly code sets this up so that we can jump to C as soon as possible.
Here is the code:

**boot.S**
```
.section ".text.boot"

.global _start

_start:
    mov sp, #0x8000

    ldr r4, =__bss_start
    ldr r9, =__bss_end
    mov r5, #0
    mov r6, #0
    mov r7, #0
    mov r8, #0
    b       2f

1:
    stmia r4!, {r5-r8}

2:
    cmp r4, r9
    blo 1b

    ldr r3, =kernel_main
    blx r3

halt:
    wfe
    b halt
```

Lets walk through this line by line.

<br>
```
.section ".text.boot"

.globl _start
```
These are notes to the linker.  The first is about where this code belongs in the compiled binary.  In a little bit, we are going to specify where that is.  The second specifies that \_start is a name that should be visible from outside of the assembly file

<br>
```
_start:
    mov sp, #0x8000
```
This is the first instruction of our kernel.  It says that our C stack should start at address 0x8000 and grow downwards.  Why 0x8000?
Well when the hardware loads our kernel in to memory, it does not load it into address 0, but to address 0x8000.  Since runs from 0x8000 and up, our stack can safely run from 0x8000 and down without clobbering our kernel.

<br>
```
    ldr r4, =__bss_start
    ldr r9, =__bss_end
```
This loads the addresses of the start end end of the BSS section into registers.  If you are not familiar with what BSS is, it is where C global variables that are not initialized at compile time are stored.  The C runtime requires that uninitialized global variables are zero, so we must zero out this entire section ourselves.  The symbols `__bss_start` and `__bss_end` are going to be defined later in when we work with the linker, so don't worry about where they come from for now.

<br>
```
    mov r5, #0
    mov r6, #0
    mov r7, #0
    mov r8, #0
    b       2f

1:
    stmia r4!, {r5-r8}

2:
    cmp r4, r9
    blo 1b
```
This code is what zeros out the BSS section.  First it loads 0 into four consecutive registers.  Then it checks whether the address stored in r4 is less than the one in r9.  If it is, then it executes `stmia r4!, {r5-r8}`.  This instruction has a lot going on for anyone not familiar with ARM.  The `stm` instruction stores the second operand into the address contained in the first.  The `ia` suffix on the instruction means *increment after*, or increment the address in r4 to the address after the last address written by the instruction.  The `!` means store that address back in r4, as opposed to throwing it out.  The `{r5-r8}` operand means that `stm` should store the values in the consecutive registers r5,r6,r7,r8 (so 16 bytes) into r4.  So overall, the instruction stores 16 bytes of zeros into the address in r4, then increments that address by 16 bytes.  This loops until r4 is greater than or equal to r9, and the whole BSS section is zeroed out.

<br>
```
    ldr r3, =kernel_main
    blx r3

halt:
    wfe
    b halt
```
This loads the address of the C function called `kernel_main` into a register and jumps to that location.  When the C function returns, it enters the `halt` procedure where it loops forever doing nothing.


## kernel.c - The C code

This file contains the meat of our baby kernel.  The bulk of the code is for setting up the hardware for basic I/O. The I/O is done through the UART hardware, which allows us to send and recieve text data through the serial ports.  The only way to take advantage of this on actual hardware is to get a [USB to TTL serial cable](https://www.adafruit.com/product/954).  Since I don't have one of those cables, I am going interact with the kernel through the VM until we can get more sophisticated I/O like HDMI out and USB keyboard.  Here is the code:

**kernel.c**
``` c
#include <stddef.h>
#include <stdint.h>

static inline void mmio_write(uint32_t reg, uint32_t data)
{
    *(volatile uint32_t*)reg = data;
}

static inline uint32_t mmio_read(uint32_t reg)
{
    return *(volatile uint32_t*)reg;
#}

// Loop <delay> times in a way that the compiler won't optimize away
static inline void delay(int32_t count)
{
    asm volatile("__delay_%=: subs %[count], %[count], #1; bne __delay_%=\n"
            : "=r"(count): [count]"0"(count) : "cc");
}

enum
{
    // The GPIO registers base address.
    GPIO_BASE = 0x3F200000, // for raspi2 & 3, 0x20200000 for raspi1

    GPPUD = (GPIO_BASE + 0x94),
    GPPUDCLK0 = (GPIO_BASE + 0x98),

    // The base address for UART.
    UART0_BASE = 0x3F201000, // for raspi2 & 3, 0x20201000 for raspi1

    UART0_DR     = (UART0_BASE + 0x00),
    UART0_RSRECR = (UART0_BASE + 0x04),
    UART0_FR     = (UART0_BASE + 0x18),
    UART0_ILPR   = (UART0_BASE + 0x20),
    UART0_IBRD   = (UART0_BASE + 0x24),
    UART0_FBRD   = (UART0_BASE + 0x28),
    UART0_LCRH   = (UART0_BASE + 0x2C),
    UART0_CR     = (UART0_BASE + 0x30),
    UART0_IFLS   = (UART0_BASE + 0x34),
    UART0_IMSC   = (UART0_BASE + 0x38),
    UART0_RIS    = (UART0_BASE + 0x3C),
    UART0_MIS    = (UART0_BASE + 0x40),
    UART0_ICR    = (UART0_BASE + 0x44),
    UART0_DMACR  = (UART0_BASE + 0x48),
    UART0_ITCR   = (UART0_BASE + 0x80),
    UART0_ITIP   = (UART0_BASE + 0x84),
    UART0_ITOP   = (UART0_BASE + 0x88),
    UART0_TDR    = (UART0_BASE + 0x8C),
};

void uart_init()
{
    mmio_write(UART0_CR, 0x00000000);

    mmio_write(GPPUD, 0x00000000);
    delay(150);

    mmio_write(GPPUDCLK0, (1 << 14) | (1 << 15));
    delay(150);

    mmio_write(GPPUDCLK0, 0x00000000);

    mmio_write(UART0_ICR, 0x7FF);

    mmio_write(UART0_IBRD, 1);
    mmio_write(UART0_FBRD, 40);

    mmio_write(UART0_LCRH, (1 << 4) | (1 << 5) | (1 << 6));

    mmio_write(UART0_IMSC, (1 << 1) | (1 << 4) | (1 << 5) | (1 << 6) |
            (1 << 7) | (1 << 8) | (1 << 9) | (1 << 10));

    mmio_write(UART0_CR, (1 << 0) | (1 << 8) | (1 << 9));
}

void uart_putc(unsigned char c)
{
    while ( mmio_read(UART0_FR) & (1 << 5) ) { }
    mmio_write(UART0_DR, c);
}

unsigned char uart_getc()
{
    while ( mmio_read(UART0_FR) & (1 << 4) ) { }
    return mmio_read(UART0_DR);
}

void uart_puts(const char* str)
{
    for (size_t i = 0; str[i] != '\0'; i ++)
        uart_putc((unsigned char)str[i]);
}

void kernel_main(uint32_t r0, uint32_t r1, uint32_t atags)
{
    (void) r0;
    (void) r1;
    (void) atags;

    uart_init();
    uart_puts("Hello, kernel World!\r\n");

    while (1) {
        uart_putc(uart_getc());
        uart_putc('\n');
    }
}
```

# Background on the Raspberry Pi hardware - Memory Mapped IO, Peripherals and Registers
Before we break this into pieces, this code requires some background on the Raspberry Pi hardware.  All interactions with hardware occur through *memory mapped IO*.  Each hardware device, also known as a *peripheral* has a specific address in memory that it writes data to and/or reads data from.  The peripheral region of memory starts at 0x20000000 on the Raspberry Pi model 1, and at 0x0x3F000000 on the models 2 and 3.  Each peripheral can be described as an offset from this base address.  In the above code, we see `UART0_BASE = 0x3F201000`.  This means that the UART hardware is mapped to offset 0x201000 from the peripheral base.  This begs the questions: why 0x3F000000? why 0x201000?  The answer is that these addresses are hard wired into the hardware itself.  These are the addresses that the designers of the Raspberry Pi's chip arbitrarily decided.

Each peripheral has a number of 4 byte *registers* through which data can be read from or written to.  These registers are at predefined offsets from the peripheral's base address.  For example, it is quite common for one at least one register to be a control register, where each bit in the register corresponds to a certain behavior that the hardware should have.  Another common register is a write register, where anything written in it gets sent off to the hardware.

Figuring out where all the peripherals are, what registers they have, and how to use them can mostly be found in [the BCM2835 ARM peripheral manual](https://www.raspberrypi.org/app/uploads/2012/02/BCM2835-ARM-Peripherals.pdf).  The BCM2835 is the name of the chipset the Raspberry Pi model 1 uses, and most of the information is good for the model 2 and 3.  This document is not easy to parse, and it is missing quite a bit of information, but it is a good starting point, as well as proof that I am not pulling all these addresses out of thin air.

Now, on to the code

# Peripheral Specification and Basic Read and Write
``` c
static inline void mmio_write(uint32_t reg, uint32_t data)
{
    *(volatile uint32_t*)reg = data;
}

static inline uint32_t mmio_read(uint32_t reg)
{
    return *(volatile uint32_t*)reg;
}

static inline void delay(int32_t count)
{
    asm volatile("__delay_%=: subs %[count], %[count], #1; bne __delay_%=\n"
            : "=r"(count): [count]"0"(count) : "cc");
}

enum
{
    // The GPIO registers base address.
    GPIO_BASE = 0x3F200000, // for raspi2 & 3, 0x20200000 for raspi1

    GPPUD = (GPIO_BASE + 0x94),
    GPPUDCLK0 = (GPIO_BASE + 0x98),

    // The base address for UART.
    UART0_BASE = 0x3F201000, // for raspi2 & 3, 0x20201000 for raspi1

    UART0_DR     = (UART0_BASE + 0x00),
    UART0_RSRECR = (UART0_BASE + 0x04),
    UART0_FR     = (UART0_BASE + 0x18),
    UART0_ILPR   = (UART0_BASE + 0x20),
    UART0_IBRD   = (UART0_BASE + 0x24),
    UART0_FBRD   = (UART0_BASE + 0x28),
    UART0_LCRH   = (UART0_BASE + 0x2C),
    UART0_CR     = (UART0_BASE + 0x30),
    UART0_IFLS   = (UART0_BASE + 0x34),
    UART0_IMSC   = (UART0_BASE + 0x38),
    UART0_RIS    = (UART0_BASE + 0x3C),
    UART0_MIS    = (UART0_BASE + 0x40),
    UART0_ICR    = (UART0_BASE + 0x44),
    UART0_DMACR  = (UART0_BASE + 0x48),
    UART0_ITCR   = (UART0_BASE + 0x80),
    UART0_ITIP   = (UART0_BASE + 0x84),
    UART0_ITOP   = (UART0_BASE + 0x88),
    UART0_TDR    = (UART0_BASE + 0x8C),
};
```
`mmio_write` and `mmio_read` both take as input a register, which is an absolute address that is going to look something like `0x20000000 + peripheral base + register offset`.  Write takes a 4 byte word to write to the register, while read returns whatever 4 byte word was in the register.  `delay` is just a function that busy loops for a while.  It is a very imprecise way of giving the hardware some time to respond to any writes we may have made.

The enum defines the peripheral offset of the GPIO and the UART hardware systems, as well as some of their registers.  Don't get caught up worrying about what each register is, as I am going to explain them as they are used.

# Setting up the Hardware
``` c
void uart_init()
{
    mmio_write(UART0_CR, 0x00000000);

    mmio_write(GPPUD, 0x00000000);
    delay(150);

    mmio_write(GPPUDCLK0, (1 << 14) | (1 << 15));
    delay(150);

    mmio_write(GPPUDCLK0, 0x00000000);

    mmio_write(UART0_ICR, 0x7FF);

    mmio_write(UART0_IBRD, 1);
    mmio_write(UART0_FBRD, 40);

    mmio_write(UART0_LCRH, (1 << 4) | (1 << 5) | (1 << 6));

    mmio_write(UART0_IMSC, (1 << 1) | (1 << 4) | (1 << 5) | (1 << 6) |
            (1 << 7) | (1 << 8) | (1 << 9) | (1 << 10));

    mmio_write(UART0_CR, (1 << 0) | (1 << 8) | (1 << 9));
}

```
This is the `uart_init` function.  It sets up the UART hardware for use.  It consists of just setting come configuration flags in various registers.

<br>

``` c
    mmio_write(UART0_CR, 0x00000000);
```
This line disables all aspects of the UART hardware.  UART0_CR is the UART's Control Register.

<br>
``` c
    mmio_write(GPPUD, 0x00000000);
    delay(150);

    mmio_write(GPPUDCLK0, (1 << 14) | (1 << 15));
    delay(150);

    mmio_write(GPPUDCLK0, 0x00000000);
```
These lines disable pins 14 and 15 of the GPIO.  Writing 0 to GPPUD marks that pins should be disabled.  Writing `(1 << 14) | (1 << 15)` to GPPUDCLK0 marks which pins
should be disabled, and writing 0 to GPPUDCLK0 makes the whole thing take effect.  Since we aren't using the GPIO pins, this part isn't really that important

<br>
``` c
    mmio_write(UART0_ICR, 0x7FF);
```
This line sets all flags in the Interrupt Clear Register.  This has the effect of clearing all pending interrupts from the UART hardware.  What interrupts actually are and how they work have their own section, so I won't talk about that here.

<br>
``` c
    mmio_write(UART0_IBRD, 1);
    mmio_write(UART0_FBRD, 40);
```
This sets the *baud rate* of the connection.  This is essentially the bits-per-second that can go across the serial port.  This code is trying to get a baud rate of 115200.  To set the baud rate, we must calcuate a *baud rate divisor* and plug that into some registers.  The divisor is calcluated by `UART_CLOCK_SPEED/(16 *
         DESIRED_BAUD)`. The integer part of this calculation goes into IBRD the register of the UART. This calculation likely does not yield an integer, and in our case,
it winds up being about 1.67.  This means we store 1 into IBRD and then we must
also must calculate a *fractional baud rate divisor* from the fractional part of the previous calculation using this formula `(.67 * 64) + .5`.
This winds up being about 40, so we set the FBRD register to 40.

<br>
``` c
    mmio_write(UART0_LCRH, (1 << 4) | (1 << 5) | (1 << 6));
```
This writes bit 4, 5, and 6 to the Line control register.  Setting bit 4 means that the UART hardware will hold data in an 8 item deep FIFO, instead of a 1 item deep
register.  Setting 5 and 6 to 1 means that data sent or received will have 8-bit long words.

<br>
``` c
    mmio_write(UART0_IMSC, (1 << 1) | (1 << 4) | (1 << 5) | (1 << 6) |
            (1 << 7) | (1 << 8) | (1 << 9) | (1 << 10));
```
This disables all interrupts from the UART by writing a one to the relevent bits of the Interrupt Mask Set Clear register.

<br>
``` c
    mmio_write(UART0_CR, (1 << 0) | (1 << 8) | (1 << 9));
```
This writes bits 0, 8, and 9 to the control register.  Bit 0 enables the UART hardware, bit 8 enables the ability to receive data, and bit 9 enables the ability to
transmit data.

# Reading and Writing Text
``` c
void uart_putc(unsigned char c)
{
    while ( mmio_read(UART0_FR) & (1 << 5) ) { }
    mmio_write(UART0_DR, c);
}

unsigned char uart_getc()
{
    while ( mmio_read(UART0_FR) & (1 << 4) ) { }
    return mmio_read(UART0_DR);
}

void uart_puts(const char* str)
{
    for (size_t i = 0; str[i] != '\0'; i ++)
        uart_putc((unsigned char)str[i]);
}
```
This code enables reading and writing characters to and from the UART.  FR is the flags register, and it tells us whether the read FIFO has any data for us to read, and
whether the write FIFO can accept any data.  DR is the data register, which is where data is both read from and written to.
the `uart_puts` function just wraps up putc in a loop so we can write whole strings.

# The Heart of the Kernel
``` c
void kernel_main(uint32_t r0, uint32_t r1, uint32_t atags)
{
    // Declare as unused
    (void) r0;
    (void) r1;
    (void) atags;

    uart_init();
    uart_puts("Hello, kernel World!\r\n");

    while (1) {
        uart_putc(uart_getc());
        uart_putc('\n');
    }
}
```
This is the main function for our kernel.  This is where control is transfered to from boot.S.  All it does is call the `init_uart` function, print "Hello, kerel World!",
and print out any character you type.  This is where we will add calls to many other initialization functions.

The arguments to this function look a bit weird.  Normally, the `main` function of C code looks something like `int main(int argc, char ** argv)`, but our `kernel_main` looks nothing like that.  In ARM, the convention is that the first three parameters of a function are passed through registers r0, r1 and r2.  When the bootloader loads our kernel, it also places some information about the hardware and the command line used to run the kernel in memory.  This information is called *atags*, and a pointer to the atags is placed in r2 just before boot.S runs.  So for our `kernel_main`, r0 and r1 are parameters to the function simply by convention, but we don't care about those.  r2 contains the atags pointer, so the third argument to `kernel_main` is the atags pointer.

## linker.ld - Tying the pieces together
For those unfamiliar with the C compilation process, there are, broadly speaking, three main steps.  The first is preproccessing, where all of your `#define` statements are expanded.  The second is compilation to object files, where the individual code files are converted to individual binaries called object files.  The third is linking, where these individual object files are tied together into a single executable.

By default, GCC links your program as if it were user level code.  We need to override the default, because our kernel is not an ordinary user program.  We do this with a linker script.  Here is the linker script we will be using:

**linker.ld**
```
ENTRY(_start)
 
SECTIONS
{
    /* Starts at LOADER_ADDR. */
    . = 08000;
    __start = .;
    __text_start = .;
    .text :
    {
        KEEP(*(.text.boot))
        *(.text)
    }
    . = ALIGN(4096); /* align to page size */
    __text_end = .;
 
    __rodata_start = .;
    .rodata :
    {
        *(.rodata)
    }
    . = ALIGN(4096); /* align to page size */
    __rodata_end = .;
 
    __data_start = .;
    .data :
    {
        *(.data)
    }
    . = ALIGN(4096); /* align to page size */
    __data_end = .;
 
    __bss_start = .;
    .bss :
    {
        bss = .;
        *(.bss)
    }
    . = ALIGN(4096); /* align to page size */
    __bss_end = .;
    __end = .;
}
```
Before we step through this, I want to note a couple things.  First, in a linker script, `.` means *current address*.  You can assign the current address and also assign things to the current address.  Second, I want to review the sections of a C program.  `.text` is where executable code goes. `.rodata` is *read only data*; it is where global constants are placed.  `.data` is where global variables that are initialized at compile time are placed.  `.bss` is where uninitialized global variables are placed.

Now we step through this file:

<br>
```
ENTRY(_start)
```
This declares that the symbol `_start` from boot.S is the entry point to our code.

<br>
```
    . = 08000;
    __start = .;
    __text_start = .;
    .text :
    {
        KEEP(*(.text.boot))
        *(.text)
    }
    . = ALIGN(4096); /* align to page size */
    __text_end = .;
```
This sets the symbols __start, and __text_start to be 0x8000.  It then declares the .text section to start right after that.  The first part of the .text section is .text.boot, where the code from boot.S resides.  KEEP signifies that the linker should not try to optimize out the code in .text.boot, even though it is not referenced anywhere.  The second part of the .text section is all .text sections from all other objects, in any order.  Then declare __text_end to be the next largest address divisible by 4096 after all of the .text is put in.  This rounding to the nearest 4096 is called *page alignment*, and it becomes important when we start working with memory.

<br>
```
    __rodata_start = .;
    .rodata :
    {
        *(.rodata)
    }
    . = ALIGN(4096); /* align to page size */
    __rodata_end = .;
 
    __data_start = .;
    .data :
    {
        *(.data)
    }
    . = ALIGN(4096); /* align to page size */
    __data_end = .;
 
    __bss_start = .;
    .bss :
    {
        bss = .;
        *(.bss)
    }
    . = ALIGN(4096); /* align to page size */
    __bss_end = .;
    __end = .;
```
Similarly, we declare __rodata_start to be the same as __text_end, then we declare the .rodata section, which consists of all the .rodata sections from all the object files, then declare the __rodata_end to be the next page aligned address after the .rodata.  We then repeat this for the .data and .bss sections.

## Compiling and Running
To compile this code for the VM, we must run the following commands:
```
./gcc-arm-none-eabi-X-XXXX-XX-update/bin/arm-none-eabi-gcc -mcpu=cortex-a7 -fpic -ffreestanding -c boot.S -o boot.o
./gcc-arm-none-eabi-X-XXXX-XX-update/bin/arm-none-eabi-gcc -mcpu=cortex-a7 -fpic -ffreestanding -std=gnu99 -c kernel.c -o kernel.o -O2 -Wall -Wextra
./gcc-arm-none-eabi-X-XXXX-XX-update/bin/arm-none-eabi-gcc -T linker.ld -o myos.elf -ffreestanding -O2 -nostdlib boot.o kernel.o
```
The first two commands compile boot.S and kernel.c into object code, respectively.  The second links those object files into an executable elf file.

Lets take a look at those less used gcc options.  `-mcpu=cortex-a7` means that the target ARM cpu is the cortex-a7, which is what the raspberry pi model 2 has, and what our VM emulates.  `-fpic` means create position independent code.  This means that references to any function, variable, or symbol should be done relative to the current instruction, and not by an absolute address.  `-ffreestanding` means that gcc cannot depend on libc being availible at runtime, and that there may not be a `main` function as an entry point.

<br>
To run the code in the VM, execute this command:
```
qemu-system-arm -m 256 -M raspi2 -serial stdio -kernel myos.elf
```
This runs a VM that emulates the raspberry pi model 2 with 256 megabytes of memory.  It is set up to read/write data from/to your normal terminal as if it were connected to the raspberry pi through a serial connection.  It specifies our kernel elf file as the kernel to run in the VM.

After running this, you should see "Hello, kernel World!" in your normal terminal. If you type in your terminal, it should echo every character.

Congratulations! You have just booted your kernel!