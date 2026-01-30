**Lekcja Druga Języka Asemblera FFmpeg**

Teraz, gdy napisałeś swoją pierwszą funkcję w języku asemblera, wprowadzimy instrukcje skoków i pętle.

Zaczniemy od wprowadzenia idei etykiet i skoków. W sztucznym przykładzie poniżej instrukcja jmp przenosi instrukcję kodu za „.loop:”. „.loop:” jest znane jako *etykieta* (ang. *label*), a kropka poprzedzająca etykietę oznacza, że jest to *lokalna etykieta*, co skutecznie pozwala na ponowne użycie tej samej nazwy etykiety w różnych funkcjach. Ten przykład, oczywiście, pokazuje nieskończoną pętlę, ale rozszerzymy to później na coś bardziej realistycznego.

```assembly
mov  r0q, 3
.loop:
    dec  r0q
    jmp .loop
```

Zanim stworzymy realistyczną pętlę, musimy wprowadzić rejestr *FLAGS* (z ang. FLAGI). Nie będziemy zbytnio zagłębiać się w zawiłości *FLAGS* (ponownie, ponieważ operacje na GPR są głównie szkieletem), ale istnieje kilka flag, takich jak Flaga Zerowa (Zero-Flag), Flaga Znaku (Sign-Flag) i Flaga Przepełnienia (Overflow-Flag), które są ustawiane na podstawie wyniku większości instrukcji innych niż mov na danych skalarnych, takich jak operacje arytmetyczne i przesunięcia.

Tutaj jest przykład, w którym licznik pętli odlicza do zera, a jg (skok, jeśli większe niż zero) jest warunkiem pętli. dec r0q ustawia *FLAGS* na podstawie wartości r0q po instrukcji i można na ich podstawie wykonać skok.

```assembly
mov  r0q, 3
.loop:
    ; wnętrze pętli
    dec  r0q
    jg  .loop ; skok jeśli większe niż zero
```
Jest to równoważne z następującym kodem w języku C:

```c
int i = 3;
do
{
   // wnętrze pętli
   i--;
} while(i > 0)
```

Powyższy kod w języku C jest nieco nietypowy. Zazwyczaj pętla w języku C jest zapisywana w ten sposób:

```c
int i;
for(i = 0; i < 3; i++) {
    // wnętrze pętli
}
```

Jest to mniej więcej równoważne z (nie ma prostego sposobu na dopasowanie tej pętli ```for```):

```assembly
xor r0q, r0q
.loop:
    ; wnętrze pętli
    inc r0q
    cmp r0q, 3
    jl  .loop ; skok jeśli (r0q - 3) < 0, tzn. (r0q < 3)
```

W tym fragmencie jest kilka rzeczy, na które warto zwrócić uwagę. Po pierwsze, ```xor r0q, r0q``` jest powszechnym sposobem ustawiania rejestru na zero, który na niektórych systemach jest szybszy niż ```mov r0q, 0```, ponieważ, mówiąc prosto, nie dochodzi do faktycznego ładowania. Może być również używany na rejestrach SIMD z ```pxor m0, m0```, aby wyzerować cały rejestr. Następna rzecz to użycie cmp. cmp efektywnie odejmuje drugi rejestr od pierwszego (bez przechowywania wartości gdziekolwiek) i ustawia *FLAGS*, ale jak wynika z komentarza, można go czytać razem ze skokiem (jl = skok jeśli mniej niż zero), aby wykonać skok, jeśli ```r0q < 3```.

Zauważ, że w tym fragmencie jest jedna dodatkowa instrukcja (cmp). Ogólnie rzecz biorąc, mniej instrukcji oznacza szybszy kod, dlatego preferowany jest wcześniejszy fragment. Jak zobaczysz w przyszłych lekcjach, istnieją kolejne sztuczki używane do unikania tej dodatkowej instrukcji i ustawiania *FLAGS* przez operacje arytmetyczne lub inne. Zauważ, że nie piszemy asemblera, aby dokładnie dopasować pętle w języku C, piszemy pętle tak, aby były jak najszybsze w asemblerze.

Poniżej znajdują się niektóre popularne mnemotechniki skoków, których będziesz używać (flagi są podane dla kompletności, ale nie musisz znać szczegółów, aby pisać pętle):

| Mnemonika | Opis  | FLAGS |
| :---- | :---- | :---- |
| JE/JZ | Jump if Equal/Zero (skok jeśli równe/zero) | ZF = 1 |
| JNE/JNZ | Jump if Not Equal/Not Zero (skok jeśli nierówne/nie zero) | ZF = 0 |
| JG/JNLE | Jump if Greater/Not Less or Equal (signed) (skok jeśli większe/nie mniejsze lub równe, znakowane) | ZF = 0 and SF = OF |
| JGE/JNL | Jump if Greater or Equal/Not Less (signed) (skok jeśli większe lub równe/nie mniejsze, znakowane) | SF = OF |
| JL/JNGE | Jump if Less/Not Greater or Equal (signed) (skok jeśli mniejsze/nie większe lub równe, znakowane) | SF ≠ OF |
| JLE/JNG | Jump if Less or Equal/Not Greater (signed) (skok jeśli mniejsze lub równe/nie większe, znakowane) | ZF = 1 or SF ≠ OF |

