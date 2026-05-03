**FFmpeg 汇编语言第二课**

既然你已经写了你的第一个汇编语言函数，现在将介绍分支和循环。

首先介绍标签（label）和跳转的概念。在下面的人工示例中，jmp 指令将代码执行移动到 ".loop:" 之后。".loop:" 被称为*标签*（label），标签前的点表示它是一个*局部标签*（local label），允许你在多个函数中重用相同的标签名。当然，这个例子展示了一个无限循环，但稍后会将其扩展为更实际的内容。

```assembly
mov  r0q, 3
.loop:
    dec  r0q
    jmp .loop
```

在建立一个实际的循环之前，必须介绍 *FLAGS*（标志）寄存器。这里不会过多纠结于 *FLAGS* 的细节（因为 GPR 操作很大程度上只是脚手架），但有几个标志，如零标志（Zero-Flag）、符号标志（Sign-Flag）和溢出标志（Overflow-Flag），它们根据大多数非 mov 指令对标量数据（如算术运算和移位）的输出来设置。

这是一个例子，循环计数器递减直到零，jg（如果大于零则跳转）是循环条件。dec r0q 根据指令后 r0q 的值设置 FLAGS，你可以根据它们进行跳转。

```assembly
mov  r0q, 3
.loop:
    ; 做一些事情
    dec  r0q
    jg  .loop ; 如大于零则跳转
```

这相当于以下的 C 代码：

```c
int i = 3;
do
{
   // 做一些事情
   i--;
} while(i > 0)
```

这段 C 代码有点不自然。通常 C 中的循环是这样写的：

```c
int i;
for(i = 0; i < 3; i++) {
    // 做一些事情
}
```

这大致相当于（没有简单的方法匹配这个 ```for``` 循环）：

```assembly
xor r0q, r0q
.loop:
    ; 做一些事情
    inc r0q
    cmp r0q, 3
    jl  .loop ; 如果 (r0q - 3) < 0 跳转, 即 (r0q < 3)
```

在这个片段中有几点需要指出。首先是 ```xor r0q, r0q```，这是将寄存器清零的常用方法，在某些系统上比 ```mov r0q, 0``` 更快，因为没有实际的加载发生。它也可以用于 SIMD 寄存器，如 ```pxor m0, m0``` 来清零整个寄存器。接下来要注意的是 cmp 的使用。cmp 实际上是从第一个寄存器中减去第二个寄存器（但不存储结果）并设置 *FLAGS*，正如注释所示，它可以与跳转指令组合来读取——(jl = 小于零则跳转) 即如果 ```r0q < 3``` 则跳转。

注意这个片段中多了一条指令（cmp）。一般来说，指令越少代码越快，这就是为什么前面的写法更好。正如你在后续课程中会看到的，还有更多技巧可以避免这条额外指令，让 *FLAGS* 由算术或其他操作来设置。注意编写汇编并不是为了精确匹配 C 循环，而是为了让循环在汇编中尽可能快。

这里有一些你最终会用到的常见跳转助记符（*FLAGS* 在这里是为了完整性，但你不需要知道细节来编写循环）：

| 助记符 | 描述 | FLAGS |
| :---- | :---- | :---- |
| JE/JZ | 相等/为零时跳转 (Jump if Equal/Zero) | ZF = 1 |
| JNE/JNZ | 不相等/不为零时跳转 (Jump if Not Equal/Not Zero) | ZF = 0 |
| JG/JNLE | 大于/不小于等于时跳转 (有符号) (Jump if Greater/Not Less or Equal) | ZF = 0 且 SF = OF |
| JGE/JNL | 大于等于/不小于时跳转 (有符号) (Jump if Greater or Equal/Not Less) | SF = OF |
| JL/JNGE | 小于/不大于等于时跳转 (有符号) (Jump if Less/Not Greater or Equal) | SF ≠ OF |
| JLE/JNG | 小于等于/不大于时跳转 (有符号) (Jump if Less or Equal/Not Greater) | ZF = 1 或 SF ≠ OF |

