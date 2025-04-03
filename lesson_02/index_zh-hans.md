**FFmpeg 汇编语言第二课**

现在你已经写过了第一个汇编函数，我们开始介绍分支（branches）和循环（loops）。

我们先介绍标签（labels）与跳转（jumps）的概念。在下方构造的例子里，jmp（跳转）指令将代码执行流程移动到 ".loop:" 指令下方。".loop:" 就是一个*标签*，标签前缀的点（.）表示它是一个*本地标签*，这样就可以在多个函数中重复使用同一个标签名。当然，这个例子展示的是一个无限循环，但我们稍后会将其扩展到更现实的场景。

```assembly
mov  r0q, 3
.loop:
    dec  r0q
    jmp .loop
```

在开始实际循环之前，我们必须先介绍 *FLAGS* (标志)寄存器。我们不会过多讨论 *FLAGS* (标志)的复杂性（同样因为 GPR 操作主要是脚手架），但有几个标志（flags），如零标志（Zero-Flag）、符号标志（Sign-Flag）和溢出标志（Overflow-Flag），这些标志是根据大多数非移动（non-mov）指令对标量数据（如算术运算和移位）的输出设置的。

下面的示例中，循环计数器倒数到 0，jg（**j**ump if **g**reater than zero，大于 0 则跳转）是循环条件。dec r0q 指令会根据该指令执行后 r0q 的值设置 FLAGS，然后你可以根据这些 FLAGS 进行跳转。

```assembly
mov  r0q, 3
.loop:
    ; do something
    dec  r0q
    jg  .loop ; 大于 0 则跳转
```

它等同于如下 C 代码：

```c
int i = 3;
do
{
   // do something
   i--;
} while(i > 0)
```

这段 C 代码有点不太自然。一般来说 C 语言里循环功能是这么写的：

```c
int i;
for(i = 0; i < 3; i++) {
    // do something
}
```

这段大致等同于（没有简单的方法来完全匹配这个 ```for``` 循环）：

```assembly
xor r0q, r0q
.loop:
    ; do something
    inc r0q
    cmp r0q, 3
    jl  .loop ; 如果 (r0q - 3) < 0 则跳转, 即 (r0q < 3)
```

这段代码有几点值得注意。首先是 ```xor r0q, r0q```，这是一种常见的将寄存器清零的方法，在某些系统上比 ```mov r0q, 0``` 更快，简而言之，这是因为不需要进行实际的加载操作。这也可以用于 SIMD 寄存器，通过 ```pxor m0, m0``` 来将整个寄存器清零。另一个需要注意的是 cmp (compare，对比)的使用。cmp 实际上是从第一个寄存器中减去第二个寄存器的值（但不存储结果）并设置 **FLAGS**，但根据注释所示，它可以与跳转指令一起理解，（jl = **j**ump if **l**ess than zero，如果小于零则跳转）以在 ```r0q < 3``` 时进行跳转。

请注意，在这个代码片段中有一条额外的指令（cmp）。一般来说，指令越少意味着代码运行越快，这就是为什么之前的代码片段更受青睐。正如你将在后续课程中看到的，还有更多技巧可以用来避免这种额外指令，并利用算术或其他操作来设置 **FLAGS** 寄存器。需要注意的是，我们编写汇编代码时并不是为了完全匹配 C 语言的循环结构，而是为了在汇编中让循环尽可能地高效。

以下是一些你将会用到的常见跳转助记符（jump mnemonics）（*FLAGS* 在此列出只是为了完整性，你不需要了解具体细节就能编写循环）：

| 助记符 | 描述  | FLAGS |
| :---- | :---- | :---- |
| JE/JZ | 相等/为零时跳转 | ZF = 1 |
| JNE/JNZ | 不相等/不为零时跳转 | ZF = 0 |
| JG/JNLE | 大于/不小于等于时跳转（有符号） | ZF = 0 and SF = OF |
| JGE/JNL | 大于等于/不小于时跳转（有符号） | SF = OF |
| JL/JNGE | 小于/不大于等于时跳转（有符号） | SF ≠ OF |
| JLE/JNG | 小于等于/不大于时跳转（有符号） | ZF = 1 or SF ≠ OF |

