**Lekcja Trzecia Języka Asemblera FFmpeg**

Wyjaśnimy teraz trochę więcej żargonu i przedstawimy krótką lekcję historii.

**Zestawy instrukcji**

Być może zauważyłeś w poprzedniej lekcji, że mówiliśmy o SSE2, które jest zestawem instrukcji SIMD. Gdy wydawana jest nowa generacja procesorów, może ona zawierać nowe instrukcje, a czasami większe rozmiary rejestrów. Historia zestawu instrukcji x86 jest bardzo skomplikowana, więc oto uproszczona historia (istnieje o wiele więcej podkategorii):

* MMX - Wydany w 1997 roku, pierwsze SIMD w procesorach Intela, 64-bitowe rejestry, historyczny
* SSE (Streaming SIMD Extensions) - Wydany w 1999 roku, 128-bitowe rejestry
* SSE2 - Wydany w 2000 roku, wiele nowych instrukcji
* SSE3 - Wydany w 2004 roku, pierwsze instrukcje poziome
* SSSE3 (Supplemental SSE3) - Wydany w 2006 roku, nowe instrukcje, ale co najważniejsze instrukcja pshufb shuffle, prawdopodobnie najważniejsza instrukcja w przetwarzaniu wideo
* SSE4 - Wydany w 2008 roku, wiele nowych instrukcji, w tym spakowane minimum i maksimum.
* AVX - Wydany w 2011 roku, 256-bitowe rejestry (tylko float) i nowa składnia trzech operandów
* AVX2 - Wydany w 2013 roku, 256-bitowe rejestry dla instrukcji całkowitoliczbowych
* AVX512 - Wydany w 2017 roku, 512-bitowe rejestry, nowa funkcja maski operacji. Miały ograniczone zastosowanie w FFmpeg ze względu na obniżanie częstotliwości procesora podczas używania nowych instrukcji. Pełne 512-bitowe tasowanie (permute) z vpermb.
* AVX512ICL - Wydany w 2019 roku, brak obniżania częstotliwości zegara.
* AVX10 - Nadchodzący

