[avx512-removed]: https://www.igorslab.de/en/intel-deactivated-avx-512-on-alder-lake-but-fully-questionable-interpretation-of-efficiency-news-editorial/
[steam-survey]: https://store.steampowered.com/hwsurvey/Steam-Hardware-Software-Survey-Welcome-to-Steam
[fate-testsuite]: https://fate.ffmpeg.org/query=subarch:x86_64%2F%2F
[sign-ext-wiki]: https://en.wikipedia.org/wiki/Sign_extension

**FFmpeg Assembly Lesson Three**

Let’s explain some more jargon and give you a short history lesson.

**Instruction Sets**

You may have seen in the previous lesson we talked about SSE2 which is a set of SIMD
instructions. When a new CPU generation is released it may come with new instructions and
sometimes larger register sizes. The history of the x86 instruction set is very complex so
this is a simplified history (there are many more subcategories):

* MMX - Launched in 1997, first SIMD in Intel Processors, 64-bit registers, historic
* SSE (Streaming SIMD Extensions) - Launched in 1999, 128-bit registers
* SSE2 - Launched in 2000, many new instructions
* SSE3 - Launched in 2004, first horizontal instructions
* SSSE3 (Supplemental SSE3) - Launched in 2006, new instructions but most importantly
pshufb shuffle instruction, arguably the most important instruction in video processing
* SSE4 - Launched in 2008, many new instructions including packed minimum and maximum.
* AVX - Launched in 2011, 256-bit registers (float only) and new three-operand syntax
* AVX2 - Launched in 2013, 256-bit registers for integer instructions
* AVX512 - Launched in 2017, 512-bit registers, new operation mask feature. These had
limited use at the time in FFmpeg because of CPU frequency downscaling when new
instructions were used. Full 512-bit shuffle (permute) with vpermb.
* AVX512ICL - Launched 2019, no more clock frequency downscaling.
* AVX10 - Upcoming

It’s worth noting that instruction sets can be removed as well as added to CPUs. For
example [AVX512 was removed][avx512-removed] controversially, in 12th Generation Intel
CPUs. It’s for this reason that FFmpeg does runtime CPU detection. FFmpeg detects the
capabilities of the CPU it’s running on.

As you saw in the assignment, function pointers are C by default and are replaced with a
particular instruction set variant. This means detection is done once and then never needs
to be done again. This is in contrast to many proprietary applications which hardcode a
particular instruction set making a perfectly functional computer obsolete. This also
allows optimised functions to be turned on/off at runtime. This is one of the big benefits
of open source.

Programs like FFmpeg are used on billions of devices around the world, some of which may be
very old. FFmpeg technically supports machines supporting SSE only, which are 25 years old!
Thankfully x86inc.asm is capable of telling you if you use an instruction that’s not
available in a particular instruction set.

To give you an idea of real-world capabilities, here is the instruction set availability
from the [Steam Survey][steam-survey] as of November 2024 (this is obviously biased towards
gamers):

| Instruction Set | Availability |
| :-- | :-- |
| SSE2 | 100% |
| SSE3 | 100% |
| SSSE3 | 99.86% |
| SSE4.1 | 99.80% |
| AVX | 97.39% |
| AVX2 | 94.44% |
| AVX512 (Steam does not separate between AVX512 and AVX512ICL) | 14.09% |

For an application like FFmpeg with billions of users, even 0.1% is a very large number of
users and bug reports if something breaks. FFmpeg has extensive testing infrastructure for
testing the variations of CPU/OS/Compiler in our [FATE testsuite][fate-testsuite]. Every single commit is run on hundreds of machines to make
sure nothing breaks.

Intel provides a detailed instruction set manual here:
<https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html>

It can be cumbersome to search through a PDF so there is an unofficial web based
alternative here: <https://www.felixcloutier.com/x86/>

There is also a visual representation of SIMD instructions available here:
<https://www.officedaytime.com/simd512e/>

Part of the challenge of x86 assembly is finding the right instruction for your needs. In
some cases instructions can be used in a way they were not originally intended.

**Pointer offset trickery**

Let’s go back to our original function from Lesson 1, but add a width argument to the C
function.

We use ptrdiff_t for the width variable instead of int to make sure that the upper 32-bits
of the 64-bit argument are zero. If we directly passed an int width in the function
signature, and then attempted to use it as a quad for pointer arithmetic (i.e. using
`widthq`) the upper 32-bits of the register can be filled with arbitrary values. We could
fix this by sign extending width with `movsxd` (also see macro `movsxdifnidn` in
x86inc.asm), but this is an easier way.

The function below has the pointer offset trickery in it:

```assembly
;static void add_values(uint8_t *src, const uint8_t *src2, ptrdiff_t width)
INIT_XMM sse2
cglobal add_values, 3, 3, 2, src, src2, width
   add srcq, widthq
   add src2q, widthq
   neg widthq

.loop
    movu  m0, [srcq+widthq]
    movu  m1, [src2q+widthq]

    paddb m0, m1

    movu  [srcq+widthq], m0
    add   widthq, mmsize
    jl .loop

    RET
```

Let’s go through this step by step as it can be confusing:

```assembly
   add srcq, widthq
   add src2q, widthq
   neg widthq
```

The width is added to each pointer such that each pointer now points to the end of the
buffer to be processed. The width is then negated.

```assembly
    movu  m0, [srcq+widthq]
    movu  m1, [src2q+widthq]
```

