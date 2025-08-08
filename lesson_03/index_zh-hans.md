**FFmpeg 汇编语言第三课**

让我们解释一些更多的术语，并为你提供一个简短的历史课程。

**指令集（Instruction Sets）**

你可能已经注意到在上一课中我们谈到了 SSE2，它是一组 SIMD 指令。当新一代 CPU 发布时，它可能会配备新的指令，有时还会有更大的寄存器尺寸。x86 指令集的历史非常复杂，所以这里是一个简化的历史（还有更多的子类别）：

* MMX - 1997年推出，Intel处理器中的首个SIMD，64位寄存器，历史悠久
* SSE (Streaming SIMD Extensions) - 1999年推出，128位寄存器
* SSE2 - 2000年推出，许多新指令
* SSE3 - 2004年推出，首个水平（horizontal）指令
* SSSE3 (Supplemental SSE3) - 2006年推出，新指令，但最重要的是 pshufb 洗牌（shuffle）指令，可以说是视频处理中最重要的指令
* SSE4 - 2008年推出，许多新指令，包括打包最小值和最大值
* AVX - 2011年推出，256位寄存器（仅浮点）和新的三操作数语法
* AVX2 - 2013年推出，整数指令的256位寄存器
* AVX512 - 2017年推出，512位寄存器，新的操作掩码特性。当时在 FFmpeg 中使用有限，因为使用新指令时CPU频率会降低。具有完整的512位洗牌（排列）功能，使用 vpermb
* AVX512ICL - 2019年推出，不再有时钟频率降低问题
* AVX10 - 即将推出

