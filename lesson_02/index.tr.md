**FFmpeg Assembly Dili İkinci Ders**

Artık ilk assembly dili fonksiyonunuzu yazdığınıza göre, şimdi dallanma ve döngüleri tanıtacağız.

Önce etiket ve atlama fikrini tanıtmamız gerekiyor. Aşağıdaki yapay örnekte, jmp talimatı kod talimatını ".loop:" dan sonrasına taşır. ".loop:" *etiket* olarak bilinir, etiketin önündeki nokta bunun *yerel etiket* olduğu anlamına gelir, bu da aynı etiket adını birden fazla fonksiyon boyunca yeniden kullanmanıza olanak tanır. Bu örnek tabii ki sonsuz bir döngü gösteriyor, ancak bunu daha sonra daha gerçekçi bir şeye genişleteceğiz.

```assembly
mov  r0q, 3
.loop:
    dec  r0q
    jmp .loop
```

Gerçekçi bir döngü yapmadan önce *FLAGS* register'ını tanıtmamız gerekiyor. *FLAGS*'ın inceliklerine çok fazla dalacağımız yok (yine GPR işlemleri büyük ölçüde iskelet olduğu için) ancak Sıfır-Bayrağı, İşaret-Bayrağı ve Taşma-Bayrağı gibi birkaç bayrak vardır ve bunlar skalar veri üzerinde aritmetik işlemler ve kaydırmalar gibi mov olmayan talimatların çıktısına göre ayarlanır.

İşte döngü sayacının sıfıra kadar geriye saydığı ve jg'nin (sıfırdan büyükse atla) döngü koşulu olduğu bir örnek. dec r0q talimat sonrasında r0q değerine göre FLAGS'ı ayarlar ve bunlara göre atlayabilirsiniz.

```assembly
mov  r0q, 3
.loop:
    ; bir şey yap
    dec  r0q
    jg  .loop ; sıfırdan büyükse atla
```

Bu aşağıdaki C koduna eşdeğerdir:

```c
int i = 3;
do
{
   // bir şey yap
   i--;
} while(i > 0)
```

Bu C kodu biraz doğal değil. Genellikle C'de bir döngü şöyle yazılır:

```c
int i;
for(i = 0; i < 3; i++) {
    // bir şey yap
}
```

Bu kabaca şuna eşdeğerdir (bu ```for``` döngüsünü eşleştirmenin basit bir yolu yoktur):

```assembly
xor r0q, r0q
.loop:
    ; bir şey yap
    inc r0q
    cmp r0q, 3
    jl  .loop ; (r0q - 3) < 0 ise atla, yani (r0q < 3)
```

Bu parçacıkta işaret edilecek birkaç şey vardır. İlki ```xor r0q, r0q``` olup, bu bir register'ı sıfıra ayarlamanın yaygın bir yoludur ve bazı sistemlerde ```mov r0q, 0```'dan daha hızlıdır, çünkü basitçe söylemek gerekirse gerçek bir yükleme gerçekleşmez. ```pxor m0, m0``` ile SIMD register'larında tüm register'ı sıfırlamak için de kullanılabilir. Dikkat edilecek bir diğer şey ise cmp kullanımıdır. cmp etkili olarak ikinci register'ı birinciden çıkarır (değeri herhangi bir yerde saklamadan) ve *FLAGS*'ı ayarlar, ancak yorumda da belirtildiği gibi, ```r0q < 3``` ise atlamak için atlama ile birlikte okunabilir (jl = sıfırdan küçükse atla).

Bu parçacıkta bir ekstra talimat (cmp) olduğuna dikkat edin. Genel olarak, daha az talimat daha hızlı kod anlamına gelir, bu yüzden önceki parçacık tercih edilir. Gelecek derslerde göreceğiniz gibi, bu ekstra talimatı önlemek ve *FLAGS*'ın aritmetik veya başka bir işlemle ayarlanmasını sağlamak için daha fazla numara kullanılır. Assembly'de C döngülerini tam olarak eşleştirmek için assembly yazmadığımıza, assembly'de mümkün olduğunca hızlı hale getirmek için döngüler yazdığımıza dikkat edin.

İşte kullanacağınız bazı yaygın atlama mnemonikleri (*FLAGS* eksiksizlik için oradadır, ancak döngü yazmak için ayrıntıları bilmenize gerek yoktur):

| Mnemonic | Açıklama  | FLAGS |
| :---- | :---- | :---- |
| JE/JZ | Eşitse/Sıfırsa Atla | ZF = 1 |
| JNE/JNZ | Eşit Değilse/Sıfır Değilse Atla | ZF = 0 |
| JG/JNLE | Büyükse/Küçük veya Eşit Değilse Atla (işaretli) | ZF = 0 and SF = OF |
| JGE/JNL | Büyük veya Eşitse/Küçük Değilse Atla (işaretli) | SF = OF |
| JL/JNGE | Küçükse/Büyük veya Eşit Değilse Atla (işaretli) | SF ≠ OF |
| JLE/JNG | Küçük veya Eşitse/Büyük Değilse Atla (işaretli) | ZF = 1 or SF ≠ OF |

**Sabitler**