**Stałe (ang. Constants)**

Spójrzmy na kilka przykładów pokazujących, jak używać stałych:

```assembly
SECTION_RODATA

constants_1: db 1,2,3,4
constants_2: times 2 dw 4,3,2,1
```

* SECTION_RODATA określa, że jest to sekcja danych tylko do odczytu. (Jest to makro, ponieważ różne formaty plików wyjściowych używane przez systemy operacyjne deklarują to inaczej)
* constants_1: Etykieta constants_1 jest zdefiniowana jako ```db``` (declare byte - deklaruj bajt) - tzn. równoważne z uint8_t constants_1[4] = {1, 2, 3, 4};
* constants_2: To używa makra ```times 2```, aby powtórzyć zadeklarowane słowa - tzn. równoważne z uint16_t constants_2[8] = {4, 3, 2, 1, 4, 3, 2, 1};

Etykiety te, które asembler konwertuje na adres pamięci, mogą być następnie używane w operacjach ładowania (ale nie zapisywania, ponieważ są tylko do odczytu). Niektóre instrukcje przyjmują adres pamięci jako operand, więc mogą być używane bezpośrednio bez jawnego ładowania do rejestru (ma to swoje zalety i wady).

**Offsety (ang. Offsets)**

Offsety są odległością (w bajtach) między kolejnymi elementami w pamięci. Offset jest określany przez **rozmiar każdego elementu** w strukturze danych.

Teraz, gdy potrafimy pisać pętle, czas pobrać dane. Ale istnieją pewne różnice w porównaniu z językiem C. Spójrzmy na następującą pętlę w języku C:

```c
uint32_t data[3];
int i;
for(i = 0; i < 3; i++) {
    data[i];
}
```

Offset 4-bajtowy między elementami danych jest wstępnie obliczany przez kompilator języka C. Jednak podczas ręcznego pisania asemblera musisz samodzielnie obliczać te offsety.

Zobaczmy składnię obliczeń adresów pamięci. Ma to zastosowanie do wszystkich typów adresów pamięci:

```assembly
[base + scale*index + disp]
```

* base - baza, jest to GPR (zazwyczaj wskaźnik z argumentu funkcji C)
* scale - skala, może być równa 1, 2, 4, 8. Domyślnie jest to 1
* index - indeks, jest to GPR (zazwyczaj licznik pętli)
* disp - przesunięcie jest liczbą całkowitą (do 32-bitów). Przesunięcie jest offsetem do danych

x86asm zapewnia stałą mmsize, która informuje o rozmiarze rejestru SIMD, z którym pracujesz.

Tutaj jest prosty (i bezsensowny) przykład ilustrujący ładowanie z niestandardowych offsetów:

```assembly
;static void simple_loop(const uint8_t *src)
INIT_XMM sse2
cglobal simple_loop, 1, 2, 2, src
     movq r1q, 3
.loop:
     movu m0, [srcq]
     movu m1, [srcq+2*r1q+3+mmsize]

     ; tutaj można coś zrobić 

     add srcq, mmsize
dec r1q
jg .loop

RET
```

Zauważ, jak w ```movu m1, [srcq+2*r1q+3+mmsize]``` asembler wstępnie obliczy odpowiednią stałą przesunięcia do użycia. W następnej lekcji pokażemy Ci sztuczkę, aby uniknąć konieczności wykonywania add i dec w pętli, zastępując je pojedynczym add.

**LEA**

Teraz, gdy rozumiesz offsety, możesz używać lea (Load Effective Address - załaduj efektywny adres). Pozwala to na wykonywanie mnożenia i dodawania za pomocą jednej instrukcji, co będzie szybsze niż używanie wielu instrukcji. Oczywiście istnieją ograniczenia dotyczące tego, przez co można mnożyć i co dodawać, ale nie przeszkadza to w byciu potężną instrukcją.

```assembly
lea r0q, [base + scale*index + disp]
```

Pomimo swojej nazwy, LEA może być używane zarówno do normalnej arytmetyki, jak i do obliczeń adresów. Możesz zrobić coś tak skomplikowanego jak:

```assembly
lea r0q, [r1q + 8*r2q + 5]
```

Zauważ, że nie wpływa to na zawartość r1q i r2q. Nie wpływa to również na *FLAGS* (więc nie możesz wykonywać skoków na podstawie wyniku). Użycie LEA unika wszystkich tych instrukcji i tymczasowych rejestrów (ten kod nie jest równoważny, ponieważ add zmienia *FLAGS*):

```assembly
movq r0q, r1q
movq r3q, r2q
sal  r3q, 3 ; shift arithmetic left 3 - przesunięcie w lewo o 3 = * 8
add  r3q, 5
add  r0q, r3q
```

Zobaczysz, że lea jest często używane do ustawiania adresów przed pętlami lub wykonywania obliczeń jak powyżej. Oczywiście pamiętaj, że nie możesz wykonywać wszystkich rodzajów mnożenia i dodawania, ale mnożenia przez 1, 2, 4, 8 oraz dodawanie stałego przesunięcia jest powszechne.

W następnym zadaniu będziesz musiał załadować stałą i dodać wartości do wektora SIMD w pętli.

[Następna lekcja](../lesson_03/index.pl.md)
