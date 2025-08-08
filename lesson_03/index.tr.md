**FFmpeg Assembly Üçüncü Ders**

Biraz daha jargon açıklayalım ve size kısa bir tarih dersi verelim.

**Talimat Setleri**

Önceki derste SSE2'den bahsettiğimizi, bunun bir SIMD talimat seti olduğunu görmüş olabilirsiniz. Yeni bir CPU nesli piyasaya çıktığında yeni talimatlar ve bazen daha büyük register boyutları ile gelebilir. x86 talimat setinin tarihçesi çok karmaşıktır, bu nedenle bu basitleştirilmiş bir tarihçedir (çok daha fazla alt kategori vardır):

* MMX - 1997'de piyasaya çıktı, Intel İşlemcilerinde ilk SIMD, 64-bit register'lar, tarihsel
* SSE (Streaming SIMD Extensions) - 1999'da piyasaya çıktı, 128-bit register'lar
* SSE2 - 2000'de piyasaya çıktı, birçok yeni talimat
* SSE3 - 2004'te piyasaya çıktı, ilk yatay talimatlar
* SSSE3 (Supplemental SSE3) - 2006'da piyasaya çıktı, yeni talimatlar ancak en önemlisi pshufb shuffle talimatı, tartışmasız video işlemede en önemli talimat
* SSE4 - 2008'de piyasaya çıktı, paketlenmiş minimum ve maksimum dahil birçok yeni talimat.
* AVX - 2011'de piyasaya çıktı, 256-bit register'lar (sadece float) ve yeni üç-operand sözdizimi
* AVX2 - 2013'te piyasaya çıktı, integer talimatları için 256-bit register'lar
* AVX512 - 2017'de piyasaya çıktı, 512-bit register'lar, yeni işlem maskesi özelliği. Yeni talimatlar kullanıldığında CPU frekans düşürme nedeniyle o zamanlar FFmpeg'de sınırlı kullanımları vardı. vpermb ile tam 512-bit shuffle (permute).
* AVX512ICL - 2019'da piyasaya çıktı, artık clock frekans düşürme yok.
* AVX10 - Yakında

