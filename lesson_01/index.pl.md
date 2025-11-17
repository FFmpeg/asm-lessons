**Lekcja Pierwsza Języka Asemblera FFmpeg**

**Wprowadzenie**

Witaj w Szkole Języka Asemblera FFmpeg. Zrobiłeś pierwszy krok na najbardziej interesującej, wymagającej i satysfakcjonującej ścieżce w programowaniu. Te lekcje zapewnią Ci fundamenty wiedzy o tym, jak asembler jest wykorzystywany w FFmpeg i otworzą oczy na to, co naprawdę dzieje się w waszym komputerze.

**Wymagana wiedza**

* Znajomość języka C, w szczególności wskaźników. Jeśli nie znasz C, przestudiuj książkę [The C Programming Language](https://en.wikipedia.org/wiki/The_C_Programming_Language)  
* Matematyka na poziomie szkoły średniej (skalary vs wektory, dodawanie, mnożenie itp.)

**Czym jest język asemblera?**

Język asemblera to język programowania, w którym tworzy się kod odpowiadający bezpośrednio instrukcjom przetwarzanym przez procesor (CPU). Czytelny dla człowieka kod asemblera jest, jak sama nazwa wskazuje, *asemblowany* (przekładany) na dane binarne, znane jako *kod maszynowy*, które są zrozumiałe dla procesora. Kod języka asemblera bywa określany mianem „assembly” lub, w skrócie, „asm”.

Zdecydowana większość kodu asemblera w FFmpeg to tzw. SIMD (ang. Single Instruction Multiple Data – pojedyncza instrukcja, wiele danych). SIMD bywa czasem nazywane programowaniem wektorowym. Oznacza to, że dana instrukcja wykonuje operacje na wielu elementach danych jednocześnie. Większość języków programowania operuje na jednym elemencie danych naraz, co określa się mianem programowania skalarnego.

Jak można się domyślić, SIMD doskonale sprawdza się w przetwarzaniu obrazów, wideo i dźwięku, które składają się z dużej ilości danych ułożonych w pamięci sekwencyjnie. W procesorze dostępne są specjalistyczne instrukcje, które pomagają w przetwarzaniu takich danych sekwencyjnych.

W FFmpeg terminy „assembly function” (funkcja asemblera), „SIMD” i „vector(ise)” (wektor(yzacja)) są używane zamiennie. Wszystkie odnoszą się do tego samego: pisania funkcji w języku asemblera ręcznie, aby przetwarzać wiele elementów danych jednocześnie. Niektóre projekty mogą również określać je jako „assembly kernels” (jądra asemblera).

Wszystko to może brzmieć skomplikowanie, ale ważne jest, aby pamiętać, że w FFmpeg kod asemblera pisały osoby w wieku szkolnym. Jak w przypadku wszystkiego, nauka to w 50% żargon, a w 50% rzeczywista wiedza.

**Dlaczego piszemy w języku asemblera?**  
Aby przyspieszyć przetwarzanie multimediów. Często można uzyskać 10-krotną lub większą poprawę wydajności dzięki pisaniu kodu asemblera, co jest szczególnie ważne, gdy chcemy odtwarzać filmy w czasie rzeczywistym bez zacinania się. Oszczędza to również energię i wydłuża żywotność baterii. Warto zauważyć, że funkcje kodowania i dekodowania wideo są jednymi z najczęściej używanych funkcji na świecie, zarówno przez użytkowników końcowych, jak i przez duże firmy w ich centrach danych. Nawet niewielka poprawa szybkości sumuje się szybko.

Często można zobaczyć, online, ludzi używających *intrinsics*, które są funkcjami podobnymi do C, które mapują się na instrukcje asemblera, aby umożliwić szybszy dewelopemnt. W FFmpeg nie używamy intrinsics, ale zamiast tego piszemy kod asemblera ręcznie. Jest to obszar kontrowersji, ale intrinsics są zazwyczaj o około 10-15% wolniejsze niż ręcznie napisany asembler (zwolennicy intrinsics nie zgadzają się), w zależności od kompilatora. Dla FFmpeg każdy kawałek dodatkowej wydajności pomaga, dlatego piszemy bezpośrednio w kodzie asemblera. Istnieje również argument, że intrinsics są trudne do odczytania z powodu ich użycia „[Notacji Węgierskiej (Hungarian Notation)](https://pl.wikipedia.org/wiki/Notacja_w%C4%99gierska)”.

Możesz również zobaczyć *inline assembly* (wstawka asemblera, tj. nie używając intrinsics) w kilku miejscach w FFmpeg ze względów historycznych lub w projektach takich jak jądro Linuksa ze względu na bardzo specyficzne przypadki użycia. Jest to sytuacja, w której kod asemblera nie znajduje się w osobnym pliku, ale jest napisany wraz z kodem C. W projektach takich jak FFmpeg panuje powszechna opinia, że kod ten jest nieczytelny, słabo wspierany przez kompilatory i trudny w utrzymaniu.

I wreszcie – w internecie spotkasz wielu samozwańczych ekspertów twierdzących, że to wszystko jest zbędne, a kompilator może przeprowadzić całą tę „wektoryzację” za ciebie. Przynajmniej na etapie nauki – zignoruj ich. Niedawne testy, np. [projekcie dav1d](https://www.videolan.org/projects/dav1d.html), wykazały około dwukrotne przyspieszenie dzięki automatycznej wektoryzacji, podczas gdy wersje napisane ręcznie osiągały przyspieszenie rzędu 8x.

**Warianty języka asemblera**  
Te lekcje skupią się na języku asemblera x86 64-bitowym. Jest on również znany jako amd64, chociaż działa także na procesorach firmy Intel. Istnieją inne rodzaje asemblera dla innych procesorów, takich jak ARM czy RISC-V, i być może w przyszłości lekcje te zostaną o nie rozszerzone.

W internecie spotkasz dwa warianty składni asemblera x86: AT&T oraz Intel. Składnia AT&T jest starsza i trudniejsza w czytaniu w porównaniu do składni Intela. Dlatego będziemy używać składni Intela.

**Materiały pomocnicze**  
Możesz być zaskoczony informacją, że książki czy zasoby internetowe, takie jak Stack Overflow, nie są w tym przypadku zbyt pomocnymi źródłami wiedzy. Wynika to częściowo z naszego wyboru ręcznego pisania asemblera w składni Intela. Ale również dlatego, że większość zasobów internetowych koncentruje się na programowaniu systemów operacyjnych lub hardware'u, zazwyczaj wykorzystując kod inny niż SIMD. Asembler w FFmpeg skupia się w szczególności na wysokowydajnym przetwarzaniu obrazu i – jak sam zobaczysz – jest to wyjątkowo specyficzne podejście do programowania w tym języku. Niemniej jednak, po ukończeniu tych lekcji bez trudu zrozumiesz także inne zastosowania asemblera.

Wiele książek zagłębia się w szczegóły architektury komputerów, zanim w ogóle przejdzie do nauki asemblera. Jest to w porządku, jeśli to właśnie chcesz zgłębić, ale z naszego punktu widzenia przypomina to studiowanie budowy silnika przed nauką prowadzenia samochodu.

Warto jednak dodać, że diagramy w dalszych rozdziałach książki „The Art of 64-bit assembly”, ukazujące instrukcje SIMD i ich działanie w formie wizualnej, są bardzo pomocne: https://artofasm.randallhyde.com/

Dostępny jest serwer Discord, na którym odpowiadamy na pytania:
[https://discord.com/invite/Ks5MhUhqfB](https://discord.com/invite/Ks5MhUhqfB)

**Rejestry**  
Registers are areas in the CPU where data can be processed. CPUs don’t operate on memory directly, but instead data is loaded into registers, processed, and written back to memory. In assembly language, generally, you cannot directly copy data from one memory location to another without first passing that data through a register.

Rejestry to obszary wewnątrz procesora (CPU), w których mogą być przetwarzane dane. Procesory nie wykonują operacji bezpośrednio na pamięci. Zamiast tego dane są wczytywane do rejestrów, przetwarzane, a następnie zapisywane z powrotem do pamięci.
W języku asemblera generalnie nie można skopiować danych bezpośrednio z jednego miejsca w pamięci do drugiego bez uprzedniego przepuszczenia ich przez rejestr.

**Rejestry ogólnego przeznaczenia (GPR)**  
Pierwszym typem rejestru jest tzw. Rejestr Ogólnego Przeznaczenia (ang. General Purpose Register – GPR). Określa się je mianem „ogólnego przeznaczenia”, ponieważ mogą one zawierać zarówno dane, w tym przypadku wartości do 64 bitów, jak i adresy pamięci (wskaźniki). Wartość znajdująca się w rejestrze GPR może być przetwarzana za pomocą operacji takich jak dodawanie, mnożenie, przesunięcia bitowe itp.

W większości książek poświęconych asemblerowi całe rozdziały dedykowane są niuansom rejestrów GPR, ich tłu historycznemu itd. Wynika to z faktu, że GPR-y są kluczowe w programowaniu systemów operacyjnych czy inżynierii wstecznej. Jednak w kodzie asemblera pisanym dla FFmpeg, rejestry GPR pełnią raczej rolę rusztowania, a przez większość czasu ich zawiłości nie są potrzebne i pozostają ukryte.

**Rejestry wektorowe (SIMD)**  
Rejestry wektorowe (SIMD), jak sama nazwa wskazuje, zawierają wiele elementów danych. Istnieją różne typy rejestrów wektorowych:

* rejestry mm – rejestry MMX, rozmiar 64-bitowy, historyczne i obecnie rzadko używane.
* rejestry xmm – rejestry XMM, rozmiar 128-bitowy, szeroko dostępne.
* rejestry ymm – rejestry YMM, rozmiar 256-bitowy, pewne komplikacje przy ich używaniu.
* rejestry zmm – rejestry ZMM, rozmiar 512-bitowy, ograniczona dostępność.

Większość obliczeń w kompresji i dekompresji wideo opiera się na liczbach całkowitych (ang. integer-based), więc przy tym pozostaniemy. Oto przykład 16 bajtów (bytes) w rejestrze xmm:

| a | b | c | d | e | f | g | h | i | j | k | l | m | n | o | p |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Ale może to być również osiem słów (words) (16-bitowe liczby całkowite):

| a | b | c | d | e | f | g | h |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Lub cztery podwójne słowa (doublewords) (32-bitowe liczby całkowite):

| a | b | c | d |
| :---- | :---- | :---- | :---- |

Lub dwa poczwórne słowa (quadwords) (64-bitowe liczby całkowite):

| a | b |
| :---- | :---- |

Podsumowując:


* **b**ytes - 8-bitowe dane  
* **w**ords - 16-bitowe dane  
* **d**oublewords - 32-bitowe dane  
* **q**uadwords - 64-bitowe dane  
* **d**ouble **q**uadwords - 128-bitowe dane

Pogrubione znaki będą ważne później.

**Dołaczanie x86inc.asm**  

W wielu przykładach zauważysz, że dołączamy plik x86inc.asm. Jest to lekka warstwa abstrakcji używana w projektach FFmpeg, x264 oraz dav1d, mająca na celu ułatwienie życia programiście asemblera. Pomaga ona na wiele sposobów, ale na początek warto wspomnieć o jednej z jej przydatnych funkcji: nadaje ona GPR-om etykiety r0, r1, r2. Oznacza to, że nie musisz pamiętać nazw rejestrów. Jak wspomniano wcześniej, rejestry GPR są zazwyczaj tylko „rusztowaniem”, więc takie uproszczenie znacznie ułatwia pracę.

**Prosty fragment skalarnego asm**

Przyjrzyjmy się prostemu (i mocno sztucznemu) fragmentowi asemblera skalarnego (kodu, który w ramach jednej instrukcji operuje na pojedynczych elementach danych, jeden po drugim), aby zobaczyć, o co w tym chodzi:

```assembly
mov  r0q, 3  
inc  r0q  
dec  r0q  
imul r0q, 5
```

W pierwszej linii *immediate value (wartość natychmiastowa)* 3 (wartość zapisana bezpośrednio w samym kodzie asemblera, w przeciwieństwie do wartości pobranej z pamięci) jest zapisywana w rejestrze r0 jako poczwórne słowo (quadword). Zauważ, że w składni Intela operand źródłowy (wartość lub lokalizacja dostarczająca dane, znajdująca się po prawej) jest przenoszony do operandu docelowego (lokalizacja odbierająca dane, po lewej), co bardzo przypomina działanie funkcji memcpy. Możesz to również czytać jako „r0q = 3”, ponieważ kolejność jest taka sama. Sufiks „q” przy r0 oznacza, że rejestr ten jest używany jako poczwórne słowo. inc inkrementuje (zwiększa o 1) wartość, więc r0q zawiera 4.dec dekrementuje (zmniejsza o 1) wartość z powrotem do 3. imul mnoży wartość przez 5.Zatem na końcu r0q zawiera 15. 

Pamiętaj, że czytelne dla człowieka instrukcje, takie jak mov i inc, które są zamieniane przez asembler na kod maszynowy, znane są jako *mnemonics* (mnemoniki). W internecie i książkach możesz spotkać mnemoniki zapisane wielkimi literami, np. MOV i INC, ale oznaczają one to samo, co wersje małymi literami. W FFmpeg używamy mnemoników pisanych małymi literami, a wielkie litery rezerwujemy dla makr.

**Zrozumienie podstawowej funkcji wektorowej**

Oto nasza pierwsza funkcja SIMD:

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

Przejdźmy przez nią linia po linii:

```assembly
%include "x86inc.asm"
```

To jest "header" (nagłówek) opracowany w społecznościach x264, FFmpeg i dav1d, który dostarcza pomocnicze funkcje, predefiniowane nazwy i makra (takie jak cglobal poniżej), aby uprościć pisanie asemblera.

```assembly
SECTION .text
```

To oznacza sekcję, w której umieszczany jest kod, który ma być wykonywany. Jest to w przeciwieństwie do sekcji .data, w której można umieścić dane stałe.

```assembly
;static void add_values(uint8_t *src, const uint8_t *src2)  
INIT_XMM sse2
```

Pierwsza linia to komentarz (średnik „;” w asm jest jak „//” w C) pokazujący, jak wygląda argument funkcji w C. Druga linia pokazuje, jak inicjalizujemy funkcję do użycia rejestrów XMM, używając zestawu instrukcji sse2. Dzieje się tak, ponieważ paddb jest instrukcją sse2. Omówimy sse2 bardziej szczegółowo w następnej lekcji.

```assembly
cglobal add_values, 2, 2, 2, src, src2
```

To jest ważna linia, ponieważ definiuje funkcję C o nazwie „add_values”.

Przejdźmy przez każdy element po kolei:

* pierwszy parametr po cglobal to nazwa funkcji.
* następny parametr pokazuje, że ma dwa argumenty funkcji.
* parametr po nim pokazuje, że w tej funkcji użyjemy dwóch rejestrów GPR, w tym argumentów. W niektórych przypadkach możemy chcieć użyć więcej rejestrów GPR, więc musimy powiedzieć x86util, że potrzebujemy więcej.
* parametr po tym mówi x86util, ile rejestrów XMM zamierzamy użyć.
* dwa kolejne parametry to etykiety dla argumentów funkcji.

Warto zauważyć, że starszy kod może nie mieć etykiet dla argumentów funkcji, ale zamiast tego bezpośrednio adresować rejestry GPR, używając r0, r1 itd.

```assembly
    movu  m0, [srcq]  
    movu  m1, [src2q]
```

movi jest skrótem od movdqu (move double quad unaligned). Alignement (wyrównanie) zostanie omówione w innej lekcji, ale na razie movu można traktować jako 128-bitowe przeniesienie z [srcq]. W przypadku mov nawiasy oznaczają, że adres w [srcq] jest dereferencjonowany, co jest równoważne z **src w C.* Jest to to, co nazywane jest załadowaniem (load). Należy zauważyć, że przyrostek „q” odnosi się do rozmiaru wskaźnika *(*tj. w C reprezentuje *sizeof(*src) == 8 na systemach 64-bitowych, a x86asm jest na tyle inteligentny, że używa 32-bitowego na systemach 32-bitowych), ale podstawowe ładowanie jest 128-bitowe.

Zwróć uwagę, że nie odnosimy się do rejestrów wektorowych ich pełną nazwą, w tym przypadku xmm0, ale jako m0, przez formę abstrakcyjną. W przyszłych lekcjach zobaczysz, jak oznacza to, że możesz napisać napisać kod raz, w taki sposób, że będzie działał na wielu rozmiarach rejestrów SIMD.

```assembly
paddb m0, m1
```

paddb (czytaj to w myślach jako *p-add-b (p-dodaj-b*)) dodaje każdy bajt w każdym rejestrze, jak pokazano poniżej. Przedrostek „p” oznacza „packed” (spakowany) i służy do identyfikacji instrukcji wektorowych w przeciwieństwie do skalarnych. Przyrostek „b” pokazuje, że jest to bytewise addition (dodawanie bajtowe).

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

Jest to tak zwana operacja store (zapisu). Dane są zapisywane z powrotem pod adres wskazany przez wskaźnik srcq.

```assembly
RET
```

Jest to makro oznaczające powrót z funkcji. Praktycznie wszystkie funkcje asemblerowe w FFmpeg modyfikują dane w argumentach (w miejscu), zamiast zwracać konkretną wartość.

Jak zobaczysz w zadaniu, tworzymy wskaźniki na funkcje asemblerowe i wykorzystujemy je tam, gdzie są one dostępne.

[Następna lekcja](../lesson_02/index.pl.md)
