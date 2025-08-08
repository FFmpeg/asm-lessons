# FFmpeg Assembly Language Lesson One Assignments

**Overview**  
In Lesson One you mastered registers and basic loops. These exercises give you hands-on practice with loops, data movement, and the FFmpeg calling conventions.

---

## Assignment 1: Sum an Array

Write an x86-64 assembly function that computes the sum of an array of 32-bit integers:

```c
int sum_array(int *arr, int length);
```

Your implementation should:

Declare via

```assembly
cglobal sum_array, 2, 0, 0, arr, length
```

Accept:

- %rdi → pointer to arr[0]
- %rsi → length

Return:

- %rax → sum of all elements

Use a simple loop:

- Zero %rax
- Load each element (movl [rdi], r0d)
- Add it into %rax
- Advance pointer or decrement counter
- dec rsi; jg .loop

Comment every register use and loop label.

---

## Assignment 2: Implement my_memcpy

Create a FFmpeg-style routine to copy bytes:

```c
void *my_memcpy(void *dest, const void *src, size_t n);
```

Your code should:

Declare via

```assembly
cglobal my_memcpy, 3, 0, 0, dest, src, n
```

Accept:

- %rdi → dest
- %rsi → src
- %rdx → n (byte count)

Return:

- %rax → dest

Copy data in two phases:

- 8-byte chunks with movq in a loop
- Byte-wise remainder with movb

Use labels and jumps (jz, jnz) to manage each loop.

Annotate your choice of loop structure in comments.
