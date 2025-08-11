**FFmpeg Assembly Dili Birinci Ders**

**Giriş**

FFMPEG Assembly Dil Okulu'na hoş geldiniz. Programlamadaki en ilginç, zorlayıcı ve ödüllendirici yolculuğa ilk adımı attınız. Bu dersler, FFmpeg'de Assembly dilinin nasıl yazıldığına dair temel bilgiler verecek ve bilgisayarınızda gerçekte neler olup bittiğini anlamanıza yardımcı olacaktır.

**Gerekli Bilgiler**

* C dili bilgisi, özellikle işaretçiler. C dilini bilmiyorsanız [The C Programming Language](https://en.wikipedia.org/wiki/The_C_Programming_Language) kitabı ile çalışın.
* Lise Matematiği (skaler ve vektörel, toplama, çarpma vb.)

**Assembly dili nedir?**

Assembly dili, bir CPU'nun işlediği komutlara doğrudan karşılık gelen kod yazdığınız bir programlama dilidir. İnsan tarafından okunabilir assembly dili -adından da anlaşılacağı gibi- CPU'nun anlayabileceği *makine kodu* olarak bilinen ikili verilere *dönüştürülür* (assemble edilir). Assembly dilinin kısaca “assembly” veya “asm” olarak adlandırıldığını görebilirsiniz.

FFMPEG'de bulunan Assembly kodlarının büyük bir kısmı *SIMD, Single Instruction Multiple Data* (Tek Talimat Çoklu Veri) olarak bilinir.
SIMD bazen vektör programlama olarak da adlandırılır. Bu, belirli bir komutun aynı anda birden fazla veri öğesi üzerinde işlem yaptığı anlamına gelir. Skaler programlama olarak bilinen çoğu programlama dili bir seferde tek bir veri öğesi üzerinde işlem yapar.

Anlayacağınız SIMD, bellekte sıralı olarak düzenlenmiş çok sayıda veri içeren görüntü, video ve ses dosyalarının işlenmesine çok uygundur. CPU'da sıralı verileri işlememize yardımcı olan özel komutlar bulunmaktadır.

FFmpeg'de, “assembly fonksiyonu”, ‘SIMD’ ve “vektör(leştirme)” terimleri birbirinin yerine kullanılmaktadır. Bunların hepsi birden fazla veri öğesini tek seferde işlemek için Assembly dilinde elle bir fonksiyon yazmayı ifade eder. Bazı projeler bunları “Assembly çekirdekleri” olarak da adlandırabilir.

Tüm bunlar kulağa karmaşık gelebilir, ancak FFmpeg'de lise öğrencilerinin Assembly kodu yazdığını unutmamak önemlidir. Her şeyde olduğu gibi, öğrenmenin %50'si jargon, %50'si ise gerçek öğrenmedir.

**Neden Assembly dilinde yazıyoruz?**  
Multimedya işlemeyi hızlandırmak için. Assembly kodu yazarak 10 kat veya daha fazla hız artışı elde etmek çok yaygındır ve bu, videoları gerçek zamanlı olarak takılmadan oynatmak istediğinizde özellikle önemlidir. Ayrıca enerji tasarrufu sağlar ve pil ömrünü uzatır. Video kodlama ve kod çözme işlevlerinin hem son kullanıcılar hem de veri merkezlerindeki büyük şirketler tarafından en yoğun kullanılan işlevlerden bazıları olduğunu belirtmek gerekir. Bu nedenle, küçük bir iyileştirme bile hızlı bir şekilde büyük bir fark yaratır.

Çevrimiçi ortamda, insanların daha hızlı geliştirme sağlamak için Assembly komutlarına eşlenen C benzeri işlevler olan *intrinsics* kullandığını sıklıkla görebilirsiniz. FFmpeg'de intrinsics kullanmıyoruz, bunun yerine Assembly kodunu elle yazıyoruz. Bu tartışmalı bir konudur, ancak intrinsics genellikle derleyiciye bağlı olarak elle yazılmış Assembly kodundan yaklaşık %10-15 daha yavaştır ancak intrinsics destekçileri buna katılmayacaktır. FFmpeg için ekstra performansın her bir parçası önemlidir, bu yüzden doğrudan Assembly kodu yazıyoruz. Intrinsics'in “[Macar Notasyonu (İngilizce)](https://en.wikipedia.org/wiki/Hungarian_notation)” kullanımı nedeniyle okunması zor olduğu yönünde bir tartışma da vardır.

Geçmiş nedenlerden dolayı FFmpeg'de veya Linux çekirdeği gibi projelerdeki çok özel kullanım durumları nedeniyle birkaç yerde *inline assembly*, yani intrinsics kullanmayan yöntem görebilirsiniz. Burada Assembly kodu ayrı bir dosyada değil, C kodu ile birlikte inline olarak yazılmıştır. FFmpeg gibi projelerde hakim görüş, bu kodun okunmasının zor olduğu, derleyiciler tarafından yaygın olarak desteklenmediği ve bakımı zor olduğu yönündedir.

Son olarak, internette birçok kendini uzman ilan eden kişinin, bunların hiçbirinin gerekli olmadığını ve derleyicinin tüm bu “vektörleştirme” işlemlerini sizin için yapabileceğini söylediğini göreceksiniz. En azından öğrenme amacıyla, bunları görmezden gelin: örneğin [dav1d projesi](https://www.videolan.org/projects/dav1d.html) gibi son testler, bu otomatik vektörleştirme ile yaklaşık 2 kat hız artışı sağlarken, elle yazılmış versiyonlar 8 kata kadar ulaşabildiğini gösterdi.

**Assembly dilinin özellikleri**  

Bu dersler x86 64-bit Assembly dilini baz alacaktır. Bu ayırca amd64 olarak da bilinir fakat Intel CPU'lar ile de çalışmaktadır. Assembly'nin ARM ve RISC-V gibi farklı CPU'lar için olan türü de vardır ve muhtemelen önümüzdeki dersler bunları da kapsayacaktır.

İnternette AT&T ve Intel olarak iki tür x86 Assembly sözdizimi göreceksiniz. AT&T sözdizimi Intel sözdizimine kıyasla daha eskidir ve okunması daha zordur. Bu nedenle Intel sözdizimini kullanacağız.

**Destekleyici materyaller**  
Stack Overflow gibi kitapların veya çevrimiçi kaynakların referans olarak pek yardımcı olmadığını duyunca şaşırabilirsiniz. Bunun nedeni nispeten Intel sözdizimini kullanan el yazısı derlemeyi tercih etmemizdir. Ancak bunun yanı sıra, birçok çevrimiçi kaynak işletim sistemi programlama veya donanım programlamaya odaklanmaktadır ve genellikle SIMD olmayan kod kullanmaktadır. FFmpeg Assembly özellikle yüksek performanslı görüntü işlemeye odaklanmıştır ve göreceğiniz gibi, Assembly programlamaya özellikle benzersiz bir yaklaşımdır. Bununla birlikte, bu dersleri tamamladıktan sonra diğer Assembly kullanım örneklerini anlamak kolaydır.

Birçok kitap, Assembly öğretmeden önce bilgisayar mimarisiyle ilgili birçok ayrıntıya girer. Öğrenmek istediğiniz şey buysa sorun yok ve bizim açımızdan bu araba kullanmayı öğrenmeden önce motorları incelemek gibidir.

Bununla birlikte “The Art of 64-bit Assembly” kitabının sonraki bölümlerinde SIMD komutlarını ve davranışlarını görsel bir biçimde gösteren diyagramlar yararlı olabilir: [https://artofasm.randallhyde.com/](https://artofasm.randallhyde.com/)

Soruları yanıtlamak için bir Discord sunucusu mevcuttur:  
[https://discord.com/invite/Ks5MhUhqfB](https://discord.com/invite/Ks5MhUhqfB)

**Yazmaçlar**  
Yazmaçlar, CPU'da verilerin işlenebildiği alanlardır. CPU'lar bellek üzerinde doğrudan işlem yapmazlar, bunun yerine veriler yazmaçlara yüklenir, işlenir ve belleğe geri yazılır. Assembly dilinde genellikle verileri bir yazmaç üzerinden geçirmeden bir bellek konumundan başka bir bellek konumuna doğrudan kopyalayamazsınız.

**Genel Amaçlı Yazmaçlar**  
İlk yazmaç türü, Genel Amaçlı Yazmaç (GPR) olarak bilinir. GPR'lar genel amaçlı olarak adlandırılır çünkü hem veri (bu durumda 64-bit'e kadar bir değer) hem de bir bellek adresi (işaretçi) içerebilirler. Bir GPR'deki değer, toplama, çarpma, kaydırma vb. işlemlerle işlenebilir.

Çoğu Assembly kitabında, GPR'lerin inceliklerine, tarihsel geçmişine vb. ayrılmış bütün bölümler vardır. Bunun nedeni, GPR'lerin işletim sistemi programlama, tersine mühendislik vb. konularda önemli olmasıdır. FFmpeg'de yazılan Assembly kodunda GPR'ler daha çok bir iskelet görevi görür, çoğu zaman karmaşıklıklarına ihtiyaç duyulmaz ve soyutlanarak ortadan kaldırılır.

**Vektör yazmaçları**  
Vektör (SIMD) yazmaçları, adından da anlaşılacağı gibi, birden fazla veri öğesi içerir. Çeşitli türde vektör yazmaçları vardır:

* mm yazmaçları - MMX yazmaçları, 64 bit boyutunda, eski ve artık pek kullanılmayan
* xmm yazmaçları - XMM yazmaçları, 128 bit boyutunda, yaygın olarak kullanılabilir
* ymm yazmaçları - YMM yazmaçları, 256 bit boyutunda, kullanırken bazı zorluklar olabilir   
* zmm yazmaçları - ZMM yazmaçları, 512 bit boyutunda, sınırlı kullanılabilirlik

Video sıkıştırma ve açma işlemlerindeki hesaplamaların çoğu tamsayı tabanlıdır, bu yüzden biz de buna uyacağız. İşte xmm yazmacındaki 16 baytlık bir örnek:

| a | b | c | d | e | f | g | h | i | j | k | l | m | n | o | p |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Fakat 8 kelime de olabilir (16 bitlik tamsayılar)

| a | b | c | d | e | f | g | h |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

ya da 4 çift kelime (doublewords) (32 bitlik tamsayılar)

| a | b | c | d |
| :---- | :---- | :---- | :---- |

Veya iki quadwords (64 bitlik tamsayılar):

| a | b |
| :---- | :---- |

Özetle:


* **b**ytes - 8 bitlik veri  
* **w**ords - 16 bitlik veri  
* **d**oublewords - 32 bitlik veri  
* **q**uadwords - 64 bitlik veri  
* **d**ouble **q**uadwords - 128 bitlik veri

Kalın yazılmış karakterler daha sonra önemli olacaktır.

**x86inc.asm dahil**  
Birçok örnekte x86inc.asm dosyasını dahil ettiğimizi göreceksiniz. X86inc.asm, FFmpeg, x264 ve dav1d'de birleştirme programcılarının işini kolaylaştırmak için kullanılan hafif bir soyutlama katmanıdır. Birçok yönden yardımcı olur ancak başta yaptığı yararlı şeylerden biri GPR'lari, r0, r1, r2 olarak etiketlemektir. Bu herhangi bir kayıt adını hatırlamanız gerekmediği anlamına gelir. Daha önce de belirtildiği gibi, GPR'ler genellikle sadece iskelet görevi görür, bu da işi çok daha kolaylaştırır.
**Basit bir skaler asm parçası**

Neler olup bittiğini görmek için basit ve oldukça yapay bir skaler asm (her komutta tek tek veri öğeleri üzerinde işlem yapan assembly kodu) parçasına bakalım:

```assembly
mov  r0q, 3  
inc  r0q  
dec  r0q  
imul r0q, 5
```

İlk satırda, *anlık değer* 3 (bellekten alınmak yerine doğrudan assembly kodunun içinde saklanan bir değer), r0 yazmacına bir quadword olarak depolanır. Intel sözdiziminde, kaynak işlenen (sağda bulunan, veriyi sağlayan değer veya konum), hedef işlenene (solda bulunan, veriyi alan konum) tıpkı memcpy'nin davranışına benzer şekilde aktarılır. Sıra aynı olduğu için bunu “r0q = 3” olarak da okuyabilirsiniz. r0'ın “q” soneki, yazmacın bir quadword olarak kullanıldığını belirtir. inc değeri artırır, böylece r0q 4 içerir, dec değeri tekrar 3'e düşürür. imul değeri 5 ile çarpar. Yani sonunda r0q 15 içerir.

mov ve inc gibi insan tarafından okunabilir komutların, assembler tarafından makine koduna dönüştürülen hallerinin *mnemonics* olarak bilindiğini unutmayın. Çevrimiçi kaynaklarda ve kitaplarda MOV ve INC gibi büyük harflerle temsil edilen mnemonics görebilirsiniz, ancak bunlar küçük harfli versiyonlarla aynıdır. FFmpeg'de küçük harfli mnemonics kullanırız ve büyük harfleri makrolar için ayırırız.

**Basit bir vektör fonksiyonunu anlayalım**

İşte ilk SIMD fonksiyonumuz:

```assembly
%include "x86inc.asm"

SECTION .text

;static void add_values(uint8_t *src, const uint8_t *src2)  
INIT_XMM sse2  
cglobal add_values, 2, 2, 2, src, src2   
    movu  m0, [srcq]  
    movu  m1, [src2q]

    paddb m0, m1

    movu  [srcq], m0

    RET
```

Satır satır üzerinden geçelim

```assembly
%include "x86inc.asm"
```

Bu, x264, FFmpeg ve dav1d topluluklarında geliştirilen bir “başlıktır" ve derleme yazımını basitleştirmek için yardımcılar, önceden tanımlanmış isimler ve makrolar (aşağıdaki cglobal gibi) sağlar.

```assembly
SECTION .text
```

Bu, yürütmek istediğiniz kodun yerleştirildiği bölümü belirtir. Bu, sabit verileri yerleştirebileceğiniz .data bölümünün aksine bir durumdur.

```assembly
;static void add_values(uint8_t *src, const uint8_t *src2)  
INIT_XMM sse2
```

İlk satır, fonksiyon argümanının C dilinde nasıl göründüğünü gösteren bir yorumdur (asm'deki noktalı virgül “;” C dilindeki “//” gibidir). İkinci satır, sse2 komut setini kullanarak fonksiyonu XMM kayıtlarını kullanacak şekilde nasıl başlattığımızı gösterir. Bunun nedeni, paddb'nin bir sse2 komutu olmasıdır. Bir sonraki derste sse2'yi daha ayrıntılı olarak ele alacağız.

```assembly
cglobal add_values, 2, 2, 2, src, src2
```

Bu, “add_values” adlı bir C işlevini tanımladığı için önemli bir satırdır. 

Her bir öğeyi tek tek inceleyelim:

* Bir sonraki parametre, iki işlev argümanı olduğunu gösterir. 
* Bundan sonraki parametre, argümanlar için iki GPR kullanacağımızı gösterir. Bazı durumlarda daha fazla GPR kullanmak isteyebiliriz, bu nedenle x86util'e daha fazlasına ihtiyacımız olduğunu belirtmeliyiz.   
* Bundan sonraki parametre, x86util'e kaç tane XMM kaydı kullanacağımızı bildirir.
* Sonraki iki parametre, işlev argümanları için etiketlerdir.

Eski kodlarda işlev argümanları için etiketler olmayabilir, bunun yerine r0, r1 vb. kullanılarak GPR'lere doğrudan adres verilebilir.

```assembly
    movu  m0, [srcq]  
    movu  m1, [src2q]
```

movu, movdqu (move double quad unaligned) komutunun kısaltmasıdır. Hizalama konusu başka bir derste ele alınacaktır, ancak şimdilik movu, [srcq]'dan 128 bitlik bir taşıma olarak düşünülebilir. mov komutunda, köşeli parantezler [srcq]'daki adresin dereferanslandığını, yani C dilinde **src ile eşdeğer olduğunu gösterir.* Bu, yükleme olarak bilinir. “q” sonekinin işaretçinin boyutunu ifade ettiğini unutmayın *(*yani C'de 64 bit sistemlerde *sizeof(*src) == 8 anlamına gelir ve x86asm, 32 bit sistemlerde 32 bit kullanacak kadar akıllıdır), ancak temel yükleme 128 bittir.

Vektör yazmaçlarına tam adlarıyla, bu durumda xmm0 ile, değil m0 gibi soyut bir biçimle atıfta bulunmadığımızı unutmayın. İleriki derslerde, bunun bir kez kod yazıp birden fazla SIMD kayıt boyutunda çalışmasını nasıl sağladığını göreceksiniz.

```assembly
paddb m0, m1
```

paddb (bunu kafanızda *p-add-b* olarak okuyun) aşağıda gösterildiği gibi her bir kayıtta her bir baytı ekler. “p” öneki ‘paketlenmiş’ anlamına gelir ve vektör komutlarını skaler komutlardan ayırmak için kullanılır. “b” soneki ise bunun bayt bazında toplama (baytların toplamı) olduğunu gösterir.

| a | b | c | d | e | f | g | h | i | j | k | l | m | n | o | p |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

\+

| q | r | s | t | u | v | w | x | y | z | aa | ab | ac | ad | ae | af |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

\=

| a+q | b+r | c+s | d+t | e+u | f+v | g+w | h+x | i+y | j+z | k+aa | l+ab | m+ac | n+ad | o+ae | p+af |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

```assembly
movu  [srcq], m0
```

Bu, depo olarak bilinen şeydir. Veriler srcq işaretçisindeki adrese geri yazılır.

```assembly
RET
```

Bu, fonksiyonun döndürdüğü değeri belirtmek için kullanılan bir makrodur. FFmpeg'deki neredeyse tüm Assembly fonksiyonları, bir değer döndürmek yerine argümanlardaki verileri değiştirir.

Ödevde göreceğiniz gibi Assembly fonksiyonlarına fonksiyon işaretçileri oluşturuyor ve bunları kullanılabilir oldukları yerlerde kullanıyoruz.

[Sonraki Ders](../lesson_02/index.tr.md)
