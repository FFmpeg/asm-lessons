# FFmpeg Assembly Language Lesson Three Assignments

**Overview**  
Lesson Three introduced multiple instruction-set generations, pointer-offset trickery, alignment, and SIMD range-extension. These tasks let you apply those patterns in real FFmpeg-style functions.

---

## Assignment 1: Pointer-Offset Trickery

Implement this variant of add_values:

```c
static void add_values_trick(uint8_t *src, const uint8_t *src2, ptrdiff_t width);
```

Your code should:

Declare via

```assembly
cglobal add_values_trick, 3, 3, 2, src, src2, width
```

Accept:

- %rdi → src
- %rsi → src2
- %rdx → width (in bytes)

Use the "width-as-counter" trick:

- add srcq, widthq & add src2q, widthq
- neg widthq
- Loop .L::
  - movu m0, [srcq+widthq]
  - movu m1, [src2q+widthq]
  - paddb m0, m1
  - movu [srcq+widthq], m0
  - add widthq, mmsize
  - jl .L

Comment how widthq serves both as offset and loop counter.

---

## Assignment 2: Range Extension (Zero-Extend)

Write a function to zero-extend unsigned bytes to words:

```c
void extend_u8_to_u16(const uint8_t *src, uint16_t *dst, int count);
```

Requirements:

Declare via

```assembly
cglobal extend_u8_to_u16, 3, 0, 0, src, dst, count
```

Accept:

- %rdi → src
- %rsi → dst
- %rdx → count (elements)

In each iteration:

- Load 16 bytes: movu m0, [rdi]
- Zero a vector register: pxor m1, m1
- punpcklbw m0, m1 → low 8 bytes → words in m0
- punpckhbw m2, m1 → high 8 bytes → words in m2
- Store both halves to dst
- Advance pointers by mmsize and decrement count

Use jl to loop while negative count remains.

---

## Assignment 3: Byte Shuffle

Implement a byte-shuffle kernel using SSSE3's pshufb:

```c
void shuffle_bytes(const uint8_t *src, const uint8_t *mask, uint8_t *out, int count);
```

Your code should:

Declare via

```assembly
cglobal shuffle_bytes, 4, 0, 0, src, mask, out, count
```

Accept:

- %rdi → src data
- %rsi → mask table (16-byte shuffle mask)
- %rdx → out
- %rcx → count (blocks of 16 bytes)

In each block:

- Load data: movu m0, [src]
- Load mask: movu m1, [mask]
- pshufb m0, m1
- Store result: movu [out], m0
- Advance src, out by mmsize; decrement count

Loop with jl and document your byte-mask logic.
