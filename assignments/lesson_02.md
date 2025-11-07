# FFmpeg Assembly Language Lesson Two Assignments

**Overview**  
Lesson Two covered branches, labels, and the FLAGS register. These problems will cement your grasp of conditional jumps and nested loops.

---

## Assignment 1: Bubble Sort

Implement bubble sort in assembly:

```c
void bubble_sort(int *arr, int len);
```

Requirements:

Declare via

```assembly
cglobal bubble_sort, 2, 0, 0, arr, len
```

Accept:

- %rdi → arr
- %rsi → len

Sort arr[] in-place in ascending order.

Use nested loops:

- Outer pass loop (.pass)
- Inner compare loop (cmp [rdi+r8*4], [rdi+r8*4+4])
- Swap out-of-order elements with mov/xchg
- Track if any swap occurred; break if none

Label every loop and document your FLAGS-based jumps.

---

## Assignment 2: Reverse a String

Write a routine that reverses a NUL-terminated string in place:

```c
void reverse_string(char *s);
```

Your implementation should:

Declare via

```assembly
cglobal reverse_string, 1, 0, 0, s
```

Accept:

- %rdi → pointer to the first character of s

Perform two-pointer reversal:

- Scan forward to find the NUL (movzx r8b, [rdi]; cmp r8b, 0; jne .scan)
- Set r8q just before the NUL; r9q at start
- Swap bytes with xchg [r9q], [r8q]
- Advance r9q, retreat r8q; loop until r9q >= r8q

Use jl or jg on a cmp r9q, r8q to control your loop
