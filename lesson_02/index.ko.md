# FFmpeg 어셈블리어 강의 2강

첫 번째 어셈블리어 함수를 작성해보았습니다. 이번에는 분기(branch)와 반복문(loop)을 소개하겠습니다.

먼저 레이블과 점프(jump)의 개념을 알아야 합니다. 아래의 예제에서 `jmp` 명령어는 코드 실행을 `.loop:` 뒤로 이동시킵니다. `.loop:`는 **레이블** 이라 부르며, 레이블 앞에 붙은 점(.)은 이것이 로컬 레이블(local label) 임을 의미합니다. 로컬 레이블을 사용하면 여러 함수에서 동일한 레이블을 재사용 할 수 있습니다.

아래는 무한 루프의 예제입니다. 이후 이를 좀 더 현실적인 예시로 확장할 것입니다.

```assembly
mov  r0q, 3
.loop:
    dec  r0q
    jmp .loop
```

보다 현실적인 반복문을 만들기 전에 FLAGS 레지스터를 소개해야 합니다. FLAGS의 세부 동작에 대해서는 깊게 다루지 않겠습니다(앞서 설명했듯이 GPR 연산은 주로 보조 역할이기 때문입니다). 다만, 산술 연산이나 시프트처럼 스칼라 데이터에 대해 mov 이외의 대부분의 명령을 실행하면 **Zero-Flag**, **Sign-Flag**, **Overflow-Flag**와 같은 여러 플래그가 설정됩니다.

다음은 루프 카운터가 0까지 감소하도록 하고, `jg`(0보다 크면 점프)가 반복 조건이 되는 예시입니다. `dec r0q` 명령은 실행 후 `r0q` 값에 따라 플래그들을 설정하며, 이 플래그를 바탕으로 점프를 할 수 있습니다.

```assembly
mov  r0q, 3
.loop:
    ; ...
    dec  r0q
    jg  .loop ; 0보다 크면 점프
```

이는 다음 C 코드와 동일합니다.

```c
int i = 3;
do
{
   // ...
   i--;
} while(i > 0)
```

이 C 코드는 약간 부자연스럽습니다. 일반적인 반복문은 다음과 같이 작성합니다.

```c
int i;
for(i = 0; i < 3; i++) {
    // ...
}
```

