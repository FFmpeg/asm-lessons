[hung-notation-wiki]: https://en.wikipedia.org/wiki/Hungarian_notation "Wikipedia"
[dav-project]: https://www.videolan.org/projects/dav1d.html "Open-source AV1 decoder"
[the-art-asm]: https://artofasm.randallhyde.com "Book website"

# Lesson One

## What is Assembly Language?

Assembly Language is a programming language where you write code that directly corresponds
to the instructions a CPU processes. Human readable Assembly Language is, as the name
suggests, *assembled* into binary data, known as *machine code*, that the CPU can
understand. You might see Assembly Language code referred to as "Assembly" or "ASM" for
short.

The vast majority of Assembly code in FFmpeg is what's known as SIMD (Single Instruction
Multiple Data). SIMD is sometimes referred to as vector programming. This means that a
particular instruction operates on multiple elements of data at the same time. Most
programming languages operate on one data element at a time, known as scalar programming.

As you might have guessed, SIMD lends itself well to processing images, video, and audio
which have lots of data ordered sequentially in memory. There are specialist instructions
available in the CPU to help us process sequential data.

In FFmpeg, you'll see the terms "Assembly function", "SIMD", and "vector(ise)" used
interchangeably. They all refer to the same thing: Writing a function in Assembly Language
by hand to process multiple elements of data in one go. Some projects may also refer to
these as "Assembly kernels".

All of this might sound complicated, but it's important to remember that in FFmpeg, high
schoolers have written Assembly code. As with everything, learning is 50% jargon and 50%
actual learning.

## Why do we write in Assembly Language?

To make multimedia processing fast. It's very common to get a 10x or more speed improvement
from writing Assembly code, which is especially important when wanting to play videos in
real time without stuttering. It also saves energy and extends battery life. It's worth
pointing out that video encode and decode functions are some of the most heavily used
functions on earth, both by end-users and by big companies in their datacentres. So even a
small improvement adds up quickly.

You'll often see, online, people use *intrinsics,* which are C-like functions that map to
Assembly instructions to allow for faster development. In FFmpeg we don't use intrinsics
but instead write Assembly code by hand. This is an area of controversy, but intrinsics are
typically around 10-15% slower than hand-written Assembly (intrinsics supporters would
disagree), depending on the compiler. For FFmpeg, every bit of extra performance helps,
which is why we write in Assembly code directly. There's also an argument that intrinsics
are difficult to read owing to their use of "[Hungarian Notation][hung-notation-wiki]".

You may also see *inline Assembly* (i.e. not using intrinsics) remaining in a few places in
FFmpeg for historical reasons, or in projects like the Linux Kernel because of very
specific use cases there. This is where Assembly code is not in a separate file, but
written inline with C code. The prevailing opinion in projects like FFmpeg is that this
code is hard to read, not widely supported by compilers and unmaintainable.

Lastly, you'll see a lot of self-proclaimed experts online saying none of this is necessary
and the compiler can do all of this "vectorisation" for you. At least for the purpose of
learning, ignore them; recent tests in e.g. [the dav1d project][dav-project] showed around
a 2x speedup from this automatic vectorisation, while the hand-written versions could reach
8x.

### Flavours of Assembly Language

These lessons will focus on x86 64-bit Assembly Language. This is also known as amd64,
although it still works on Intel CPUs. There are other types of Assembly for other CPUs
like ARM and RISC-V and potentially in the future these lessons will be extended to cover
those.

There are two flavours of x86 Assembly syntax that you'll see online: AT&T and Intel. AT&T
Syntax is older and harder to read compared to Intel Syntax. So we will use Intel Syntax.

### Supporting materials

You might be surprised to hear that books or online resources like Stack Overflow are not
particularly helpful as references. This is in part because of our choice to use
handwritten Assembly with Intel Syntax. But also because a lot of online resources are
focused on operating system programming or hardware programming, usually using non-SIMD
code. FFmpeg Assembly is particularly focused on high performance image processing, and as
you'll see it's a particularly unique approach to Assembly programming. That said, it's
easy to understand other Assembly use-cases once you've completed these lessons

