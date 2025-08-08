**FFmpeg Assembly Dili Birinci Ders**

**Giriş**

FFmpeg Assembly Dili Okuluna hoş geldiniz. Programlamanın en ilginç, zorlu ve ödüllendirici yolculuğunda ilk adımı attınız. Bu dersler, assembly dilinin FFmpeg'de nasıl yazıldığı konusunda size temel bilgi verecek ve bilgisayarınızda gerçekte neler olup bittiğine gözlerinizi açacaktır.

**Gerekli Bilgiler**

* C bilgisi, özellikle pointer'lar. Eğer C bilmiyorsanız, [The C Programming Language](https://en.wikipedia.org/wiki/The_C_Programming_Language) kitabını çalışın  
* Lise Matematiği (skalar vs vektör, toplama, çarpma vb.)

**Assembly dili nedir?**

Assembly dili, CPU'nun işlediği talimatlara doğrudan karşılık gelen kod yazdığınız bir programlama dilidir. İnsan tarafından okunabilir assembly dili, adından da anlaşılacağı gibi, CPU'nun anlayabileceği *makine kodu* olarak bilinen ikili veriye *derlenir*. Assembly dili kodunun kısaca "assembly" veya "asm" olarak anıldığını görebilirsiniz.

FFmpeg'deki assembly kodunun büyük çoğunluğu *SIMD, Single Instruction Multiple Data* (Tek Talimat Çoklu Veri) olarak bilinen türdedir. SIMD bazen vektör programlama olarak da anılır. Bu, belirli bir talimatın aynı anda birden fazla veri öğesi üzerinde çalıştığı anlamına gelir. Çoğu programlama dili bir seferde bir veri öğesi üzerinde çalışır, bu skalar programlama olarak bilinir.

Tahmin edebileceğiniz gibi, SIMD kendisini bellekte sıralı olarak düzenlenmiş çok miktarda veriye sahip görüntü, video ve ses işlemeye iyi uyarlar. CPU'da sıralı veriyi işlememize yardımcı olacak özel talimatlar mevcuttur.

FFmpeg'de "assembly fonksiyonu", "SIMD" ve "vektör(leştirme)" terimlerinin birbirinin yerine kullanıldığını göreceksiniz. Hepsi aynı şeyi ifade eder: Birden fazla veri öğesini tek seferde işlemek için elle assembly dilinde bir fonksiyon yazmak. Bazı projeler bunları "assembly kernel'leri" olarak da adlandırabilir.

Tüm bunlar karmaşık gelebilir, ancak FFmpeg'de lise öğrencilerinin assembly kodu yazdığını hatırlamak önemlidir. Her şeyde olduğu gibi, öğrenme %50 jargon ve %50 gerçek öğrenmedir.

**Neden assembly dilinde yazıyoruz?**  
Multimedya işlemeyi hızlı hale getirmek için. Assembly kodu yazmaktan 10 kat veya daha fazla hız iyileştirmesi elde etmek çok yaygındır, bu özellikle videoları takılma olmadan gerçek zamanlı oynatmak istediğinizde önemlidir. Ayrıca enerji tasarrufu sağlar ve pil ömrünü uzatır. Video kodlama ve kod çözme fonksiyonlarının hem son kullanıcılar hem de büyük şirketlerin veri merkezlerinde dünyadaki en yoğun kullanılan fonksiyonlardan bazıları olduğunu belirtmek gerekir. Bu nedenle küçük bir iyileştirme bile hızla büyük sonuçlar doğurur.

Çevrimiçi olarak, insanların daha hızlı geliştirme için assembly talimatlarına eşlenen C benzeri fonksiyonlar olan *intrinsic'ler* kullandığını sık sık görürsünüz. FFmpeg'de intrinsic kullanmayız, bunun yerine assembly kodunu elle yazarız. Bu tartışmalı bir alan olmakla birlikte, intrinsic'ler genellikle derleyiciye bağlı olarak elle yazılmış assembly'den yaklaşık %10-15 daha yavaştır (intrinsic destekçileri buna katılmaz). FFmpeg için her parça ekstra performans yardımcı olur, bu yüzden assembly kodunu doğrudan yazarız. Ayrıca intrinsic'lerin "[Hungarian Notation](https://en.wikipedia.org/wiki/Hungarian_notation)" kullanımları nedeniyle okunması zor olduğu argümanı da vardır.

