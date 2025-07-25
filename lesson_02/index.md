# FFmpeg Assembly Language Lesson Two

Now that you've written your first assembly language function, we will now introduce
branches and loops.

We need to first introduce the idea of labels and jumps. In the artificial example below,
the `jmp` instruction moves the code instruction to after `.loop:`. `.loop:` is known as a
*label*, with the dot prefixing the label meaning it's a *local label*, effectively
allowing you to reuse the same label name across multiple functions. This example, of
course, shows an infinite loop, but we'll extend this later to something more realistic.

```assembly
mov  r0q, 3
.loop:
    dec  r0q
    jmp .loop
```

Before making a realistic loop we have to introduce the *FLAGS* register. We won't dwell on
the intricacies of *FLAGS* too much (again because GPR operations are largely scaffolding)
but there are several flags such as Zero-Flag, Sign-Flag and Overflow-Flag which are set
based on the output of most *non-mov* instructions on scalar data such as arithmetic
operations and shifts.

Here's an example where the loop counter counts down until zero and `jg` (**j**ump if
**g**reater than zero) is the loop condition. `dec r0q` sets the FLAGs based on the value
of `r0q` after the instruction and you can jump based on them.

```assembly
mov  r0q, 3
.loop:
    ; do something
    dec  r0q
    jg  .loop ; jump if greater than zero
```

This is equivalent to the following C code:

```c
int i = 3;
do
{
   // do something
   i--;
} while(i > 0)
```

This C code is a bit unnatural. Usually a loop in C is written like this:

```c
int i;
for(i = 0; i < 3; i++) {
    // do something
}
```

This is roughly equivalent to (there's no simple way of matching this `for` loop):

```assembly
xor r0q, r0q
.loop:
    ; do something
    inc r0q
    cmp r0q, 3
    jl  .loop ; jump if (r0q - 3) < 0, i.e (r0q < 3)
```

There are several things to point out in this snippet. First is `xor r0q, r0q` which is
a common way of setting a register to zero, that on some systems is faster than `mov r0q,
0`, because, put simply, there is no actual load taking place. It can also be used on
SIMD registers with `pxor m0, m0` to zero out an entire register. The next thing to
note is the use of `cmp`. `cmp` effectively subtracts the second register from the first
(without storing the value anywhere) and sets *FLAGS*, but as per the comment, it can be
read together with the jump, (`jl` = **j**ump if **l**ess than zero) to jump if `r0q < 3`.

Note how there is one extra instruction (`cmp`) in this snippet. Generally speaking, fewer
instructions means faster code, which is why the earlier snippet is preferred. As you'll
see in future lessons, there are more tricks used to avoid this extra instruction and have
*FLAGS* be set by arithmetic or another operation. Note how we are not writing assembly to
match C loops exactly, we write loops to make them as fast as possible in assembly.

Here are some common jump mnemonics you'll end up using (*FLAGS* are there for
completeness, but you don't need to know the specifics to write loops):

| Mnemonic | Description  | FLAGS |
| :-- | :-- | :-- |
| `JE`/`JZ` | Jump if Equal/Zero | `ZF == 1` |
| `JNE`/`JNZ` | Jump if Not Equal/Not Zero | `ZF == 0` |
| `JG`/`JNLE` | Jump if Greater/Not Less or Equal (signed) | `ZF == 0 a&& SF = OF` |
| `JGE`/`JNL` | Jump if Greater or Equal/Not Less (signed) | `SF == OF` |
| `JL`/`JNGE` | Jump if Less/Not Greater or Equal (signed) | `SF != OF` |
| `JLE`/`JNG` | Jump if Less or Equal/Not Greater (signed) | `ZF == 1 || SF != OF` |

## Constants

Let's look at some examples showing how to use constants:

```assembly
SECTION_RODATA

constants_1: db 1,2,3,4
constants_2: times 2 dw 4,3,2,1
```

* `SECTION_RODATA` specifies this is a read-only data section. (This is a macro because
different output file formats that operating systems use declare this differently)
* `constants_1:` The label `constants_1`, is defined as `db` (declare byte) - i.e
equivalent to `uint8_t constants_1[4] = {1, 2, 3, 4};`
* `constants_2:` This uses the `times 2` macro to repeat the declared words - i.e
equivalent to `uint16_t constants_2[8] = {4, 3, 2, 1, 4, 3, 2, 1};`

These labels, which the assembler converts to a memory address, can then be used in loads
(but not stores as they are read-only). Some instructions take a memory address as an
operand so they can be used without explicit loads into a register (there are pros and cons
to this).

## Offsets

Offsets are the distance (in bytes) between consecutive elements in memory. The offset is
determined by the **size of each element** in the data structure.

Now that we're able to write loops, it's time to fetch data. But there are some differences
compared to C. Let's look at the following loop in C:

```c
uint32_t data[3];
int i;
for(i = 0; i < 3; i++) {
    data[i];
}
```

The 4-byte offset between elements of data is precalculated by the C compiler. But when
handwriting assembly you need to calculate these offsets yourself.

Let's look at the syntax for memory address calculations. This applies to all types of
memory addresses:

```assembly
[base + scale*index + disp]
```

* base - This is a GPR (usually a pointer from a C function argument)
* scale - This can be `1`, `2`, `4`, `8`. `1` is the default
* index - This is a GPR (usually a loop counter)
* disp - This is an integer (up to 32-bit). Displacement is an offset into the data

x86asm provides the constant `mmsize`, which lets you know the size of the SIMD register you
are working with.

Here's a simple (and nonsensical) example to illustrate loading from custom offsets:

```assembly
;static void simple_loop(const uint8_t *src)
INIT_XMM sse2
cglobal simple_loop, 1, 2, 2, src
     movq r1q, 3
.loop:
     movu m0, [srcq]
     movu m1, [srcq+2*r1q+3+mmsize]

     ; do some things

     add srcq, mmsize
dec r1q
jg .loop

RET
```

Note how in `movu m1, [srcq+2*r1q+3+mmsize]` the assembler will precalculate the right
displacement constant to use. In the next lesson we'll show you a trick to avoid having to
do `add` and `dec` in the loop, replacing them with a single `add`.

## LEA

Now that you understand offsets you are able to use lea (Load Effective Address). This lets
you perform multiplication and addition with one instruction, which is going to be faster
than using multiple instructions. There are, of course, limitations on what you can
multiply by and add to but this does not stop lea being a powerful instruction.

```assembly
lea r0q, [base + scale*index + disp]
```

Contrary to the name, LEA can be used for normal arithmetic as well as address
calculations. You can do something as complicated as:

```assembly
lea r0q, [r1q + 8*r2q + 5]
```

Note that this does not affect the contents of `r1q` and `r2q`. It also doesn't affect
*FLAGS* (so you can't jump based on the output). Using LEA avoids all these instructions
and temporary registers (this code is not equivalent because add changes *FLAGS*):

```assembly
movq r0q, r1q
movq r3q, r2q
sal  r3q, 3 ; shift arithmetic left 3 = * 8
add  r3q, 5
add  r0q, r3q
```

You'll see lea used a lot to set up addresses before loops or perform calculations like the
above. Note of course, that you can't do all types of multiply and addition, but
multiplications by `1`, `2`, `4`, `8` and addition of a fixed offset is common.

In the assignment you'll have to load a constant and add the values to a SIMD vector in a
loop.

> [Next Lesson](../lesson_03/index.md)