Many books go into a lot of computer architecture details before teaching Assembly. This is
fine if that's what you want to learn, but from our standpoint, it's like studying engines
before learning to drive a car. That said, the diagrams in the later parts of "[The Art of
64-bit Assembly][the-art-asm]" book showing SIMD instructions and their behaviour in a visual
form are helpful.

## Registers

Registers are areas in the CPU where data can be processed. CPUs don't operate on memory
directly, but instead data is loaded into registers, processed, and written back to memory.
In Assembly Language, generally, you cannot directly copy data from one memory location to
another without first passing that data through a register.

### General Purpose Registers

The first type of register is what is known as a General Purpose Register (GPR). GPRs are
referred to as general purpose because they can contain either data, in this case up to a
64-bit value, or a memory address (a pointer). A value in a GPR can be processed through
operations like addition, multiplication, shifting, etc.

In most Assembly books, there are whole chapters dedicated to the subtleties of GPRs, the
historical background, etc. This is because GPRs are important when it comes to operating
system programming, reverse engineering, etc. In the Assembly code written in FFmpeg, GPRs
are more like scaffolding and most of the time their complexities are not needed and
abstracted away.

### Vector registers

Vector (SIMD) registers, as the name suggests, contain multiple data elements. There are
various types of vector registers:

*  mm/MMX Registers ( 64-bit sized): historic and not used much any more.
* xmm/XMM Registers (128-bit sized): widely available.
* ymm/YMM Registers (256-bit sized): some complications when using these.
* zmm/ZMM Registers (512-bit sized): limited availability.

Most calculations in video compression and decompression are integer-based so we'll stick
to that.

---

* Here's an example of 16 bytes in an XMM Register:

|a|b|c|d|e|f|g|h|i|j|k|l|m|n|o|p|
|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|

---

* But it could be eight words (16-bit integers):

|a|b|c|d|e|f|g|h|
|:--|:--|:--|:--|:--|:--|:--|:--|

---

* Or four double words (32-bit integers):

|a|b|c|d|
|:--|:--|:--|:--|

---

* Or two quadwords (64-bit integers):

|a|b|
|:--|:--|

---

* To recap:
	* **b**ytes - 8-bit data
	* **w**ords - 16-bit data
	* **d**oublewords - 32-bit data
	* **q**uadwords - 64-bit data
	* **d**ouble **q**uadwords - 128-bit data

> The bold characters will be important later.

## `x86inc.asm` include

You'll see in many examples we include the file `x86inc.asm`. `X86inc.asm` is a lightweight
abstraction layer used in FFmpeg, x264, and dav1d to make an Assembly programmer's life
easier. It helps in many ways, but to begin with, one of the useful things it does is it
labels GPRs, `r0`, `r1`, `r2`. This means you don't have to remember any register names. As
mentioned before, GPRs are generally just scaffolding so this makes life a lot easier.

## A simple scalar ASM snippet

Let's look at a simple (and very much artificial) snippet of scalar ASM (Assembly code that
operates on individual data items, one at a time, within each instruction) to see what's
going on:

```assembly
mov  r0q, 3
inc  r0q
dec  r0q
imul r0q, 5
```

In the first line, the *immediate value* `3` (a value stored directly in the Assembly code
itself as opposed to a value fetched from memory) is being stored into register `r0` as a
quadword. Note that in Intel Syntax, the source operand (the value or location providing
the data, located on the right) is transferred to the destination operand (the location
receiving the data, located on the left), much like the behavior of `memcpy`. You can also
read it as `r0q = 3`, since the order is the same. The `q` suffix of `r0` designates the
register as being used as a quadword. `inc` increments the value so that `r0q` contains `4`,
`dec` decrements the value back to `3`. `imul` multiplies the value by `5`. So at the end,
`r0q` contains `15`.

Note that the human readable instructions such as `mov` and `inc`, which are assembled into
machine code by the assembler, are known as *mnemonics*. You may see online and in books
mnemonics represented with capital letters like `MOV` and `INC` but these are the same as
the *lowercase* versions (i.e. they are not case sensitive). In FFmpeg, we use *lowercase*
mnemonics and keep *UPPERCASE* reserved for macros.