Ayrıca FFmpeg'de tarihsel nedenlerle veya Linux Kernel gibi projelerde çok özel kullanım durumları nedeniyle birkaç yerde *inline assembly* (yani intrinsic kullanmayan) kalıntıları görebilirsiniz. Bu, assembly kodunun ayrı bir dosyada değil, C kodu ile satır içinde yazıldığı durumdur. FFmpeg gibi projelerdeki genel kanı, bu kodun okunması zor, derleyiciler tarafından yaygın olarak desteklenmeyen ve bakımı yapılamayan kod olduğudur.

Son olarak, çevrimiçi birçok sözde uzmanın bunların hiçbirinin gerekli olmadığını ve derleyicinin sizin için tüm bu "vektörleştirmeyi" yapabileceğini söylediğini göreceksiniz. En azından öğrenme amacıyla, onları görmezden gelin: örneğin [dav1d projesi](https://www.videolan.org/projects/dav1d.html)'ndeki son testler bu otomatik vektörleştirmeden yaklaşık 2 kat hızlanma gösterirken, elle yazılmış sürümler 8 kata ulaşabiliyordu.

**Assembly dili çeşitleri**  
Bu dersler x86 64-bit assembly diline odaklanacak. Bu amd64 olarak da bilinir, ancak Intel CPU'larında da çalışır. ARM ve RISC-V gibi diğer CPU'lar için başka assembly türleri vardır ve potansiyel olarak gelecekte bu dersler bunları da kapsayacak şekilde genişletilebilir.

Çevrimiçi göreceğiniz x86 assembly sözdiziminin iki çeşidi vardır: AT&T ve Intel. AT&T Sözdizimi daha eskidir ve Intel sözdizimime kıyasla okunması daha zordur. Bu nedenle Intel sözdizimini kullanacağız.

**Destekleyici materyaller**  
Kitapların veya Stack Overflow gibi çevrimiçi kaynakların referans olarak özellikle yardımcı olmadığını duymak sizi şaşırtabilir. Bu kısmen Intel sözdizimi ile elle yazılmış assembly kullanma seçimimizden kaynaklanır. Ayrıca birçok çevrimiçi kaynak, genellikle SIMD olmayan kod kullanarak işletim sistemi programlama veya donanım programlamaya odaklanır. FFmpeg assembly'si özellikle yüksek performanslı görüntü işlemeye odaklanır ve göreceğiniz gibi assembly programlamaya özellikle benzersiz bir yaklaşımdır. Bununla birlikte, bu dersleri tamamladıktan sonra diğer assembly kullanım durumlarını anlamak kolaydır.

Birçok kitap assembly öğretmeden önce çok sayıda bilgisayar mimarisi detayına girer. Öğrenmek istediğiniz bu ise bu iyidir, ancak bizim bakış açımızdan, araba kullanmayı öğrenmeden önce motorları incelemek gibidir.

Bununla birlikte, "The Art of 64-bit assembly" kitabının sonraki bölümlerinde SIMD talimatlarını ve davranışlarını görsel formda gösteren diyagramlar yardımcıdır: [https://artofasm.randallhyde.com/](https://artofasm.randallhyde.com/)

Soruları yanıtlamak için bir discord sunucusu mevcuttur:  
[https://discord.com/invite/Ks5MhUhqfB](https://discord.com/invite/Ks5MhUhqfB)

**Register'lar**  
Register'lar CPU'da verinin işlenebileceği alanlardır. CPU'lar doğrudan bellek üzerinde çalışmaz, bunun yerine veri register'lara yüklenir, işlenir ve belleğe geri yazılır. Assembly dilinde, genellikle, veriyi önce bir register'dan geçirmeden bir bellek konumundan diğerine doğrudan kopyalayamazsınız.

**Genel Amaçlı Register'lar**  
İlk register türü Genel Amaçlı Register (GPR) olarak bilinir. GPR'lar genel amaçlı olarak anılır çünkü bu durumda 64-bit'e kadar bir değer veya bir bellek adresi (bir pointer) içerebilirler. Bir GPR'daki değer toplama, çarpma, kaydırma vb. işlemlerle işlenebilir.

Çoğu assembly kitabında, GPR'ların inceliklerine, tarihsel geçmişe vb. adanmış tüm bölümler vardır. Bunun nedeni GPR'ların işletim sistemi programlama, tersine mühendislik vb. söz konusu olduğunda önemli olmalarıdır. FFmpeg'de yazılan assembly kodunda, GPR'lar daha çok iskelet gibidir ve çoğu zaman karmaşıklıkları gerekli değildir ve soyutlanmıştır.

**Vektör register'ları**  
Vektör (SIMD) register'ları, adından da anlaşılacağı gibi, birden fazla veri öğesi içerir. Çeşitli vektör register türleri vardır:

* mm register'ları - MMX register'ları, 64-bit boyutunda, tarihsel ve artık pek kullanılmıyor  
* xmm register'ları - XMM register'ları, 128-bit boyutunda, yaygın olarak mevcut  
* ymm register'ları - YMM register'ları, 256-bit boyutunda, bunları kullanırken bazı komplikasyonlar var   
* zmm register'ları - ZMM register'ları, 512-bit boyutunda, sınırlı erişilebilirlik

Video sıkıştırma ve açmadaki hesaplamaların çoğu integer tabanlıdır, bu nedenle buna sadık kalacağız. İşte bir xmm register'ında 16 byte örneği:

| a | b | c | d | e | f | g | h | i | j | k | l | m | n | o | p |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Ancak sekiz word (16-bit integer) olabilir

| a | b | c | d | e | f | g | h |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Veya dört double word (32-bit integer)

| a | b | c | d |
| :---- | :---- | :---- | :---- |

Veya iki quadword (64-bit integer):

| a | b |
| :---- | :---- |

Özetlemek gerekirse:

* **b**yte'lar - 8-bit veri  
* **w**ord'ler - 16-bit veri  
* **d**oubleword'ler - 32-bit veri  
* **q**uadword'ler - 64-bit veri  
* **d**ouble **q**uadword'ler - 128-bit veri

Kalın harfler daha sonra önemli olacak.

**x86inc.asm include**  
Birçok örnekte x86inc.asm dosyasını dahil ettiğimizi göreceksiniz. X86inc.asm, FFmpeg, x264 ve dav1d'de bir assembly programcısının hayatını kolaylaştırmak için kullanılan hafif bir soyutlama katmanıdır. Birçok şekilde yardımcı olur, ancak başlangıç için faydalı yaptığı şeylerden biri GPR'ları r0, r1, r2 olarak etiketlemesidir. Bu, herhangi bir register adını hatırlamanız gerekmediği anlamına gelir. Daha önce bahsedildiği gibi, GPR'lar genellikle sadece iskelet olduğu için bu hayatı çok kolaylaştırır.

**Basit bir skalar asm parçacığı**

Neler olup bittiğini görmek için basit (ve oldukça yapay) bir skalar asm parçacığına (her talimat içinde aynı anda bireysel veri öğeleri üzerinde çalışan assembly kodu) bakalım:

```assembly
mov  r0q, 3  
inc  r0q  
dec  r0q  
imul r0q, 5
```

İlk satırda, *immediate değer* 3 (bellekten getirilen bir değerin aksine doğrudan assembly kodunun kendisinde saklanan bir değer) r0 register'ına quadword olarak saklanıyor. Intel sözdiziminde, kaynak operand (veriyi sağlayan değer veya konum, sağda yer alır) hedef operand'a (veriyi alan konum, solda yer alır) aktarılır, tıpkı memcpy davranışı gibi. Bunu "r0q = 3" olarak da okuyabilirsiniz, çünkü sıra aynıdır. r0'ın "q" eki register'ın quadword olarak kullanıldığını belirtir. inc değeri artırır böylece r0q 4 içerir, dec değeri 3'e geri azaltır. imul değeri 5 ile çarpar. Sonunda, r0q 15 içerir.

mov ve inc gibi makine koduna derleyici tarafından derlenen insan tarafından okunabilir talimatların *mnemonik* olarak bilindiğini unutmayın. Çevrimiçi ve kitaplarda MOV ve INC gibi büyük harflerle temsil edilen mnemonikleri görebilirsiniz ancak bunlar küçük harf sürümleri ile aynıdır. FFmpeg'de küçük harf mnemonikleri kullanırız ve büyük harfleri makrolar için saklı tutarız.

**Temel bir vektör fonksiyonunu anlama**

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

Satır satır gözden geçirelim:

```assembly
%include "x86inc.asm"
```

Bu, assembly yazmayı basitleştirmek için yardımcılar, önceden tanımlanmış isimler ve makrolar (aşağıdaki cglobal gibi) sağlamak amacıyla x264, FFmpeg ve dav1d topluluklarında geliştirilen bir "header"dır.

```assembly
SECTION .text
```

Bu, çalıştırmak istediğiniz kodun yerleştirildiği bölümü belirtir. Bu, sabit veri koyabileceğiniz .data bölümünün aksine.

```assembly
;static void add_values(uint8_t *src, const uint8_t *src2)  
INIT_XMM sse2
```

İlk satır fonksiyon argümanının C'de nasıl göründüğünü gösteren bir yorumdur (assembly'de noktalı virgül ";" C'deki "//" gibidir). İkinci satır, paddb bir sse2 talimatı olduğu için sse2 talimat setini kullanarak XMM register'larını nasıl başlattığımızı gösterir. sse2'yi bir sonraki derste daha detaylı ele alacağız.

```assembly
cglobal add_values, 2, 2, 2, src, src2
```

Bu, "add_values" adlı bir C fonksiyonu tanımladığı için önemli bir satırdır.

Her öğeyi tek tek gözden geçirelim:

* Sonraki parametre iki fonksiyon argümanına sahip olduğunu gösterir.   
* Bundan sonraki parametre argümanlar için iki GPR kullanacağımızı gösterir. Bazı durumlarda daha fazla GPR kullanmak isteyebiliriz, bu nedenle x86util'e daha fazlasına ihtiyacımız olduğunu söylemek zorundayız.   
* Bundan sonraki parametre x86util'e kaç XMM register'ı kullanacağımızı söyler.  
* Takip eden iki parametre fonksiyon argümanları için etiketlerdir.

Eski kodların fonksiyon argümanları için etiket bulundurmayabileceğini, bunun yerine r0, r1 vb. kullanarak GPR'lara doğrudan hitap edebileceğini belirtmek gerekir.

```assembly
    movu  m0, [srcq]  
    movu  m1, [src2q]
```

movu, movdqu'nun (move double quad unaligned - hizalanmamış çift quad taşıma) kısaltmasıdır. Hizalama başka bir derste ele alınacak ancak şimdilik movu, [srcq]'den 128-bit taşıma olarak değerlendirilebilir. mov durumunda, köşeli parantezler [srcq]'deki adresin referans alındığı anlamına gelir, C'de *src'nin eşdeğeridir. Bu *yükleme* olarak bilinir. "q" ekinin pointer'ın boyutunu ifade ettiğini (yani C'de sizeof(*src) == 64-bit sistemlerde 8 ve x86asm 32-bit sistemlerde 32-bit kullanacak kadar akıllı) ancak temel yüklemenin 128-bit olduğunu unutmayın.

Vektör register'larını bu durumda xmm0 olan tam adlarıyla değil, soyutlanmış bir form olan m0 olarak andığımızı unutmayın. Gelecek derslerde bunun kod yazmayı bir kez yapıp birden fazla SIMD register boyutunda çalışmasını nasıl sağladığını göreceksiniz.

```assembly
paddb m0, m1
```

paddb (bunu kafanızda *p-add-b* olarak okuyun) aşağıda gösterildiği gibi her register'daki her byte'ı topluyor. "p" ön eki "packed" (paketlenmiş) anlamına gelir ve vektör talimatları ile skalar talimatları tanımlamak için kullanılır. "b" eki bunun byte'lık toplama (byte toplamı) olduğunu gösterir.

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

Bu *saklama* olarak bilinir. Veri srcq pointer'ındaki adrese geri yazılır.

```assembly
RET
```

Bu, fonksiyonun döndüğünü belirten bir makrodur. FFmpeg'deki assembly fonksiyonlarının neredeyse tamamı bir değer döndürmek yerine argümanlardaki veriyi değiştirir.

Ödevde göreceğiniz gibi, assembly fonksiyonlarına fonksiyon pointer'ları oluşturuyoruz ve mevcut olduklarında bunları kullanıyoruz.

[Sonraki Ders](../lesson_02/index.md)