**常量**

来看一些展示如何使用常量的例子：

```assembly
SECTION_RODATA

constants_1: db 1,2,3,4
constants_2: times 2 dw 4,3,2,1
```

* SECTION_RODATA 指定这是一个只读数据段。（这是一个宏，因为操作系统使用的不同输出文件格式对此有不同的声明）
* constants_1: 标签 constants_1，被定义为 ```db```（declare byte，即定义字节）- 相当于 ```uint8_t constants_1[4] = {1, 2, 3, 4};```
* constants_2: 这使用了 ```times 2``` 宏来重复声明的字 - 相当于 ```uint16_t constants_2[8] = {4, 3, 2, 1, 4, 3, 2, 1};```

这些标签（汇编器会将其转换为内存地址）可用于加载（但不能用于存储，因为是只读的）。有些指令可以直接接受内存地址作为操作数，因此无需先显式加载到寄存器（这样做各有利弊）。

**偏移量（Offsets）**

偏移量是内存中连续元素之间的距离（以字节为单位）。偏移量由数据结构中**每个元素的大小**决定。

既然能够编写循环了，是时候获取数据了。但这与 C 相比有一些差异。来看下面的 C 循环：

```c
uint32_t data[3];
int i;
for(i = 0; i < 3; i++) {
    data[i];
}
```

数据元素之间的 4 字节偏移量是由 C 编译器预先计算的。但是当手写汇编时，你需要自己计算这些偏移量。

来看内存地址计算的语法。这适用于所有类型的内存地址：

```assembly
[base + scale*index + disp]
```

* base - 这是一个 GPR（通常是来自 C 函数参数的指针）
* scale - 这可以是 1, 2, 4, 8。1 是默认值
* index - 这是一个 GPR（通常是循环计数器）
* disp - 这是一个整数（最高 32 位）。位移（Displacement）是进入数据的偏移量

x86asm 提供了常量 mmsize，它让你知道你正在使用的 SIMD 寄存器的大小。

这是一个简单（且无实际意义）的例子，用来说明从自定义偏移量加载：

```assembly
;static void simple_loop(const uint8_t *src)
INIT_XMM sse2
cglobal simple_loop, 1, 2, 2, src
     movq r1q, 3
.loop:
     movu m0, [srcq]
     movu m1, [srcq+2*r1q+3+mmsize]

     ; 做一些事情

     add srcq, mmsize
dec r1q
jg .loop

RET
```

注意在 ```movu m1, [srcq+2*r1q+3+mmsize]``` 中，汇编器将预先计算要使用的正确位移常量。在下一课中，将展示一个技巧，以避免在循环中进行 add 和 dec，而是用一个单独的 add 替换它们。

**LEA**

既然你理解了偏移量，就可以使用 lea（Load Effective Address，即加载有效地址）了。它让你能用一条指令完成乘法和加法，比使用多条指令更快。当然，能乘以什么、加上什么是有限制的，但这不妨碍 lea 成为一条强大的指令。

```assembly
lea r0q, [base + scale*index + disp]
```

与名称相反，LEA 不仅可以用于地址计算，还可以用于普通算术。你可以做这样的事情：

```assembly
lea r0q, [r1q + 8*r2q + 5]
```

注意这不会影响 r1q 和 r2q 的内容，也不影响 *FLAGS*（所以你不能根据输出跳转）。使用 LEA 可以避免以下所有指令和临时寄存器（注意下面的代码并不等效，因为 add 会改变 *FLAGS*）：

```assembly
movq r0q, r1q
movq r3q, r2q
sal  r3q, 3 ; 算术左移 3 = * 8
add  r3q, 5
add  r0q, r3q
```

你会看到 lea 经常用于在循环之前设置地址或执行像上面那样的计算。当然要注意，你不能做所有类型的乘法和加法，但乘以 1, 2, 4, 8 和加上固定偏移量是常见的。

在作业中，你将必须加载一个常量并在循环中将这些值加到 SIMD 向量中。

[下一课](../lesson_03/index.zh.md)