This is roughly equivalent to (there's no simple way of matching this ```for``` loop):

이 `for` 루프와 동일하게 대응시키는 방법은 없지만, 대략 다음 어셈블리 코드와 비슷합니다.

```assembly
xor r0q, r0q
.loop:
    ; ...
    inc r0q
    cmp r0q, 3
    jl  .loop ; jump if (r0q - 3) < 0, i.e (r0q < 3)
```

The next thing to note is the use of cmp. cmp effectively subtracts the second register from the first (without storing the value anywhere) and sets *FLAGS*, but as per the comment, it can be read together with the jump, (jl = jump if less than zero) to jump if ```r0q < 3```.

이 코드 조각에서 주목해야 할 점이 몇 가지 있습니다.

첫째, `xor r0q, r0q`는 레지스터를 0으로 만드는 흔한 방법입니다. 일부 시스템에서는 `mov r0q, 0`보다 더 빠를 수 있는데, 간단히 설명한다면 실제 메모리 로드 동작이 일어나지 않기 때문입니다. 이 방식은 SIMD 레지스터에서도 사용할 수 있으며, 예시로 `pxor m0, m0`[^1]는 전체 레지스터를 0으로 초기화 합니다.

둘째, `cmp`의 사용입니다. `cmp`는 첫 번째 피연산자에서 두 번째 피연산자를 뺀 값을 어디에도 저장하지 않고, 그 결과에 따라 FLAGS를 설정합니다. 주석에 적힌 것처럼 점프 명령과 함께 읽어야 하며, `jl`(jump if less than zero)는 `r0q < 3`일 때 점프하게 됩니다.

또한 이 코드에는 추가 명령어(`cmp`)가 하나 더 있다는 점을 유의해야 합니다. 일반적으로 명령어 수가 적을수록 코드가 더 빠르기 때문에 이전 예제의 방식이 더 선호됩니다. 앞으로의 강의에서 보겠지만, 이러한 추가 명령을 피하고 산술 연산니아 다른 연산에서 바로 FLAGS를 설정하는 다양한 기법들이 있습니다.

마지막으로, C의 반복문을 어셈블리에서 정확히 동일하게 재현하려는 것이 아니라, 어셈블리에서 가능한 빠르게 동작하는 반복문을 작성해야 한다는 점을 유의해야 합니다.

Here are some common jump mnemonics you’ll end up using (*FLAGS* are there for completeness, but you don’t need to know the specifics to write loops):

| Mnemonic | Description  | FLAGS |
| :---- | :---- | :---- |
| JE/JZ | Jump if Equal/Zero | ZF = 1 |
| JNE/JNZ | Jump if Not Equal/Not Zero | ZF = 0 |
| JG/JNLE | Jump if Greater/Not Less or Equal (signed) | ZF = 0 and SF = OF |
| JGE/JNL | Jump if Greater or Equal/Not Less (signed) | SF = OF |
| JL/JNGE | Jump if Less/Not Greater or Equal (signed) | SF ≠ OF |
| JLE/JNG | Jump if Less or Equal/Not Greater (signed) | ZF = 1 or SF ≠ OF |

**Constants**

Let’s look at some examples showing how to use constants:

```assembly
SECTION_RODATA

constants_1: db 1,2,3,4
constants_2: times 2 dw 4,3,2,1
```

* SECTION_RODATA specifies this is a read-only data section. (This is a macro because different output file formats that operating systems use declare this differently)
* constants_1: The label constants_1, is defined as ```db``` (declare byte) - i.e equivalent to uint8_t constants_1[4] = {1, 2, 3, 4};
* constants_2: This uses the ```times 2``` macro to repeat the declared words - i.e equivalent to uint16_t constants_2[8] = {4, 3, 2, 1, 4, 3, 2, 1};

These labels, which the assembler converts to a memory address, can then be used in loads (but not stores as they are read-only). Some instructions take a memory address as an operand so they can be used without explicit loads into a register (there are pros and cons to this).

**Offsets**

Offsets are the distance (in bytes) between consecutive elements in memory. The offset is determined by the **size of each element** in the data structure.

Now that we're able to write loops, it’s time to fetch data. But there are some differences compared to C. Let’s look at the following loop in C:

```c
uint32_t data[3];
int i;
for(i = 0; i < 3; i++) {
    data[i];
}
```

The 4-byte offset between elements of data is precalculated by the C compiler. But when handwriting assembly you need to calculate these offsets yourself.

Let’s look at the syntax for memory address calculations. This applies to all types of memory addresses:

```assembly
[base + scale*index + disp]
```

* base - This is a GPR (usually a pointer from a C function argument)
* scale - This can be 1, 2, 4, 8. 1 is the default
* index - This is a GPR (usually a loop counter)
* disp - This is an integer (up to 32-bit). Displacement is an offset into the data

x86asm provides the constant mmsize, which lets you know the size of the SIMD register you are working with.

Here’s a simple (and nonsensical) example to illustrate loading from custom offsets:

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

Note how in ```movu m1, [srcq+2*r1q+3+mmsize]``` the assembler will precalculate the right displacement constant to use. In the next lesson we’ll show you a trick to avoid having to do add and dec in the loop, replacing them with a single add.

**LEA**

Now that you understand offsets you are able to use lea (Load Effective Address). This lets you perform multiplication and addition with one instruction, which is going to be faster than using multiple instructions. There are, of course, limitations on what you can multiply by and add to but this does not stop lea being a powerful instruction.

```assembly
lea r0q, [base + scale*index + disp]
```

Contrary to the name, LEA can be used for normal arithmetic as well as address calculations. You can do something as complicated as:

```assembly
lea r0q, [r1q + 8*r2q + 5]
```

Note that this does not affect the contents of r1q and r2q. It also doesn’t affect *FLAGS* (so you can’t jump based on the output). Using LEA avoids all these instructions and temporary registers (this code is not equivalent because add changes *FLAGS*):

```assembly
movq r0q, r1q
movq r3q, r2q
sal  r3q, 3 ; shift arithmetic left 3 = * 8
add  r3q, 5
add  r0q, r3q
```

You’ll see lea used a lot to set up addresses before loops or perform calculations like the above. Note of course, that you can’t do all types of multiply and addition, but multiplications by 1, 2, 4, 8 and addition of a fixed offset is common.

In the assignment you’ll have to load a constant and add the values to a SIMD vector in a loop.

[Next Lesson](../lesson_03/index.md)

- [^1]: `pxor`은 *Packed XOR*의 줄임말로 두 개의 SIMD 레지스터나 메모리 위치에 있는 데이터에 대해 비트별 XOR 연산을 병렬로 수행합니다.