**常量（Constants）**

让我们看几个展示如何使用常量的例子：

```assembly
SECTION_RODATA

constants_1: db 1,2,3,4
constants_2: times 2 dw 4,3,2,1
```

* SECTION_RODATA 指定这是一个只读数据段（read-only data section）。（这是一个宏，因为不同操作系统使用不同的声明方式输出文件格式）
* constants_1：标签 constants_1 通过 ```db```（声明字节，declare byte）定义 - 相当于 uint8_t constants_1[4] = {1, 2, 3, 4};
* constants_2：这里使用 ```times 2``` 宏来重复声明的字（words）- 相当于 uint16_t constants_2[8] = {4, 3, 2, 1, 4, 3, 2, 1};

这些标签会被汇编器转换为内存地址，然后可以用于加载操作（但由于是只读的，不能用于存储操作）。某些指令可以直接将内存地址作为操作数（operand），而无需显式地先加载到寄存器中（这种做法既有优点也有缺点）。

**偏移量（Offsets）**

偏移量是内存中连续元素之间的距离（以字节为单位）。偏移量由数据结构中**每个元素的大小**决定。

现在我们已经能够编写循环，是时候获取数据了。但这与 C 语言相比有一些区别。让我们看看以下例子中的 C 语言循环：

```c
uint32_t data[3];
int i;
for(i = 0; i < 3; i++) {
    data[i];
}
```

C 编译器会预先计算出数组元素之间的 4 字节偏移量。但在手写汇编代码时，你需要自己计算这些偏移量。

让我们看看内存地址计算的语法。这适用于所有类型的内存地址：

```assembly
[base + scale*index + disp]
```

* base - 这是一个通用寄存器（GPR）（通常是 C 函数的参数的指针）
* scale - 这可以是 1、2、4 或 8。默认值为 1
* index - 这是一个通用寄存器（GPR）（通常是循环计数器）
* disp - 这是一个整数（最大 32 位）。位移（Displacement）是数据的偏移量

x86asm 提供了常量 mmsize，它让你知道正在使用的 SIMD 寄存器的大小。

以下是一个简单的（没有实际意义的）例子，用来说明如何从自定义偏移量加载数据：

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

请注意在 ```movu m1, [srcq+2*r1q+3+mmsize]``` 中，汇编器会预先计算出正确的位移常量。在下一课中，我们将介绍一个技巧，可以避免在循环中同时使用 add 和 dec 指令，而是用单个 add 指令来替代它们。

**LEA**

现在你已经理解了偏移量，你可以使用 lea（加载有效地址，Load Effective Address）指令。它能让你用一条指令完成乘法和加法运算，这比使用多条指令要快得多。当然，lea 对可以乘以什么数和加上什么值有一些限制，但这并不妨碍 lea 成为一个强大的指令。

```assembly
lea r0q, [base + scale*index + disp]
```

与其名称相反，LEA 不仅可以用于地址计算，还可以用于普通的算术运算。你可以做一些复杂的操作，比如：

```assembly
lea r0q, [r1q + 8*r2q + 5]
```

请注意，这不会影响 r1q 和 r2q 的内容。它也不会影响 *FLAGS* 寄存器（所以你不能根据其输出结果进行跳转）。使用 LEA 可以避免所有这些指令和临时寄存器的使用（这段代码不完全等价，因为 add 会改变 *FLAGS* 寄存器）：

```assembly
movq r0q, r1q
movq r3q, r2q
sal  r3q, 3 ; shift arithmetic left（左移运算） 3 = * 8
add  r3q, 5
add  r0q, r3q
```

你会看到 lea 经常被用来在循环前设置地址或执行上述计算。当然需要注意的是，你不能用 lea 执行所有类型的乘法和加法运算，但乘以 1、2、4、8 和添加固定偏移量是很常见的用法。

在作业中，你需要在循环中加载常量并将这些值添加到 SIMD 向量中。

[下一课](../lesson_03/index_zh-hans.md)
