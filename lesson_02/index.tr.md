**FFmpeg Assembly Dili İkinci Ders**

Assembly dilinde ilk fonksiyonunuzu yazdığınıza göre, şimdi dallanma ve döngüleri tanıtacağız.

Öncelikle etiketler ve atlamalar kavramını tanıtmamız gerekiyor. Aşağıdaki yapay örnekte, jmp komutu kod komutunu “.loop:” sonrasına taşır. “. loop:" bir *etiket* olarak bilinir ve etiketin önündeki nokta, bunun bir *yerel etiket* olduğunu gösterir, böylece aynı etiket adını birden fazla fonksiyonda yeniden kullanabilirsiniz. Bu örnekte elbette sonsuz bir döngü gösterilmektedir, ancak bunu daha sonra daha gerçekçi bir şeye genişleteceğiz.

```assembly
mov  r0q, 3
.loop:
    dec  r0q
    jmp .loop
```

Gerçekçi bir döngü oluşturmadan önce *FLAGS* kaydını tanıtmamız gerekiyor. *FLAGS*'in karmaşıklığına yine GPR işlemleri büyük ölçüde iskele görevi gördüğü için çok fazla girmeyeceğiz ancak aritmetik işlemler ve kaydırmalar gibi skaler veriler üzerindeki çoğu non-mov komutunun çıktısına göre ayarlanan Zero-Flag, Sign-Flag ve Overflow-Flag gibi birkaç bayrak vardır.

İşte döngü sayacı sıfıra kadar geri sayarken jg (sıfırdan büyükse atla) döngü koşulu olan bir örnek. dec r0q, komuttan sonra r0q değerine göre FLAG'leri ayarlar ve bunlara göre atlama yapabilirsiniz.

```assembly
mov  r0q, 3
.loop:
    ; do something
    dec  r0q
    jg  .loop ; jump if greater than zero
```

Bu, aşağıdaki C koduna eşdeğerdir:

```c
int i = 3;
do
{
   // do something
   i--;
} while(i > 0)
```

Bu C kodu çok da doğal olmayan bir yapıya sahiptir. Genellikle C dilinde bir döngü şu şekilde yazılır:

```c
int i;
for(i = 0; i < 3; i++) {
    // do something
}
```

Bu, kabaca şuna eşdeğerdir (```for``` döngüsünü eşleştirmek için basit bir yol yoktur):

```assembly
xor r0q, r0q
.loop:
    ; do something
    inc r0q
    cmp r0q, 3
    jl  .loop ; jump if (r0q - 3) < 0, i.e (r0q < 3)
```

Bu kod parçasında dikkat edilmesi gereken birkaç nokta var. İlki, ```xor r0q, r0q``` komutudur. Bu, bir kaydı sıfıra ayarlamanın yaygın bir yoludur ve bazı sistemlerde ```mov r0q, 0``` komutundan daha hızlıdır, çünkü basitçe söylemek gerekirse, gerçek bir yükleme işlemi gerçekleşmez. Bu komut, SIMD kayıtlarında ```pxor m0, m0``` ile birlikte kullanılarak tüm kaydı sıfırlayabilir. Dikkat edilmesi gereken bir diğer nokta ise cmp kullanımıdır. cmp, ikinci kaydı ilk kayıttan etkili bir şekilde çıkarır (değeri hiçbir yere kaydetmeden) ve *FLAGS*'ı ayarlar, ancak yorumda belirtildiği gibi, ```r0q < 3``` ise atlamak için atlama ile birlikte okunabilir (jl = sıfırdan küçükse atlama).

Bu kod parçasında fazladan bir komut (cmp) olduğunu unutmayın. Genel olarak, komut sayısı ne kadar azsa kod o kadar hızlıdır, bu nedenle önceki kod parçası tercih edilir. İleriki derslerde göreceğiniz gibi, bu fazladan komutu önlemek ve *FLAGS*'ı aritmetik veya başka bir işlemle ayarlamak için kullanılan başka püf noktaları da vardır. C döngüleriyle tam olarak eşleşecek şekilde assembly yazmadığımızı, döngüleri assembly'de olabildiğince hızlı hale getirmek için yazdığımızı unutmayın.

İşte kullanacağınız bazı yaygın atlama mnemonikleri (*FLAGS* tamlık için vardır, ancak döngü yazmak için ayrıntıları bilmenize gerek yoktur):

| Mnemonic | Description  | FLAGS |
| :---- | :---- | :---- |
| JE/JZ | Jump if Equal/Zero | ZF = 1 |
| JNE/JNZ | Jump if Not Equal/Not Zero | ZF = 0 |
| JG/JNLE | Jump if Greater/Not Less or Equal (signed) | ZF = 0 and SF = OF |
| JGE/JNL | Jump if Greater or Equal/Not Less (signed) | SF = OF |
| JL/JNGE | Jump if Less/Not Greater or Equal (signed) | SF ≠ OF |
| JLE/JNG | Jump if Less or Equal/Not Greater (signed) | ZF = 1 or SF ≠ OF |

**Sabitler**

Sabitleri nasıl kullanacağınıza dair bazı örneklere bakalım:

```assembly
SECTION_RODATA

constants_1: db 1,2,3,4
constants_2: times 2 dw 4,3,2,1
```

* SECTION_RODATA, bunun salt okunur bir veri bölümü olduğunu belirtir. (Bu bir makrodur, çünkü işletim sistemlerinin kullandığı farklı çıktı dosyası biçimleri bunu farklı şekilde bildirir.)
* constants_1: constants_1 etiketi, ```db``` (bayt bildirimi) olarak tanımlanır - yani uint8_t constants_1[4] = {1, 2, 3, 4}; ile eşdeğerdir.
* constants_2: Bu, ```times 2``` makrosunu kullanarak bildirilen kelimeleri tekrarlar - yani uint16_t constants_2[8] = {4, 3, 2, 1, 4, 3, 2, 1}; ile eşdeğerdir.

Derleyici tarafından bellek adresine dönüştürülen bu etiketler yüklemelerde kullanılabilir ancak salt okunur oldukları için depolamalarda kullanılamazlar. Bazı komutlar, bellek adresini işlenen olarak alırlar, bu nedenle kayıtlara açıkça yüklenmeden kullanılabilirler ki bunun avantajları ve dezavantajları vardır.

**Ofsetler**

Ofsetler, bellekteki ardışık öğeler arasındaki mesafedir (bayt cinsinden). Ofset, veri yapısındaki **her bir öğenin boyutu** tarafından belirlenir.

Artık döngüler yazabildiğimize göre, veri almaya geçebiliriz. Ancak C ile bazı farklılıklar vardır. C'deki aşağıdaki döngüye bakalım:

```c
uint32_t data[3];
int i;
for(i = 0; i < 3; i++) {
    data[i];
}
```

Veri öğeleri arasındaki 4 baytlık ofset, C derleyicisi tarafından önceden hesaplanır. Ancak, elle assembler yazarken bu ofsetleri kendiniz hesaplamanız gerekir.

Bellek adresi hesaplamaları için sözdizimine bakalım. Bu, tüm bellek adresi türleri için geçerlidir:

```assembly
[base + scale*index + disp]
```

* base - Bu bir GPR'dır (genellikle bir C fonksiyon argümanından gelen bir işaretçi)
* scale - Bu 1, 2, 4, 8 olabilir. Varsayılan olarak 1'dir.
* index - Bu bir GPR'dır. (genellikle bir döngü sayacı)
* disp - Bu bir tam sayıdır (32 bit'e kadar). Yer değiştirme (displacement), verilerdeki bir kaydırmadır.

x86asm, çalıştığınız SIMD kaydının boyutunu öğrenmenizi sağlayan mmsize sabitini sağlar.

Özel ofsetlerden yüklemeyi açıklamak için basit ve anlamsız bir örnek verelim:

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

```movu m1, [srcq+2*r1q+3+mmsize]``` komutunda, assembler kullanılacak doğru yer değiştirme sabitini önceden hesaplayacaktır. Bir sonraki derste döngüde toplama ve çıkarma işlemleri yapmak zorunda kalmamak için bunları tek bir toplama işlemiyle değiştirebileceğiniz bir püf noktası göstereceğiz.

**LEA**

Ofsetleri anladıktan sonra, lea (Load Effective Address) komutunu kullanabilirsiniz. Bu komut, tek bir komutla çarpma ve toplama işlemlerini gerçekleştirmenizi sağlar, bu da birden fazla komut kullanmaktan daha hızlıdır. Elbette, çarpma ve toplama işlemlerinde bazı sınırlamalar vardır, ancak bu, lea komutunun güçlü bir komut olmasını engellemez.

```assembly
lea r0q, [base + scale*index + disp]
```

Adının aksine, LEA normal aritmetik işlemlerin yanı sıra adres hesaplamaları için de kullanılabilir. Şu kadar karmaşık bir işlem yapabilirsiniz:

```assembly
lea r0q, [r1q + 8*r2q + 5]
```

Bunun r1q ve r2q'nun içeriğini etkilemediğini unutmayın. Ayrıca *FLAGS*'ı da etkilemez (bu nedenle çıktıya göre atlama yapamazsınız). LEA kullanmak tüm bu komutları ve geçici kayıtları önler (bu kod eşdeğer değildir çünkü add *FLAGS*'ı değiştirir):

```assembly
movq r0q, r1q
movq r3q, r2q
sal  r3q, 3 ; shift arithmetic left 3 = * 8
add  r3q, 5
add  r0q, r3q
```

Döngülerden önce adresleri ayarlamak veya yukarıdaki gibi hesaplamalar yapmak için lea'in sıklıkla kullanıldığını göreceksiniz. Elbette her tür çarpma ve toplama işlemini yapamayacağınızı unutmayın, ancak 1, 2, 4, 8 ile çarpma ve sabit bir ofsetin toplama işlemi yaygın olarak kullanılır.

Ödevde, bir sabiti yüklemeniz ve değerleri bir döngü içinde SIMD vektörüne eklemeniz gerekecektir.

[Sonraki Ders](../lesson_03/index.tr.md)