The loads are then done with widthq being negative. So on the first iteration [srcq+widthq]
points to the original address of srcq, i.e points back to the beginning of the buffer.

```assembly
    add   widthq, mmsize
    jl .loop
```

mmsize is added to the negative widthq bringing it closer to zero. The loop condition is
now jl (jump if less than zero). This trick means widthq is used as a pointer offset
**and** as a loop counter at the same time, saving a cmp instruction. It also allows the
pointer offset to be used in multiple loads and stores, as well as using multiples of the
pointer offsets if needed (remember this for the assignment).

**Alignment**

In all our examples we have been using movu to avoid the topic of alignment. Many CPUs can
load and store data faster if the data is aligned, i.e if the memory address is divisible
by the SIMD register size. Where possible we try to use aligned loads and stores in FFmpeg
using mova.

In FFmpeg, av_malloc is able to provide aligned memory on the heap and the DECLARE_ALIGNED
C preprocessor directive can provide aligned memory on the stack. If mova is used with an
unaligned address, it will cause a segmentation fault and the application will crash. It’s
also important to be sure that the alignment value corresponds to the SIMD register size,
i.e 16 with xmm, 32 for ymm and 64 for zmm.

Here is how to align the beginning of the RODATA section to 64-bytes:

```assembly
SECTION_RODATA 64
```

Note that this just aligns the beginning of RODATA. Padding bytes might be needed to make
sure the next label remains on a 64-byte boundary.

**Range expansion**

Another topic we have avoided until now is overflowing. This happens, for example, when the
value of a byte goes beyond 255 after an operation like addition or multiplication. We may
want to perform an operation where we need an intermediate value larger than a byte (e.g
words), or potentially we want to leave the data in that larger intermediate size.

For unsigned bytes, this is where punpcklbw (packed unpack low bytes to words) and
punpckhbw (packed unpack high bytes to words) comes in.

Let’s look at how punpcklbw works. The syntax for the SSE2 version from the Intel Manual is
as follows:

| PUNPCKLBW xmm1, xmm2/m128 |
| :-- |

This means its source (right hand side) can be an xmm register or a memory address (m128 
means a memory address with the standard [base + scale*index + disp]) syntax and the
destination an xmm register.

The officedaytime.com website above has a good diagram showing what’s going on:

![What is this](https://raw.githubusercontent.com/duckafire/asm-lessons/refs/heads/main/lesson_03/image1.png)

You can see that bytes are interleaved from the lower half of each register respectively.
But what has this got to do with range extension? If the src register is all zeros this
interleaves the bytes in dst with zeros. This is what is known as *zero extension* as the
bytes are unsigned. punpckhbw can be used to do the same thing for the high bytes.

Here is a snippet showing how this is done:

```assembly
pxor      m2, m2 ; zero out m2

movu      m0, [srcq]
movu      m1, m0 ; make a copy of m0 in m1
punpcklbw m0, m2
punpckhbw m1, m2
```

`m0` and `m1` now contain the original bytes zero extended to words. In the next lesson
you’ll see how three-operand instructions in AVX make the second movu unnecessary.

**Sign extension**

Signed data is a bit more complicated. To range extend a signed integer, we need to use a
process known as [sign extension][sign-ext-wiki]. This pads
the MSBs with the sign bit. For example: -2 in int8_t is 0b11111110. To sign extend it t
int16_t the MSB of 1 is repeated to make 0b1111111111111110.

`pcmpgtb` (packed compare greater than byte) can be used for sign extension. By doing the
comparison (0 > byte), all the bits in the destination byte are set to 1 if the byte is
negative, otherwise the bits in the destination byte are set to 0. punpckX can be used as
above to perform the sign extension. If the byte is negative the corresponding byte is
0b11111111 and otherwise it’s 0x00000000. Interleaving the byte value with the output
of pcmpgtb performs a sign extension to word as a result.

```assembly
pxor      m2, m2 ; zero out m2

movu      m0, [srcq]
movu      m1, m0 ; make a copy of m0 in m1

pcmpgtb   m2, m0
punpcklbw m0, m2
punpckhbw m1, m2
```

As you can see there is an extra instruction compared to the unsigned case.

**Packing**

packuswb (pack unsigned word to byte) and packsswb lets you go from word to byte. It lets
you interleave two SIMD registers containing words into one SIMD register with a byte. Note
that if the values exceed the byte range, they will be saturated (i.e clamped at the
largest value).

**Shuffles**

Shuffles, also known as permutes, are arguably the most important instruction in video
processing and pshufb (packed shuffle bytes), available in SSSE3, is the most important
variant.

For each byte the corresponding source byte is used as an index of the destination
register, except when the MSB is set the destination byte is zeroed. It’s analogous to the
following C code (although in SIMD all 16 loop iterations happen in parallel):

```c
for(int i = 0; i < 16; i++) {
    if(src[i] & 0x80)
        dst[i] = 0;
    else
        dst[i] = dst[src[i]]
}
```
Here’s a simple assembly example:

```assembly
SECTION_DATA 64

shuffle_mask: db 4, 3, 1, 2, -1, 2, 3, 7, 5, 4, 3, 8, 12, 13, 15, -1

section .text

movu m0, [srcq]
movu m1, [shuffle_mask]
pshufb m0, m1 ; shuffle m0 based on m1
```

Note that -1 for easy reading is used as the shuffle index to zero out the output byte: -1
as a byte is the 0b11111111 bitfield (two’s complement), and thus the MSB (0x80) is set.
