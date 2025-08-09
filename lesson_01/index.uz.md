**FFmpeg Assembly tili: Birinchi dars**

**Kirish**

FFmpeg Assembly Tili Maktabiga xush kelibsiz. Siz dasturlashdagi eng qiziqarli, murakkab va eng mukofotli sayohatga ilk qadamni qo‘ydingiz. Ushbu darslar FFmpeg’da assembly tili qanday yozilishini tushunishga asos yaratadi va kompyuteringizda aslida nimalar sodir bo‘layotganini ko‘rishga yordam beradi.

**Zarur bilimlar**

* C tilini, xususan ko‘rsatkichlarni (pointer) bilish. Agar C ni bilmasangiz, [The C Programming Language](https://en.wikipedia.org/wiki/The_C_Programming_Language) kitobi bilan ishlang  
* Maktab matematikasi (skalyar va vektor, qo‘shish, ko‘paytirish va h.k.)

**Assembly tili nima?**

Assembly tili — bu CPU tomonidan bajariladigan ko‘rsatmalarga to‘g‘ridan-to‘g‘ri mos keladigan kod yoziladigan dasturlash tilidir. Nomidan ko‘rinib turibdiki, inson o‘qiy oladigan assembly kodi *mashina kodi* deb ataladigan ikkilik (binary) ma’lumotga *assemble qilinadi* — ya’ni, CPU tushunadigan formatga aylantiriladi. Assembly kodiga qisqacha “assembly” yoki “asm” deb ham murojaat qilinadi.

FFmpeg’dagi assembly kodlarning katta qismi *SIMD, Single Instruction Multiple Data* deb ataladi. SIMD ba’zan vektor dasturlash deb ham yuritiladi. Bu shuni anglatadiki, ma’lum bir ko‘rsatma bir vaqtning o‘zida bir nechta ma’lumot elementlari ustida ishlaydi. Aksariyat dasturlash tillari esa bir vaqtning o‘zida faqat bitta ma’lumot elementi ustida ishlaydi, bu skalyar dasturlash deb ataladi.

Taxmin qilganingizdek, SIMD xotirada ketma-ket joylashgan juda ko‘p ma’lumotlarga ega bo‘lgan tasvir, video va audio kabi ma’lumotlarni qayta ishlash uchun juda qulay. Ketma-ket ma’lumotlarni qayta ishlashga yordam beradigan maxsus ko‘rsatmalar CPU’da mavjud.

FFmpeg’da “assembly function”, “SIMD” va “vector(ise)” atamalari bir-birining o‘rnida qo‘llanadi. Ularning barchasi bir narsani anglatadi: qo‘lda assembly tilida yozilgan, bir martada bir nechta ma’lumot elementlarini qayta ishlaydigan funksiya. Ba’zi loyihalarda bular “assembly kernels” deb ham ataladi. 

Bularning barchasi qiyin eshitilishi mumkin, lekin FFmpeg’da hatto maktab o‘quvchilari ham assembly kod yozishganini yodda tutish muhim. Har qanday narsada bo‘lgani kabi, o‘rganishning 50% i jargon, qolgan 50% i esa haqiqiy o‘rganishdan iborat.

**Nega assembly tilida yozamiz?**  
Multimedia qayta ishlashni tez qilish uchun. Assembly kod yozish orqali 10 martagacha va undan ko‘p tezlanish olish odatiy hol bo‘lib, bu videoni to‘xtalishsiz real vaqtda ijro etish uchun ayniqsa muhim. Bu energiyani tejaydi va batareya umrini uzaytiradi. Shuni ham aytib o‘tish joizki, video kodlash va dekodlash funksiyalari yer yuzidagi eng ko‘p ishlatiladigan funksiyalardan bo‘lib, ularni oddiy foydalanuvchilar ham, katta kompaniyalar ham o‘z ma’lumot markazlarida juda ko‘p ishlatadilar. Shunday ekan, hatto kichik yaxshilanish ham tezda katta samara beradi.

Onlayn maydonlarda ko‘pincha odamlar *intrinsics*dan foydalanishini ko‘rasiz — bular assembly ko‘rsatmalariga mos tushadigan va tezroq ishlab chiqish imkonini beradigan C-ga o‘xshash funksiyalardir. FFmpeg’da biz intrinsics’dan foydalanmaymiz, uning o‘rniga assembly kodini qo‘lda yozamiz. Bu bahsli mavzu, ammo kompilyatorga bog‘liq holda, intrinsics odatda qo‘lda yozilgan assemblydan 10–15% sekinroq bo‘ladi (intrinsics tarafdorlari bunga qo‘shilmasligi mumkin). FFmpeg uchun har bir qo‘shimcha unumdorlik muhim, shuning uchun biz bevosita assemblyda yozamiz. Yana bir dalil shuki, intrinsics ko‘pincha “[Hungarian Notation](https://en.wikipedia.org/wiki/Hungarian_notation)”dan foydalanishi sababli o‘qishga qiyin.

Shuningdek, tarixiy sabablarga ko‘ra FFmpeg’da kam sonli joylarda yoki Linux yadrosi kabi loyihalarda aniq foydalanish holatlari tufayli *inline assembly* (ya’ni intrinsics ishlatmasdan) uchrashi mumkin. Bunda assembly kod alohida faylda emas, balki C kodi orasida yoziladi. FFmpeg kabi loyihalarda bunday kodni o‘qish qiyin, kompilyatorlar keng qo‘llab-quvvatlamaydi va unga xizmat ko‘rsatish mushkul degan qarash ustun.

Nihoyat, onlaynda o‘zini ekspert deb biladigan ko‘plab odamlar bularning hech biri kerak emas, kompilator bularning hammasini siz uchun “vektorlashtirib” beradi, deyishadi. Hech bo‘lmaganda o‘rganish maqsadida ularga e’tibor bermang: yaqindagi, masalan, [dav1d loyihasi](https://www.videolan.org/projects/dav1d.html) dagi testlar avtomatik vektorlashtirish taxminan 2 baravar tezlanish berishini, qo‘lda yozilgan versiyalar esa 8 baravargacha yetishini ko‘rsatdi.

**Assembly tilining turlari**  
Bu darslar x86 64-bit assembly tiliga e’tibor qaratadi. Bu amd64 deb ham yuritiladi, biroq Intel CPU’larida ham ishlaydi. ARM va RISC-V kabi boshqa CPU’lar uchun boshqa turdagi assemblylar ham mavjud va kelajakda bu darslar ularni ham qamrab olishi mumkin.

x86 assembly sintaksisining onlaynda uchraydigan ikki turi bor: AT&T va Intel. AT&T sintaksisi eskiroq va Intel sintaksisiga qaraganda o‘qish qiyinroq. Shuning uchun biz Intel sintaksisidan foydalanamiz.

**Qo‘llab-quvvatlovchi materiallar**  
Sizni ajablantirishi mumkin, lekin kitoblar yoki Stack Overflow kabi onlayn manbalar manba sifatida unchalik ham foydali emas. Buning sabablari orasida bizning qo‘lda yozilgan assembly va Intel sintaksisini tanlaganimiz ham bor. Bundan tashqari, ko‘plab onlayn manbalar operatsion tizim yoki apparat darajasidagi dasturlashga qaratilgan bo‘lib, odatda SIMD bo‘lmagan koddan foydalanadi. FFmpeg assembly alohida tarzda yuqori unumli tasvirni qayta ishlashga yo‘naltirilgan va ko‘rasizki, assembly dasturlashga o‘ziga xos yondashuvga ega. Shunday bo‘lsa-da, bu darslarni tugatganingizdan so‘ng boshqa assembly ishlatish holatlarini tushunish osonlashadi

Ko‘plab kitoblar assembly o‘rgatishdan oldin kompyuter arxitekturasi tafsilotlariga chuqur kirib boradi. Agar siz aynan shuni o‘rganmoqchi bo‘lsangiz, bu yomon emas, ammo bizning nuqtai nazarimizdan bu mashinani haydashni o‘rganishdan oldin dvigatellarni o‘rganishga o‘xshaydi.

Shunga qaramay, “The Art of 64-bit assembly” kitobining keyingi qismlaridagi SIMD ko‘rsatmalarini va ularning xatti-harakatini vizual shaklda ko‘rsatadigan diagrammalar foydali: [https://artofasm.randallhyde.com/](https://artofasm.randallhyde.com/)

Savollarga javob berish uchun Discord serveri mavjud:  
[https://discord.com/invite/Ks5MhUhqfB](https://discord.com/invite/Ks5MhUhqfB)

**Registrlar**  
Registrlar — bu CPU ichida ma’lumotlar qayta ishlanadigan sohalar. CPU’lar to‘g‘ridan-to‘g‘ri xotira ustida ishlamaydi; buning o‘rniga ma’lumotlar registrlarga yuklanadi, qayta ishlanadi va yana xotiraga yoziladi. Assembly tilida, odatda, ma’lumotni bir xotira manzilidan boshqasiga registr orqali o‘tkazmasdan bevosita nusxalab bo‘lmaydi.

**Umumiy maqsadli registrlar**  
Birinchi turdagi registrlar Umumiy Maqsadli Registrlar (GPR) deb ataladi. GPR’lar umumiy maqsadli deyilishining sababi — ular yoki ma’lumotni (bu yerda 64-bitgacha bo‘lgan qiymat), yoki xotira manzilini (ko‘rsatkichni) saqlashi mumkin. GPR’dagi qiymat qo‘shish, ko‘paytirish, siljitish va h.k. kabi amallar orqali qayta ishlanishi mumkin. 

Ko‘p assembly kitoblarida GPR’lardagi nozik jihatlar, tarixiy fon va hokazolarga bag‘ishlangan butun boblar bo‘ladi. Buning sababi — GPR’lar operatsion tizim dasturlashida, teskari muhandislikda va shu kabilarda muhim. FFmpeg’da yozilgan assembly kodlarda esa GPR’lar ko‘proq “qo‘shimcha karkas” vazifasini bajaradi va ko‘pincha ularning murakkabliklari kerak bo‘lmaydi hamda abstraksiyalanadi. 

**Vektor registrlari**  
Vektor (SIMD) registrlari, nomidan ko‘rinib turibdiki, bir nechta ma’lumot elementlarini o‘z ichiga oladi. Vektor registrlarining turli turlari mavjud:

* mm registrlar — MMX registrlari, 64-bit o‘lchamda, tarixiy va endi deyarli ishlatilmaydi  
* xmm registrlar — XMM registrlari, 128-bit o‘lchamda, keng tarqalgan  
* ymm registrlar — YMM registrlari, 256-bit o‘lchamda, ulardan foydalanishda ba’zi murakkabliklar bor   
* zmm registrlar — ZMM registrlari, 512-bit o‘lchamda, cheklangan darajada mavjud

Video siqish va siqilganini ochishdagi hisob-kitoblarning ko‘pi butun sonlarda amalga oshiriladi, shuning uchun shunga yopishib olamiz. Quyida xmm registrida 16 bayt misoli:

| a | b | c | d | e | f | g | h | i | j | k | l | m | n | o | p |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Lekin bu sakkizta so‘z (16-bit butun sonlar) ham bo‘lishi mumkin

| a | b | c | d | e | f | g | h |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Yoki to‘rtta dubl so‘z (32-bit butun sonlar)

| a | b | c | d |
| :---- | :---- | :---- |

Yoki ikkita kvadso‘z (64-bit butun sonlar):

| a | b |
| :---- | :---- |

Qisqacha eslatma:


* **b**yte — 8-bit ma’lumot  
* **w**ord — 16-bit ma’lumot  
* **d**oubleword — 32-bit ma’lumot  
* **q**uadword — 64-bit ma’lumot  
* **d**ouble **q**uadword — 128-bit ma’lumot

Qalin harflar keyinroq muhim bo‘ladi.

**x86inc.asm include**  
Ko‘plab misollarda biz x86inc.asm faylini ulaymiz. x86inc.asm — bu FFmpeg, x264 va dav1d’da assembly dasturchining ishini yengillashtirish uchun ishlatiladigan yengil abstraksiya qatlamidir. U ko‘p jihatdan yordam beradi, ammo hozircha foydali tomoni shundaki, u GPR’larni r0, r1, r2 kabi yorliqlaydi. Bu sizga aniq registr nomlarini yodlab qolmaslik imkonini beradi. Yuqorida aytilganidek, GPR’lar odatda faqat “karkas”, shuning uchun bu ishni osonlashtiradi.

**Oddiy skalyar asm bo‘lagi**

Keling, sodda (va ancha sun’iy) skalyar asm bo‘lagini (har bir ko‘rsatmada bitta element ustida ishlaydigan kod) ko‘rib chiqamiz:

```assembly
mov  r0q, 3  
inc  r0q  
dec  r0q  
imul r0q, 5
```

Birinchi qatorda *immediate value* 3 (to‘g‘ridan-to‘g‘ri assembly kodning o‘zida saqlanadigan, xotiradan olinmaydigan qiymat) r0 registriga kvadso‘z sifatida yozilmoqda. Eslatib o‘tamiz, Intel sintaksisida manba operand (o‘ng tomonda joylashgan qiymat yoki manzil) ma’lumotni oluvchi manzilga, ya’ni qabul qiluvchi operandga (chap tomonda) ko‘chiriladi; bu memcpy xatti-harakatiga o‘xshaydi. Buni “r0q = 3” deb ham o‘qishingiz mumkin, chunki tartibi bir xil. r0’ning “q” suffiksi registr kvadso‘z sifatida ishlatilayotganini bildiradi. inc qiymatni 1 ga oshiradi, shunda r0q 4 bo‘ladi, dec qiymatni qayta 3 ga tushiradi. imul esa qiymatni 5 ga ko‘paytiradi. Demak oxirida r0q 15 bo‘ladi. 

Assembler orqali mashina kodiga yig‘iladigan mov va inc kabi inson o‘qiy oladigan ko‘rsatmalar *mnemonik* deb ataladi. Onlayn va kitoblarda mnemoniklar MOV va INC kabi katta harflarda yozilganini ko‘rishingiz mumkin, ammo ular kichik harfdagi variantlar bilan bir xil. FFmpeg’da biz mnemoniklarni kichik harflarda yozamiz va katta harflarni makrolar uchun saqlab qo‘yamiz.

**Oddiy vektor funksiyani tushunish**

Mana bizning birinchi SIMD funksiyamiz:

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

Qatorma-qator ko‘rib chiqamiz:

```assembly
%include "x86inc.asm"
```

Bu x264, FFmpeg va dav1d hamjamiyatlarida assembly yozishni soddalashtirish uchun yordamchilar, oldindan belgilangan nomlar va makrolarni (masalan, quyidagi cglobal) taqdim etish maqsadida ishlab chiqilgan “sarlavha” (header).

```assembly
SECTION .text
```

Bu bajariladigan kod joylashadigan bo‘limni bildiradi. .data bo‘limi esa qarama-qarshi bo‘lib, u yerga o‘zgarmas ma’lumotlarni joylashtirish mumkin.

```assembly
;static void add_values(uint8_t *src, const uint8_t *src2)  
INIT_XMM sse2
```

Birinchi qator sharh (asm’da “;” belgisi C’dagi “//” kabi) bo‘lib, funksiyaning C’dagi argument ko‘rinishini ko‘rsatadi. Ikkinchi qator esa funksiyani sse2 ko‘rsatmalar to‘plamidan foydalanib XMM registrlarini ishlatishga qanday initsializatsiya qilayotganimizni ko‘rsatadi. Chunki paddb sse2 ko‘rsatmasidir. sse2’ni keyingi darsda batafsil yoritamiz.

```assembly
cglobal add_values, 2, 2, 2, src, src2
```

Bu muhim qator, chunki u “add_values” nomli C funksiyasini belgilaydi. 

Keling, har bir elementni bittadan ko‘rib chiqamiz:

* Keyingi parametr funksiyada ikkita argument borligini ko‘rsatadi.   
* Undan keyingi parametr argumentlar uchun ikkita GPR’dan foydalanishimizni bildiradi. Ba’zi hollarda biz ko‘proq GPR ishlatmoqchi bo‘lishimiz mumkin, shuning uchun x86util’ga ko‘proq kerakligini aytishimiz kerak.   
* Keyingisi x86util’ga nechta XMM registridan foydalanishimizni bildiradi.  
* Keyingi ikkita parametr esa funksiya argumentlari uchun yorliqlardir.

Shuni ham aytib o‘tish joizki, eski kodlarda funksiya argumentlari uchun yorliqlar bo‘lmasligi mumkin va ularning o‘rniga GPR’lar r0, r1 va hokazo tarzda bevosita ko‘rsatiladi.

```assembly
    movu  m0, [srcq]  
    movu  m1, [src2q]
```

movu — movdqu (move double quad unaligned) uchun qisqartma. Tekislash (alignment) boshqa darsda yoritiladi, ammo hozircha movu’ni [srcq] manzilidan 128-bitli ko‘chirish sifatida qabul qilish mumkin. mov holatida qavslar [srcq] ichidagi manzil dereferens qilinayotganini bildiradi, bu C’dagi **src ga ekvivalent.* Bu “yuklash” (load) deb ataladi. E’tibor bering, “q” suffiksi ko‘rsatkichning o‘lchamiga ishora qiladi *(*ya’ni C’da u *sizeof(*src) == 8 bo‘ladi 64-bit tizimlarda, x86asm esa 32-bit tizimlarda 32-bitdan foydalanishni o‘zi tushunadi), lekin asosiy yuklash 128-bitdir.

Shuni ham yodda tutingki, biz vektor registrlariga bu yerda to‘liq nomi bilan (masalan, xmm0) emas, balki m0 kabi abstrakt shaklda murojaat qilamiz. Keyingi darslarda bu yondashuv bir marta kod yozib, uni bir nechta SIMD registr o‘lchamlarida ishlatish imkonini berishini ko‘rasiz.

```assembly
paddb m0, m1
```

paddb (buni xayolan *p-add-b* deb o‘qing) har bir registrdagi har bir baytni quyidagidek qo‘shadi. “p” prefiksi “packed”ni bildiradi va vektor ko‘rsatmalarini skalyar ko‘rsatmalardan ajratib turadi. “b” suffiksi esa bu baytlar bo‘yicha qo‘shish (bayt qo‘shish) ekanini ko‘rsatadi.

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

Bu “saqlash” (store) deb ataladi. Ma’lumotlar srcq ko‘rsatkichidagi manzilga qayta yoziladi.

```assembly
RET
```

Bu funksiya qaytishini bildiruvchi makro. FFmpeg’dagi deyarli barcha assembly funksiyalari qiymat qaytarish o‘rniga argumentlardagi ma’lumotlarni o‘zgartiradi.

Topshiriqda ko‘rasizki, biz assembly funksiyalarga funksiya ko‘rsatkichlarini yaratamiz va ular mavjud bo‘lganda ulardan foydalanamiz.

[Keyingi dars](../lesson_02/index.uz.md)
