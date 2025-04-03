**FFmpeg 組合語言第二課**

現在你已經寫過了第一個組合函式，我們開始介紹分支（branches）和迴圈（loops）。

我們先介紹標籤（labels）與跳轉（jumps）的概念。在下方建構的例子裡，jmp（跳轉）指令將程式碼執行流程移動到 ".loop:" 指令下方。".loop:" 就是一個*標籤*，標籤字首的點（.）表示它是一個*本地標籤*，這樣就可以在多個函式中重複使用同一個標籤名稱。當然，這個例子展示的是一個無限迴圈，但我們稍後會將其擴展到更實際的場景。

```assembly
mov  r0q, 3
.loop:
    dec  r0q
    jmp .loop
```

在開始實際迴圈之前，我們必須先介紹 *FLAGS* (旗標)暫存器。我們不會過多討論 *FLAGS* (旗標)的複雜性（同樣因為 GPR 操作主要是鷹架），但有幾個旗標（flags），如零旗標（Zero-Flag）、符號旗標（Sign-Flag）和溢位旗標（Overflow-Flag），這些旗標是根據大多數非移動（non-mov）指令對標量資料（如算術運算和位移）的輸出設定的。

下面的範例中，迴圈計數器倒數到 0，jg（**j**ump if **g**reater than zero，大於 0 則跳轉）是迴圈條件。dec r0q 指令會根據該指令執行後 r0q 的值設定 FLAGS，然後你可以根據這些 FLAGS 進行跳轉。

```assembly
mov  r0q, 3
.loop:
    ; do something
    dec  r0q
    jg  .loop ; jump if greater than zero
```

它等同於如下 C 程式碼：

```c
int i = 3;
do
{
   // do something
   i--;
} while(i > 0)
```

這段 C 程式碼有點不太自然。一般來說 C 語言裡迴圈功能是這麼寫的：

```c
int i;
for(i = 0; i < 3; i++) {
    // do something
}
```

這段大致等同於（沒有簡單的方法來完全匹配這個 ```for``` 迴圈）：

```assembly
xor r0q, r0q
.loop:
    ; do something
    inc r0q
    cmp r0q, 3
    jl  .loop ; jump if (r0q - 3) < 0, i.e (r0q < 3)
```

這段程式碼有幾點值得注意。首先是 ```xor r0q, r0q```，這是一種常見的將暫存器清零的方法，在某些系統上比 ```mov r0q, 0``` 更快，簡而言之，這是因為不需要進行實際的載入操作。這也可以用於 SIMD 暫存器，通過 ```pxor m0, m0``` 來將整個暫存器清零。另一個需要注意的是 cmp (compare，比較)的使用。cmp 實際上是從第一個暫存器中減去第二個暫存器的值（但不儲存結果）並設定 **FLAGS**，但根據註解所示，它可以與跳轉指令一起理解，（jl = **j**ump if **l**ess than zero，如果小於零則跳轉）以在 ```r0q < 3``` 時進行跳轉。

請注意，在這個程式碼片段中有一條額外的指令（cmp）。一般來說，指令越少意味著程式碼執行越快，這就是為什麼之前的程式碼片段更受青睞。正如你將在後續課程中看到的，還有更多技巧可以用來避免這種額外指令，並利用算術或其他操作來設定 **FLAGS** 暫存器。需要注意的是，我們編寫組合程式碼時並不是為了完全匹配 C 語言的迴圈結構，而是為了在組合語言中讓迴圈盡可能地高效。

以下是一些你將會用到的常見跳轉助記符（jump mnemonics）（*FLAGS* 在此列出只是為了完整性，你不需要了解具體細節就能編寫迴圈）：

| 助記符 | 描述  | FLAGS |
| :---- | :---- | :---- |
| JE/JZ | 相等/為零時跳轉 | ZF = 1 |
| JNE/JNZ | 不相等/不為零時跳轉 | ZF = 0 |
| JG/JNLE | 大於/不小於等於時跳轉（有符號） | ZF = 0 and SF = OF |
| JGE/JNL | 大於等於/不小於時跳轉（有符號） | SF = OF |
| JL/JNGE | 小於/不大於等於時跳轉（有符號） | SF ≠ OF |
| JLE/JNG | 小於等於/不大於時跳轉（有符號） | ZF = 1 or SF ≠ OF |