Warto zauważyć, że zestawy instrukcji mogą być również usuwane z procesorów, a nie tylko dodawane. Na przykład AVX512 został [usunięty](https://www.igorslab.de/en/intel-deactivated-avx-512-on-alder-lake-but-fully-questionable-interpretation-of-efficiency-news-editorial/), kontrowersyjnie, w 12. generacji procesorów Intela. To właśnie z tego powodu FFmpeg wykonuje wykrywanie CPU w czasie działania. FFmpeg wykrywa możliwości procesora, na którym jest uruchamiany.

Jak widziałeś w zadaniu, wskaźniki do funkcji są domyślnie w C i są zastępowane wariantem zestawu instrukcji. Oznacza to, że wykrywanie jest wykonywane raz i potem nie musi być już wykonywane ponownie. Jest to w przeciwieństwie do wielu własnościowych aplikacji, które na stałe kodują konkretny zestaw instrukcji, czyniąc w pełni funkcjonalny komputer przestarzałym. Pozwala to również na włączanie/wyłączanie zoptymalizowanych funkcji w czasie działania. To jest jedna z dużych zalet otwartego oprogramowania.

Programy takie jak FFmpeg są używane na miliardach urządzeń na całym świecie, z których niektóre mogą być bardzo stare. FFmpeg technicznie wspiera maszyny obsługujące tylko SSE, które mają 25 lat! Na szczęście x86inc.asm potrafi powiedzieć, jeśli używasz instrukcji, która nie jest dostępna w danym zestawie instrukcji.
Aby dać Ci wyobrażenie o rzeczywistych możliwościach, oto dostępność zestawów instrukcji z [Steam Survey](https://store.steampowered.com/hwsurvey/Steam-Hardware-Software-Survey-Welcome-to-Steam) na listopad 2024 (oczywiście jest to stronnicze wobec graczy):

| Zestaw instrukcji | Dostępność |
| :---- | :---- |
| SSE2 | 100% |
| SSE3 | 100% |
| SSSE3 | 99.86% |
| SSE4.1 | 99.80% |
| AVX | 97.39% |
| AVX2 | 94.44% |
| AVX512 (Steam does not separate between AVX512 and AVX512ICL) | 14.09% |

Dla aplikacji takiej jak FFmpeg, która ma miliardy użytkowników, nawet 0,1% to bardzo duża liczba użytkowników i zgłoszeń błędów, jeśli coś przestanie działać. FFmpeg posiada rozbudowaną infrastrukturę testową do testowania wariantów CPU/OS/Compiler w naszym [FATE testsuite](https://fate.ffmpeg.org/?query=subarch:x86_64%2F%2F). Każdy pojedynczy commit jest uruchamiany na setkach maszyn, aby upewnić się, że nic nie przestaje działać.

Intel zapewnia szczegółowy podręcznik zestawu instrukcji tutaj: [https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)

Przeszukiwanie pliku PDF może być uciążliwe, dlatego istnieje nieoficjalna alternatywa internetowa tutaj: [https://www.felixcloutier.com/x86/](https://www.felixcloutier.com/x86/)

Istnieje również wizualna reprezentacja instrukcji SIMD dostępna tutaj:
[https://www.officedaytime.com/simd512e/](https://www.officedaytime.com/simd512e/)

Częścią wyzwania asemblera x86 jest znalezienie odpowiedniej instrukcji do swoich potrzeb. W niektórych przypadkach instrukcje mogą być używane w sposób, do którego pierwotnie nie były przeznaczone.

**Sztuczki z przesunięciem wskaźnika**

Wróćmy do naszej oryginalnej funkcji z Lekcji 1, ale dodajmy argument szerokości (ang. width) do funkcji C.

Zamiast używać int dla zmiennej width, używamy ptrdiff_t, aby upewnić się, że górne 32 bity 64-bitowego argumentu są zerowe. Gdybyśmy bezpośrednio przekazali int width w sygnaturze funkcji, a następnie próbowali użyć go jako quad do arytmetyki wskaźników (np. używając `widthq`), górne 32 bity rejestru mogą być wypełnione dowolnymi wartościami. Moglibyśmy to naprawić przez rozszerzenie znaku width za pomocą `movsxd` (zobacz także makro `movsxdifnidn` w x86inc.asm), ale to jest łatwiejszy sposób.

Poniższa funkcja zawiera sztuczki z przesunięciem wskaźnika:

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

Przejdźmy przez to krok po kroku, ponieważ może to być mylące:

```assembly
   add srcq, widthq
   add src2q, widthq
   neg widthq
```

Szerokość jest dodawana do każdego wskaźnika, tak że każdy wskaźnik teraz wskazuje na koniec bufora do przetworzenia. Następnie szerokość jest negowana.

```assembly
    movu  m0, [srcq+widthq]
    movu  m1, [src2q+widthq]
```

Ładowania są następnie wykonywane z widthq będącym ujemnym. Tak więc w pierwszej iteracji [srcq+widthq] wskazuje na oryginalny adres srcq, czyli wskazuje z powrotem na początek bufora.

```assembly
    add   widthq, mmsize
    jl .loop
```

mmsize jest dodawany do ujemnego widthq, co przybliża go do zera. Warunek pętli to teraz jl (skok, jeśli mniejszy niż zero). Ta sztuczka oznacza, że widthq jest używany jednocześnie jako przesunięcie wskaźnika **oraz** jako licznik pętli, co oszczędza instrukcję cmp. Pozwala to również na użycie przesunięcia wskaźnika w wielu operacjach ładowania i zapisu, a także na użycie wielokrotności przesunięć wskaźnika, jeśli to konieczne (pamiętaj o tym przy zadaniu).

**Wyrównanie**

We wszystkich naszych przykładach używaliśmy movu, aby uniknąć tematu wyrównania. Wiele procesorów może ładować i zapisywać dane szybciej, jeśli dane są wyrównane, tzn. jeśli adres pamięci jest podzielny przez rozmiar rejestru SIMD. Tam, gdzie to możliwe, staramy się używać wyrównanych ładowań i zapisów w FFmpeg za pomocą mova.

In FFmpeg, av_malloc is able to provide aligned memory on the heap and the DECLARE_ALIGNED C preprocessor directive can provide aligned memory on the stack. If mova is used with an unaligned address, it will cause a segmentation fault and the application will crash. It’s also important to be sure that the alignment value corresponds to the SIMD register size, i.e 16 with xmm, 32 for ymm and 64 for zmm.

Oto jak wyrównać początek sekcji RODATA do 64 bajtów:

```assembly
SECTION_RODATA 64
```

Zauważ, że to tylko wyrównuje początek RODATA. Może być potrzebne dodanie bajtów wypełniających, aby upewnić się, że następna etykieta pozostaje na granicy 64 bajtów.

**Rozszerzanie zakresu**

Kolejnym tematem, którego do tej pory unikaliśmy, jest przepełnienie. Dzieje się tak na przykład, gdy wartość bajtu przekracza 255 po operacji takiej jak dodawanie lub mnożenie. Możemy chcieć wykonać operację, w której potrzebujemy wartości pośredniej większej niż bajt (np. słowa), lub potencjalnie chcemy pozostawić dane w tym większym rozmiarze pośrednim.

Dla bajtów bez znaku przychodzą z pomocą instrukcje punpcklbw (packed unpack low bytes to words) i punpckhbw (packed unpack high bytes to words).

Zobaczmy, jak działa punpcklbw. Składnia dla wersji SSE2 z podręcznika Intela jest następująca:

| PUNPCKLBW xmm1, xmm2/m128 |
| :---- |

Oznacza to, że jego źródło (prawa strona) może być rejestrem xmm lub adresem pamięci (m128 oznacza adres pamięci ze standardową składnią [base + scale*index + disp]), a miejsce docelowe to rejestr xmm.

Strona officedaytime.com powyżej ma dobry diagram pokazujący, co się dzieje:

![Co to jest](image1.png)

Widać, że bajty są przeplatane z dolnej połowy każdego rejestru odpowiednio. Ale co to ma wspólnego z rozszerzaniem zakresu? Jeśli rejestr src jest wypełniony zerami, to przeplata bajty w dst z zerami. To jest to, co nazywa się *zerowym rozszerzeniem*, ponieważ bajty są bez znaku. punpckhbw może być użyty do zrobienia tego samego dla wysokich bajtów.

Oto fragment pokazujący, jak to się robi:

```assembly
pxor      m2, m2 ; zero out m2

movu      m0, [srcq]
movu      m1, m0 ; make a copy of m0 in m1
punpcklbw m0, m2
punpckhbw m1, m2
```

```m0``` oraz ```m1``` teraz zawierają oryginalne bajty rozszerzone zerowo do słów. W następnej lekcji zobaczysz, jak instrukcje trzy-operandowe w AVX sprawiają, że drugi movu jest niepotrzebny.

**Rozszerzanie znaku**

Dane ze znakiem są nieco bardziej skomplikowane. Aby rozszerzyć zakres liczby całkowitej ze znakiem, musimy użyć procesu znanego jako [rozszerzanie znaku](https://pl.wikipedia.org/wiki/Rozszerzenie_znaku_liczby_binarnej). Polega to na wypełnieniu najbardziej znaczących bitów bitem znaku. Na przykład: -2 w int8_t to 0b11111110. Aby rozszerzyć znak do int16_t, MSB o wartości 1 jest powtarzany, tworząc 0b1111111111111110.
```pcmpgtb``` (packed compare greater than byte) może być użyty do rozszerzania znaku. Poprzez wykonanie porównania (0 > bajt), wszystkie bity w docelowym bajcie są ustawiane na 1, jeśli bajt jest ujemny, w przeciwnym razie bity w docelowym bajcie są ustawiane na 0. punpckX może być użyty jak powyżej do wykonania rozszerzenia znaku. Jeśli bajt jest ujemny, odpowiadający bajt to 0b11111111, w przeciwnym razie to 0x00000000. Przeplatanie wartości bajtu z wynikiem pcmpgtb wykonuje rozszerzenie znaku do słowa jako wynik.

```assembly
pxor      m2, m2 ; wyzeruj m2

movu      m0, [srcq]
movu      m1, m0 ; zrób kopię m0 w m1

pcmpgtb   m2, m0
punpcklbw m0, m2
punpckhbw m1, m2
```

Jak widać, jest dodatkowa instrukcja w porównaniu do przypadku bez znaku.

**Pakowanie (ang. packing)**

packuswb (pack unsigned word to byte) oraz packsswb pozwalają przejść od słowa do bajtu. Pozwalają one przeplatać dwa rejestry SIMD zawierające słowa w jeden rejestr SIMD z bajtem. Należy zauważyć, że jeśli wartości przekraczają zakres bajtu, zostaną one nasycone (tzn. ograniczone do największej wartości).

**Tasowanie (ang. shuffle)**

Tasowanie, znane również jako permutacje, są prawdopodobnie najważniejszą instrukcją w przetwarzaniu wideo, a pshufb (packed shuffle bytes), dostępne w SSSE3, jest najważniejszym wariantem.

Dla każdego bajtu odpowiadający bajt źródłowy jest używany jako indeks rejestru docelowego, z wyjątkiem sytuacji, gdy ustawiony jest MSB, wtedy bajt docelowy jest zerowany. Jest to analogiczne do następującego kodu w C (chociaż w SIMD wszystkie 16 iteracji pętli odbywają się równolegle):

```c
for(int i = 0; i < 16; i++) {
    if(src[i] & 0x80)
        dst[i] = 0;
    else
        dst[i] = dst[src[i]]
}
```
Tutaj znajduje się prosty przykład w asemblerze:

```assembly
SECTION_DATA 64

shuffle_mask: db 4, 3, 1, 2, -1, 2, 3, 7, 5, 4, 3, 8, 12, 13, 15, -1

section .text

movu m0, [srcq]
movu m1, [shuffle_mask]
pshufb m0, m1 ; Tasowanie m0 na podstawie m1
```

Zauważ, że użycie -1 dla czytelności jako indeksu tasowania (shuffle) zeruje bajt wyjściowy: -1 jako bajt to 0b11111111 (uzupełnienie do dwóch), więc ustawiony jest bit MSB (0x80).

[image1]: <data:image/png;base64,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>