Talimat setlerinin CPU'lara eklenmesinin yanı sıra kaldırılabileceğini de belirtmek gerekir. Örneğin AVX512, 12. Nesil Intel CPU'larında [tartışmalı şekilde kaldırılmıştır](https://www.igorslab.de/en/intel-deactivated-avx-512-on-alder-lake-but-fully-questionable-interpretation-of-efficiency-news-editorial/). Bu nedenle FFmpeg çalışma zamanı CPU algılaması yapar. FFmpeg üzerinde çalıştığı CPU'nun yeteneklerini algılar.

Ödevde gördüğünüz gibi, fonksiyon pointer'ları varsayılan olarak C'dir ve belirli bir talimat seti varyantı ile değiştirilir. Bu, algılamanın bir kez yapıldığı ve bir daha asla yapılması gerekmediği anlamına gelir. Bu, belirli bir talimat setini hardcode eden ve mükemmel çalışan bir bilgisayarı eskiten birçok tescilli uygulamanın aksine. Bu ayrıca optimize edilmiş fonksiyonların çalışma zamanında açılıp/kapatılmasına olanak tanır. Bu, açık kaynak kodun büyük faydalarından biridir.

FFmpeg gibi programlar dünya çapında milyarlarca cihazda kullanılır, bunların bazıları çok eski olabilir. FFmpeg teknik olarak 25 yaşındaki sadece SSE destekleyen makineleri destekler! Neyse ki x86inc.asm belirli bir talimat setinde mevcut olmayan bir talimat kullanıp kullanmadığınızı söyleyebilir.

Gerçek dünya yetenekleri hakkında bir fikir vermek için, Kasım 2024 itibariyle [Steam Anketi](https://store.steampowered.com/hwsurvey/Steam-Hardware-Software-Survey-Welcome-to-Steam)'nden talimat seti kullanılabilirliği (bu açıkça oyunculara yönelik önyargılıdır):

| Talimat Seti | Kullanılabilirlik |
| :---- | :---- |
| SSE2 | %100 |
| SSE3 | %100 |
| SSSE3 | %99.86 |
| SSE4.1 | %99.80 |
| AVX | %97.39 |
| AVX2 | %94.44 |
| AVX512 (Steam AVX512 ve AVX512ICL arasında ayrım yapmıyor) | %14.09 |

Milyarlarca kullanıcısı olan FFmpeg gibi bir uygulama için, %0.1 bile çok büyük bir kullanıcı sayısıdır ve bir şey bozulursa hata raporları gelir. FFmpeg, [FATE test paketimizde](https://fate.ffmpeg.org/?query=subarch:x86_64%2F%2F) CPU/OS/Derleyici varyasyonlarını test etmek için kapsamlı test altyapısına sahiptir. Her commit hiçbir şeyin bozulmadığından emin olmak için yüzlerce makinede çalıştırılır.

Intel detaylı talimat seti kılavuzunu burada sağlar: [https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)

PDF'te arama yapmak zahmetli olabilir, bu nedenle burada resmi olmayan web tabanlı bir alternatif var: [https://www.felixcloutier.com/x86/](https://www.felixcloutier.com/x86/)

SIMD talimatlarının görsel temsili de burada mevcuttur:
[https://www.officedaytime.com/simd512e/](https://www.officedaytime.com/simd512e/)

x86 assembly'nin zorluklarından biri ihtiyaçlarınız için doğru talimatı bulmaktır. Bazı durumlarda talimatlar orijinal amaçlarından farklı şekilde kullanılabilir.

**Pointer offset numarası**

1. Dersten orijinal fonksiyonumuza geri dönelim, ancak C fonksiyonuna bir width argümanı ekleyelim.

Width değişkeni için int yerine ptrdiff_t kullanıyoruz, 64-bit argümanın üst 32-bitinin sıfır olduğundan emin olmak için. Eğer fonksiyon imzasında doğrudan bir int width geçirsek ve sonra bunu pointer aritmetiği için quad olarak kullanmaya çalışsak (`widthq` kullanarak) register'ın üst 32-biti rastgele değerlerle doldurulabilir. Bunu `movsxd` ile width'i işaret genişleterek düzeltebilirdik (x86inc.asm'daki `movsxdifnidn` makrosuna da bakın), ancak bu daha kolay bir yol.

Aşağıdaki fonksiyon pointer offset numarasını içerir:

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

Kafa karıştırabileceği için bunu adım adım gözden geçirelim:

```assembly
   add srcq, widthq
   add src2q, widthq
   neg widthq
```

Width her pointer'a eklenir böylece her pointer artık işlenecek buffer'ın sonunu işaret eder. Width daha sonra negatif yapılır.

```assembly
    movu  m0, [srcq+widthq]
    movu  m1, [src2q+widthq]
```

Yüklemeler daha sonra widthq negatif olacak şekilde yapılır. Bu nedenle ilk iterasyonda [srcq+widthq] srcq'nun orijinal adresini işaret eder, yani buffer'ın başlangıcına geri işaret eder.

```assembly
    add   widthq, mmsize
    jl .loop
```

mmsize negatif widthq'ya eklenir ve onu sıfıra yaklaştırır. Döngü koşulu artık jl'dir (sıfırdan küçükse atla). Bu numara widthq'nun aynı anda pointer offset **ve** döngü sayacı olarak kullanılması anlamına gelir, cmp talimatından tasarruf sağlar. Ayrıca pointer offset'inin birden fazla yüklemede ve saklamada kullanılmasına ve gerektiğinde pointer offset'lerinin katlarının kullanılmasına olanak tanır (ödev için bunu hatırlayın).

**Hizalama**

Tüm örneklerde hizalama konusunu önlemek için movu kullanıyorduk. Birçok CPU, veri hizalanmışsa, yani bellek adresi SIMD register boyutuna bölünebiliyorsa veriyi daha hızlı yükleyip saklayabilir. Mümkün olduğunda FFmpeg'de mova kullanarak hizalanmış yükleme ve saklamaları kullanmaya çalışırız.

FFmpeg'de av_malloc heap'te hizalanmış bellek sağlayabilir ve DECLARE_ALIGNED C ön işlemci direktifi stack'te hizalanmış bellek sağlayabilir. mova hizalanmamış bir adresle kullanılırsa, segmentasyon hatası oluşturacak ve uygulama çökecektir. Ayrıca hizalama değerinin SIMD register boyutuna karşılık geldiğinden emin olmak önemlidir, yani xmm ile 16, ymm için 32 ve zmm için 64.

RODATA bölümünün başlangıcını 64-byte'a hizalamak için:

```assembly
SECTION_RODATA 64
```

Bunun sadece RODATA'nın başlangıcını hizaladığını unutmayın. Bir sonraki etiketin 64-byte sınırında kalmasını sağlamak için dolgu byte'ları gerekebilir.

**Aralık genişletmesi**

Şimdiye kadar kaçındığımız bir diğer konu da taşmadır. Bu, örneğin toplama veya çarpma gibi bir işlemden sonra bir byte'ın değeri 255'i geçtiğinde olur. Ara değerin bir byte'dan (örn. word'ler) daha büyük olmasına ihtiyaç duyabiliriz veya potansiyel olarak veriyi o daha büyük ara boyutta bırakmak isteyebiliriz.

İşaretsiz byte'lar için punpcklbw (packed unpack low bytes to words - paketlenmiş düşük byte'ları word'lere açma) ve punpckhbw (packed unpack high bytes to words - paketlenmiş yüksek byte'ları word'lere açma) devreye girer.

punpcklbw'nin nasıl çalıştığına bakalım. Intel Kılavuzundan SSE2 sürümü için sözdizimi şöyledir:

| PUNPCKLBW xmm1, xmm2/m128 |
| :---- |

Bu, kaynağının (sağ taraf) bir xmm register'ı veya bellek adresi (m128, standart [base + scale*index + disp] sözdizimi ile bir bellek adresi anlamına gelir) ve hedefin bir xmm register'ı olabileceği anlamına gelir.

Yukarıdaki officedaytime.com web sitesi neler olup bittiğini gösteren iyi bir diyagrama sahiptir:

![Bu nedir](image1.png)

Byte'ların her register'ın alt yarısından sırasıyla interleave edildiğini görebilirsiniz. Peki bunun aralık genişletmesi ile ne ilgisi var? Eğer src register'ı tamamen sıfırsa, bu dst'deki byte'ları sıfırlarla interleave eder. Bu, byte'lar işaretsiz olduğu için *sıfır genişletmesi* olarak bilinir. punpckhbw yüksek byte'lar için aynı şeyi yapmak için kullanılabilir.

Bunun nasıl yapıldığını gösteren bir parçacık:

```assembly
pxor      m2, m2 ; m2'yi sıfırla

movu      m0, [srcq]
movu      m1, m0 ; m0'ın bir kopyasını m1'de yap
punpcklbw m0, m2
punpckhbw m1, m2
```

```m0``` ve ```m1``` artık orijinal byte'ları word'lere sıfır genişletilmiş olarak içerir. Bir sonraki derste AVX'deki üç-operand talimatlarının ikinci movu'yu nasıl gereksiz kıldığını göreceksiniz.

**İşaret genişletmesi**

İşaretli veri biraz daha karmaşık. İşaretli bir integer'ı aralık genişletmek için [işaret genişletmesi](https://en.wikipedia.org/wiki/Sign_extension) olarak bilinen bir işlem kullanmamız gerekir. Bu MSB'leri işaret biti ile doldurur. Örneğin: int8_t'de -2, 0b11111110'dur. int16_t'ye işaret genişletmek için 1'in MSB'si 0b1111111111111110 yapmak için tekrarlanır.

```pcmpgtb``` (packed compare greater than byte - paketlenmiş byte'dan büyük karşılaştırma) işaret genişletmesi için kullanılabilir. (0 > byte) karşılaştırmasını yaparak, eğer byte negatifse hedef byte'daki tüm bitler 1'e ayarlanır, aksi takdirde hedef byte'daki bitler 0'a ayarlanır. punpckX yukarıdaki gibi işaret genişletmesi gerçekleştirmek için kullanılabilir. Eğer byte negatifse karşılık gelen byte 0b11111111'dir ve aksi takdirde 0x00000000'dır. Byte değerini pcmpgtb'nin çıktısı ile interleave etmek sonuç olarak word'e işaret genişletmesi gerçekleştirir.

```assembly
pxor      m2, m2 ; m2'yi sıfırla

movu      m0, [srcq]
movu      m1, m0 ; m0'ın bir kopyasını m1'de yap

pcmpgtb   m2, m0
punpcklbw m0, m2
punpckhbw m1, m2
```

Gördüğünüz gibi işaretsiz duruma kıyasla bir ekstra talimat var.

**Paketleme**

packuswb (pack unsigned word to byte - işaretsiz word'ü byte'a paketleme) ve packsswb word'den byte'a gitmenizi sağlar. Word içeren iki SIMD register'ını byte içeren bir SIMD register'ında interleave etmenizi sağlar. Değerler byte aralığını aşarsa, doyurulacaklarını (yani en büyük değerde kısıtlanacaklarını) unutmayın.

**Shuffle'lar**

Shuffle'lar, permute olarak da bilinirler, tartışmasız video işlemede en önemli talimattır ve SSSE3'te mevcut olan pshufb (packed shuffle bytes - paketlenmiş byte shuffle), en önemli varyantıdır.

Her byte için karşılık gelen kaynak byte hedef register'ının bir indeksi olarak kullanılır, MSB ayarlandığında hariç hedef byte sıfırlanır. Aşağıdaki C koduna benzerdir (SIMD'de tüm 16 döngü iterasyonu paralel olarak gerçekleşir):

```c
for(int i = 0; i < 16; i++) {
    if(src[i] & 0x80)
        dst[i] = 0;
    else
        dst[i] = dst[src[i]]
}
```
İşte basit bir assembly örneği:

```assembly
SECTION_DATA 64

shuffle_mask: db 4, 3, 1, 2, -1, 2, 3, 7, 5, 4, 3, 8, 12, 13, 15, -1

section .text

movu m0, [srcq]
movu m1, [shuffle_mask]
pshufb m0, m1 ; m1'e göre m0'ı shuffle et
```

Kolay okunabilirlik için çıktı byte'ını sıfırlamak için shuffle indeksi olarak -1 kullanıldığını unutmayın: byte olarak -1, 0b11111111 bit alanıdır (iki'nin tümleyeni), ve bu nedenle MSB (0x80) ayarlanmıştır.