**常數（Constants）**

讓我們看幾個展示如何使用常數的例子：

```assembly
SECTION_RODATA

constants_1: db 1,2,3,4
constants_2: times 2 dw 4,3,2,1
```

* SECTION_RODATA 指定這是一個唯讀資料段（read-only data section）。（這是一個巨集，因為不同作業系統使用不同的宣告方式輸出檔案格式）
* constants_1：標籤 constants_1 通過 ```db```（宣告位元組，declare byte）定義 - 相當於 uint8_t constants_1[4] = {1, 2, 3, 4};
* constants_2：這裡使用 ```times 2``` 巨集來重複宣告的字（words）- 相當於 uint16_t constants_2[8] = {4, 3, 2, 1, 4, 3, 2, 1};

這些標籤會被組譯器轉換為記憶體位址，然後可以用於載入操作（但由於是唯讀的，不能用於儲存操作）。某些指令可以直接將記憶體位址作為運算元（operand），而無需顯式地先載入到暫存器中（這種做法既有優點也有缺點）。

**偏移量（Offsets）**

偏移量是記憶體中連續元素之間的距離（以位元組為單位）。偏移量由資料結構中**每個元素的大小**決定。

現在我們已經能夠編寫迴圈，是時候獲取資料了。但這與 C 語言相比有一些區別。讓我們看看以下例子中的 C 語言迴圈：

```c
uint32_t data[3];
int i;
for(i = 0; i < 3; i++) {
    data[i];
}
```

C 編譯器會預先計算出陣列元素之間的 4 位元組偏移量。但在手寫組合程式碼時，你需要自己計算這些偏移量。

讓我們看看記憶體位址計算的語法。這適用於所有類型的記憶體位址：

```assembly
[base + scale*index + disp]
```

* base - 這是一個通用暫存器（GPR）（通常是 C 函式的參數的指標）
* scale - 這可以是 1、2、4 或 8。預設值為 1
* index - 這是一個通用暫存器（GPR）（通常是迴圈計數器）
* disp - 這是一個整數（最大 32 位元）。位移（Displacement）是資料的偏移量

x86asm 提供了常數 mmsize，它讓你知道正在使用的 SIMD 暫存器的大小。

以下是一個簡單的（沒有實際意義的）例子，用來說明如何從自訂偏移量載入資料：

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

請注意在 ```movu m1, [srcq+2*r1q+3+mmsize]``` 中，組譯器會預先計算出正確的位移常數。在下一課中，我們將介紹一個技巧，可以避免在迴圈中同時使用 add 和 dec 指令，而是用單個 add 指令來替代它們。

**LEA**

現在你已經理解了偏移量，你可以使用 lea（載入有效位址，Load Effective Address）指令。它能讓你用一條指令完成乘法和加法運算，這比使用多條指令要快得多。當然，lea 對可以乘以什麼數和加上什麼值有一些限制，但這並不妨礙 lea 成為一個強大的指令。

```assembly
lea r0q, [base + scale*index + disp]
```

與其名稱相反，LEA 不僅可以用於位址計算，還可以用於普通的算術運算。你可以做一些複雜的操作，比如：

```assembly
lea r0q, [r1q + 8*r2q + 5]
```

請注意，這不會影響 r1q 和 r2q 的內容。它也不會影響 *FLAGS* 暫存器（所以你不能根據其輸出結果進行跳轉）。使用 LEA 可以避免所有這些指令和臨時暫存器的使用（這段程式碼不完全等價，因為 add 會改變 *FLAGS* 暫存器）：

```assembly
movq r0q, r1q
movq r3q, r2q
sal  r3q, 3 ; shift arithmetic left（左移運算） 3 = * 8
add  r3q, 5
add  r0q, r3q
```

你會看到 lea 經常被用來在迴圈前設定位址或執行上述計算。當然需要注意的是，你不能用 lea 執行所有類型的乘法和加法運算，但乘以 1、2、4、8 和添加固定偏移量是很常見的用法。

在作業中，你需要在迴圈中載入常數並將這些值添加到 SIMD 向量中。

[下一課](../lesson_03/index_zh-hans.md)