Sabitlerin nasıl kullanılacağını gösteren bazı örneklere bakalım:

```assembly
SECTION_RODATA

constants_1: db 1,2,3,4
constants_2: times 2 dw 4,3,2,1
```

* SECTION_RODATA bunun salt okunur veri bölümü olduğunu belirtir. (Bu bir makrodur çünkü işletim sistemlerinin kullandığı farklı çıktı dosya formatları bunu farklı şekilde beyan eder)
* constants_1: constants_1 etiketi ```db``` (declare byte - byte beyan et) olarak tanımlanır - yani uint8_t constants_1[4] = {1, 2, 3, 4}; eşdeğeri
* constants_2: Bu, beyan edilen word'leri tekrarlamak için ```times 2``` makrosunu kullanır - yani uint16_t constants_2[8] = {4, 3, 2, 1, 4, 3, 2, 1}; eşdeğeri

Derleyicinin bir bellek adresine dönüştürdüğü bu etiketler daha sonra yüklemelerde kullanılabilir (salt okunur oldukları için saklamalarda değil). Bazı talimatlar operand olarak bir bellek adresi alır, bu nedenle register'a açık yükleme olmadan kullanılabilirler (bunun artıları ve eksileri vardır).

**Offset'ler**

Offset'ler bellekteki ardışık öğeler arasındaki mesafedir (byte olarak). Offset, veri yapısındaki **her öğenin boyutu** ile belirlenir.

Artık döngü yazabildiğimize göre, veri almaya zaman. Ancak C ile karşılaştırıldığında bazı farklar var. Şu C döngüsüne bakalım:

```c
uint32_t data[3];
int i;
for(i = 0; i < 3; i++) {
    data[i];
}
```

Veri öğeleri arasındaki 4-byte offset C derleyicisi tarafından önceden hesaplanır. Ancak assembly'yi elle yazarken bu offset'leri kendiniz hesaplamanız gerekir.

Bellek adresi hesaplamaları için sözdizimini inceleyelim. Bu her türlü bellek adresi için geçerlidir:

```assembly
[base + scale*index + disp]
```

* base - Bu bir GPR'dır (genellikle bir C fonksiyon argümanından gelen pointer)
* scale - Bu 1, 2, 4, 8 olabilir. 1 varsayılandır
* index - Bu bir GPR'dır (genellikle bir döngü sayacı)
* disp - Bu bir integer'dır (32-bit'e kadar). Displacement veri içine bir offset'tir

x86asm, çalıştığınız SIMD register'ının boyutunu bilmenizi sağlayan mmsize sabitini sağlar.

Özel offset'lerden yüklemeyi göstermek için basit (ve anlamsız) bir örnek:

```assembly
;static void simple_loop(const uint8_t *src)
INIT_XMM sse2
cglobal simple_loop, 1, 2, 2, src
     movq r1q, 3
.loop:
     movu m0, [srcq]
     movu m1, [srcq+2*r1q+3+mmsize]

     ; bazı şeyler yap

     add srcq, mmsize
dec r1q
jg .loop

RET
```

```movu m1, [srcq+2*r1q+3+mmsize]```'de derleyicinin kullanılacak doğru displacement sabitini önceden hesaplayacağına dikkat edin. Bir sonraki derste döngüde add ve dec yapmaktan kaçınma numarasını göstereceğiz, bunları tek bir add ile değiştireceğiz.

**LEA**

Artık offset'leri anladığınıza göre lea (Load Effective Address) kullanabilirsiniz. Bu, çarpma ve toplamayı tek talimat ile yapmanıza olanak tanır, bu da birden fazla talimat kullanmaktan daha hızlı olacaktır. Tabii ki çarpabileceğiniz ve toplayabileceğiniz şeylerde sınırlamalar vardır ancak bu lea'nın güçlü bir talimat olmasını engellemez.

```assembly
lea r0q, [base + scale*index + disp]
```

Adına rağmen, LEA normal aritmetik için ve adres hesaplamaları için kullanılabilir. Şu kadar karmaşık bir şey yapabilirsiniz:

```assembly
lea r0q, [r1q + 8*r2q + 5]
```

Bunun r1q ve r2q içeriklerini etkilemediğine dikkat edin. Ayrıca *FLAGS*'ı da etkilemez (bu nedenle çıktıya göre atlayamazsınız). LEA kullanmak tüm bu talimatları ve geçici register'ları önler (bu kod eşdeğer değildir çünkü add *FLAGS*'ı değiştirir):

```assembly
movq r0q, r1q
movq r3q, r2q
sal  r3q, 3 ; shift arithmetic left 3 = * 8
add  r3q, 5
add  r0q, r3q
```

lea'nın döngülerden önce adresler ayarlamak veya yukarıdaki gibi hesaplamalar yapmak için çok kullanıldığını göreceksiniz. Tabii ki her türlü çarpma ve toplama yapamazsınız, ancak 1, 2, 4, 8 ile çarpmalar ve sabit bir offset toplamı yaygındır.

Ödevde bir sabit yüklemeniz ve değerleri bir döngüde bir SIMD vektörüne eklemeniz gerekecek.

[Sonraki Ders](../lesson_03/index.md)