## Understanding a basic vector function

Here's our first SIMD function:

```assembly
%include "x86inc.asm"

SECTION .text

;static void add_values(uint8_t *src, const uint8_t *src2)
INIT_XMM sse2
cglobal add_values, 2, 2, 2, src, src2
    movu  m0, [srcq]
    movu  m1, [src2q]

    paddb m0, m1

    movu  [srcq], m0

    RET
```

#### Let's go through it line by line:

---

```assembly
%include "x86inc.asm"
```

This is a "header" developed in the x264, FFmpeg, and dav1d communities to provide helpers,
predefined names and macros (such as `cglobal`) to simplify writing Assembly.

---

```assembly
SECTION .text
```

This denotes the section where the code you want to execute is placed. This is in contrast
to the `.data` section, where you can put constant data.

---

```assembly
;static void add_values(uint8_t *src, const uint8_t *src2)
INIT_XMM sse2
```

The first line is a comment (the semi-colon `;` in ASM is like `//` in C) showing what the
function argument looks like in C. The second line shows how we are initialising the
function to use XMM Registers, using the `sse2` instruction set. This is because paddb is
an `sse2` instruction. We'll cover `sse2` in more detail in the next lesson.

---

```assembly
cglobal add_values, 2, 2, 2, src, src2
```

This is an important line as it defines a C function called `add_values`.

* Let's go through each item one at a time:
	1. `cglobal`: content from `x86inc.asm`.
	1. `add_values`: function name.
	1. `2`: number of function arguments.
	1. `2`: number of GPRs that will be used for the arguments. In some cases we might
	want to use more GPRs so we have to tell x86util we need more.
	1. `2`: tells x86util how many XMM Registers we are going to use.
	1. `src, src2`: labels for the function arguments.

It's worth noting that older code may not have labels for the function arguments but
instead address GPRs directly using `r0`, `r1`, etc.

---

```assembly
    movu  m0, [srcq]
    movu  m1, [src2q]
```

`movu` is shorthand for `movdqu` (**mov**e **d**ouble **q**uad **u**naligned). Alignment
will be covered in another lesson but for now `movu` can be treated as a 128-bit move from
`[srcq]`. In the case of `mov`, the brackets mean that the address in `[srcq]` is being
dereferenced, the equivalent of `*src` in C. This is what's known as a load. Note that the
`q` suffix refers to the size of the pointer (i.e. in C it represents `sizeof(*src) == 8`
on 64-bit systems, and x86asm is smart enough to use 32-bit on 32-bit systems) but the
underlying load is 128-bit.

Note that we don't refer to vector registers by their full name, in this case `xmm0`, but as
`m0`, an abstracted form. In future lessons you'll see how this means you can write code once
and have it work on multiple SIMD Register sizes.

---

```assembly
paddb m0, m1
```

`paddb` (read this in your head as *p-add-b*) is adding each byte in each register as shown
below. The `p` prefix stands for "packed" and is used to identify vector instructions vs
scalar instructions. The `b` suffix shows that this is bytewise addition (addition of
bytes).

|a|b|c|d|e|f|g|h|i|j|k|l|m|n|o|p|
|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|

\+

|q|r|s|t|u|v|w|x|y|z|aa|ab|ac|ad|ae|af|
|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|

=

|a+q|b+r|c+s|d+t|e+u|f+v|g+w|h+x|i+y|j+z|k+aa|l+ab|m+ac|n+ad|o+ae|p+af|
|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|

---

```assembly
movu  [srcq], m0
```

This is what's known as a store. The data is written back to the address in the `srcq`
pointer.

```assembly
RET
```

---

This is a macro to denote the function returns. Virtually all Assembly functions in FFmpeg
modify the data in the arguments as opposed to returning a value.

As you'll see in the assignment, we create function pointers to Assembly functions and use
them where available.

> [Next lesson](../lesson-02/README.md)