值得注意的是，指令集可以从 CPU 中移除，也可以添加。例如，AVX512 在第12代 Intel CPU 中被[移除](https://www.igorslab.de/en/intel-deactivated-avx-512-on-alder-lake-but-fully-questionable-interpretation-of-efficiency-news-editorial/)，而引起了争议。因此，FFmpeg 会进行运行时 CPU 检测（runtime CPU detection）。FFmpeg 会检测正在运行的 CPU 的能力。

正如你在作业中看到的，函数指针默认为 C 语言风格，但会被特定指令集变体替换。这意味着检测只需进行一次，之后无需再次进行。这与许多专有软件形成对比，后者硬编码特定指令集，以至于即使是功能完善的计算机也因此被淘汰。这也允许在运行时开启/关闭优化函数。这是开源的一大好处。

像 FFmpeg 这样的程序正运行在全球数十亿设备上，其中一些可能非常老旧。FFmpeg 技术上支持仅支持 SSE 的机器，这些机器已有25年历史！幸好，x86inc.asm 能够告诉你是否使用了特定指令集中不可用的指令。

为了让你了解真实世界的能力，以下是截至2024年11月 [Steam 调查](https://store.steampowered.com/hwsurvey/Steam-Hardware-Software-Survey-Welcome-to-Steam) 的指令集可用性（这显然偏向于游戏玩家）：

| 指令集 | 可用性 |
| :---- | :---- |
| SSE2 | 100% |
| SSE3 | 100% |
| SSSE3 | 99.86% |
| SSE4.1 | 99.80% |
| AVX | 97.39% |
| AVX2 | 94.44% |
| AVX512 (Steam 未区分 AVX512 和 AVX512ICL) | 14.09% |

对于像 FFmpeg 这样有数十亿用户的软件，即使0.1%也是非常大的用户量，如果出现问题就会有大量的错误报告。FFmpeg 在我们的 [FATE 测试套件](https://fate.ffmpeg.org/?query=subarch:x86_64%2F%2F) 中有广泛的测试基础设施，用于测试 CPU/操作系统/编译器的各种变体。每一个提交的代码都会在数百台机器上运行，以确保没有问题发生。

Intel 在此处提供了详细的指令集手册：[https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)

在 PDF 中搜索可能很麻烦，所以这里有一个非官方的基于网页的替代方案：[https://www.felixcloutier.com/x86/](https://www.felixcloutier.com/x86/)

这里还有一个SIMD指令的可视化表示：
[https://www.officedaytime.com/simd512e/](https://www.officedaytime.com/simd512e/)

x86 汇编的一部分挑战是找到适合你需求的正确指令。在某些情况下，指令可以以非原本设计的方式使用。

**指针偏移技巧**

让我们回到第一课中原来的函数，但给 C 函数添加一个宽度参数。

我们对宽度变量使用 ptrdiff_t 而不是 int，以确保64位参数的高32位为零。如果我们在函数签名中直接传递 int 类型的宽度，并尝试将其用作指针算术的四字（即：使用 `widthq`），那么寄存器的高32位可能会填充任意值。我们可以通过使用`movsxd`（另见 x86inc.asm 中的宏 `movsxdifnidn`）对宽度进行符号扩充来修复这个问题，但这是一种更简单的方法。

下面的函数包含指针偏移技巧：

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

让我们一步步解析这个有点令人困惑的过程：

```assembly
   add srcq, widthq
   add src2q, widthq
   neg widthq
```

宽度被添加到每个指针上，使每个指针现在指向要处理的缓冲区的末尾。然后宽度被取负。

```assembly
    movu  m0, [srcq+widthq]
    movu  m1, [src2q+widthq]
```

然后使用 widthq（此时为负值）进行加载。因此，在第一次迭代中，[srcq+widthq] 指向 srcq 的原始地址，即指回缓冲区的开头。

```assembly
    add   widthq, mmsize
    jl .loop
```

mmsize 被添加到负值的 widthq 中，使其接近零。循环条件现在是 jl（如果小于零则跳转）。这个技巧意味着 widthq 同时被用作指针偏移**和**循环计数器，从而节省了 cmp 指令。它还允许在多个加载和存储中使用指针偏移，以及在需要时使用指针偏移的倍数（请记住这一点，用于作业）。

**对齐（Alignment）**

在我们所有的例子中，我们一直使用 movu 来避免对齐（alignment）这个话题。如果数据是对齐的，即内存地址可以被 SIMD 寄存器大小整除，许多 CPU 可以更快地加载和存储数据。在可能的情况下，我们尝试在 FFmpeg 中使用 mova 进行对齐加载和存储。

在 FFmpeg 中，av_malloc 能够在堆（heap）上提供对齐的内存，而 DECLARE_ALIGNED C 预处理器指令可以在栈（stack）上提供对齐的内存。如果使用未对齐的地址调用 mova，将导致段错误（segmentation fault），软件会崩溃。确保对齐值与SIMD寄存器大小相对应也很重要，即 xmm 为16，ymm 为32，zmm 为64。

以下是如何将 RODATA 部分的开头对齐到64字节：

```assembly
SECTION_RODATA 64
```

请注意，这只是对齐 RODATA 的开头。可能需要填充字节来确保下一个标签保持在64字节边界上。

**范围扩充（Range expansion）**

我们到现在为止还避免讨论的另一个话题是溢出（overflowing）。例如，当字节值在加法或乘法等操作后超过255时，就会发生这种情况。我们可能想执行一个操作，其中需要一个比字节更大的中间值（例如字），或者我们可能希望将数据保留在那个更大的中间值的大小中。

对于无符号字节，这就是 punpcklbw（packed unpack high bytes to words，打包解压低字节到字）和 punpckhbw（packed unpack high bytes to words，打包解压高字节到字）发挥作用的地方。

让我们看看 punpcklbw 如何工作。Intel 手册中 SSE2 版本的语法如下：


| PUNPCKLBW xmm1, xmm2/m128 |
| :---- |


这意味着其源（右侧）可以是 xmm 寄存器或内存地址（m128 表示具有标准 [base + scale*index + disp] 语法的内存地址），而目标是xmm寄存器。

上面的 officedaytime.com 网站有一个很好的图表展示了发生的情况：

![图片描述](image1.png)

你可以看到字节是从每个寄存器的下半部分交错的。但这与范围扩充有什么关系呢？如果 src 寄存器全为零，这会将 dst 中的字节与零交错。这就是所谓的*零扩充*，因为字节是无符号的。punpckhbw 可以用于对高字节执行相同的操作。

这段代码展示这是如何做到的：

```assembly
pxor      m2, m2 ; zero out m2

movu      m0, [srcq]
movu      m1, m0 ; make a copy of m0 in m1
punpcklbw m0, m2
punpckhbw m1, m2
```

```m0``` 和 ```m1``` 现在包含已被零扩充为字的原始字节。在下一课中，你将看到 AVX 中的三操作数（three-operand ）指令如何使第二个 movu 变得不必要。


**符号扩充（Sign extension）**

有符号数据稍微复杂一些。要扩充有符号整数的范围，我们需要使用一个称为 [符号扩充](https://zh.wikipedia.org/wiki/%E7%AC%A6%E5%8F%B7%E6%89%A9%E5%85%85) 的过程。这会用符号位填充 MSB。例如：int8_t 中的-2是 0b11111110。要将其符号扩充到 int16_t，MSB 的1被重复以生成 0b1111111111111110。

```pcmpgtb``` (packed compare greater than byte) can be used for sign extension. By doing the comparison (0 > byte), all the bits in the destination byte are set to 1 if the byte is negative, otherwise the bits in the destination byte are set to 0. punpckX can be used as above to perform the sign extension. If the byte is negative the corresponding byte is 0b11111111 and otherwise it’s 0x00000000. Interleaving the byte value with the output of pcmpgtb performs a sign extension to word as a result.

```pcmpgtb```（packed compare greater than byte，打包比较大于字节）可用于符号扩充。通过执行比较（0 > 字节），如果字节为负，则目标字节中的所有位都设置为1，否则目标字节中的位设置为0。punpckX 可以如上所述用于执行符号扩充。如果字节为负，则相应的字节为 0b11111111，否则为 0x00000000。将字节值与 pcmpgtb 的输出交错来实现字扩充。

```assembly
pxor      m2, m2 ; 将 m2 清零

movu      m0, [srcq]
movu      m1, m0 ; 将 m0 复制到 m1

pcmpgtb   m2, m0
punpcklbw m0, m2
punpckhbw m1, m2
```

如你所见，与无符号情况相比多了一条额外的指令。

**打包（Packing）**

packuswb（pack unsigned word to byte，打包无符号字到字节）和 packsswb 都允许你从字转到字节。它允许你将包含字的两个 SIMD 寄存器交错到一个包含字节的 SIMD 寄存器中。注意，如果值超出字节范围，它们将被饱和（即限制在最大值）。

**洗牌（Shuffles）**

洗牌，也称为置换，可以说是视频处理中最重要的指令，而 pshufb（packed shuffle bytes，打包洗牌字节），在 SSSE3 中可用，是最重要的变体。

对于每个字节，相应的源字节用作目标寄存器的索引，除非设置了 MSB，否则目标字节将被清零。这类似于以下 C 代码（尽管在SIMD 中，所有16个循环迭代同时进行）：

```c
for(int i = 0; i < 16; i++) {
    if(src[i] & 0x80)
        dst[i] = 0;
    else
        dst[i] = dst[src[i]]
}
```
这是一个简单的汇编示例：

```assembly
SECTION_DATA 64

shuffle_mask: db 4, 3, 1, 2, -1, 2, 3, 7, 5, 4, 3, 8, 12, 13, 15, -1

section .text

movu m0, [srcq]
movu m1, [shuffle_mask]
pshufb m0, m1 ; 根据 m0 洗牌 m1
```

注意，-1被用作洗牌索引以清零输出字节，便于阅读：字节形式的-1是 0b11111111 位场（bitfield）（二进制补码），因此MSB（0x80）被设置。

[image1]: <data:image/png;base64,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>