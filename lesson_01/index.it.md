**Linguaggio Assembly FFmpeg Lezione Uno**

**Introduzione**

Benvenuto alla Scuola del Linguaggio Assembly FFmpeg. Hai fatto il primo passo in uno dei viaggi più interessanti, impegnativi e soddisfacenti nella programmazione. Queste lezioni ti daranno una base nel modo in cui il linguaggio assembly è scritto in FFmpeg e ti faranno aprire gli occhi a cosa sta effettivamente succedendo all'interno del tuo computer.

**Conoscenze Richieste**

* Conoscenza del C, in particolare dei puntatori. Se non sei familiare con il C, leggi e studia [The C Programming Language](https://en.wikipedia.org/wiki/The_C_Programming_Language)
* Matematica delle superiori (scalare vs vettore, addizione, moltiplicazione, ecc.)

**Cos'è il linguaggio assembly?**

Il linguaggio assembly è un linguaggio di programmazione dove il codice che scrivi corrisponde alle istruzioni che una CPU processa. Il linguaggio assembly leggibile dagli umani viene, come suggerito dal nome, *assemblato* in dati binari, conosciuti come *codice macchina*, comprensibile dalla CPU. Potresti vedere il linguaggio assembly riferito come "assemblatore", abbreviato in "asm".

La gran maggioranza del codice di FFmpeg è ciò che è noto come *SIMD, Single Instruction Multiple Data (Istruzione Singola Dati Multipli)*. SIMD è a volte denominata programmazione vettoriale. La maggior parte dei linguaggi di programmazione operano su un elemento di dati alla volta, conosciuto come programmazione scalare.

Come avrai potuto intuire, SIMD si presta bene nel processare immagini, video ed audio in quanto hanno molti dati ordinati sequenzialmente in memoria. Ci sono istruzioni specializzate disponibili nella CPU per aiutarci a processare dati in sequenza.

In FFmpeg, vedrai i termini "funzione assembly", "SIMD", e "vettorizzare" usati intercambiabilmente. Fanno tutti riferimento alla stessa cosa: Scrivere una funzione nel linguaggio assembly a mano per processare più elementi di dati in una sola volta. Alcuni progetti potrebbero far riferimento a questi come "nuclei dell' assemblatore".

Tutto questo potrebbe sembrare complicato, ma è importante ricordare che in FFmpeg, degli studenti delle superiori hanno scritto codice assembly. Come con tutto, imparare è al 50% terminologia, e al 50% effettivamente imparare.

**Perchè scriviamo in linguaggio assembly?**
Per rendere l'elaborazione multimediale veloce. È molto comune raggiungere un miglioramento di velocità anche di 10 o più volte scrivendo codice assembly, il che è specialmente importante per riprodurre video in tempo reale senza interruzioni. Oltretutto, ciò risparmia energia ed estende l'autonomia delle batterie. È giusto evidenziare che funzioni di codifica e decodifica video sono alcune delle più usate nel mondo, sia da utenti finali che da grandi compagnie e datacenter. Perciò anche un piccolo miglioramento si accumula velocemente.

Online, vedrai spesso che le persone usano *intrinseche*, ovvero funzioni simili al C che mappano ad istruzioni assembly, permettendo uno sviluppo più veloce. In FFmpeg non usiamo intrinseche, ma piuttosto scriviamo codice assembly a mano. Questa rappresenta un area controversa, ma le intrinseche sono solitamente più lente del 10-15% rispetto all'assembly a mano (coloro che supportano le intrinseche non sarebbero d'accordo), in base al compilatore. Per FFmpeg, ogni piccolo miglioramento di velocità aiuta, e per questo scriviamo in codice assembly direttamente. C'è anche chi sostiene che le intrinseche siano difficili da leggere a causa dell'uso della "[Notazione Ungara](https://it.wikipedia.org/wiki/Notazione_ungara)"

Potresti anche vedere *assembly in linea* (ovvero senza usare intrinseche) da qualche parte in FFmpeg per ragioni storiche, o in progetti come il Kernel Linux a causa di casi d'uso molto specifici. Questo è quando il codice assembly non è in un file separato, ma scritto in linea con codice C. L'opinione prevalente in progetti come FFmpeg è che questo codice è difficile da leggere, non ampiamente supportato da compilatori e non manutenibile.

Infine, in rete vedrai molti esperti auto-proclamati tali che affermano che nulla di tutto ciò è necessario e che il compilatore può fare tutta questa "vettorizzazione" al posto tuo. Almeno per lo scopo di imparare, ignorali. test recenti, ad esempio nel [the dav1d project](https://www.videolan.org/projects/dav1d.html) mostrano una velocizzazione circa di 2 volte grazie a questa vettorizzazione automatica, mentre le versioni scritte a mano riuscivano a raggiungere le 8 volte.

**Gusti del linguaggio assembly**

Queste lezioni si concentreranno sull'assembly x86 a 64 bit. Questo è anche conosciuto come amd64, anche se funziona lo stesso su CPU Intel. Ci sono altri tipi di assembly per altre CPU, come ARM e RISC-V e potenzialmente nel futuro queste lezioni saranno estese per coprire tali.

Ci sono due gusti di sintassi di assembly x86 che vedrai in rete: AT&T ed Intel. La sintassi AT&T è più complicata rispetto alla sintassi Intel, quindi useremo la sintassi Intel.

**Materiali di supporto**
Potresti sorprenderti al fatto che libri o risorse in rete come Stack Overflow non sono particolarmente utili come riferimenti. Questo è in parte a causa della nostra scelta di scrivere codice assembly a mano con sintassi Intel. Ma è anche perchè molte risorse in rete si concentrano alla programmazione di sistemi operativi o alla programmazione hardware, solitamente usando codice non-SIMD. L'assembly FFmpeg è particolarmente concentrato alla processazione di immagini ad alte prestazioni, e come vedrai è un approccio particolarmente unico alla programmazione in assembly. Detto ciò, sarà facile comprendere altri casi d'uso dell'assembly una volta che avrai completato queste lezioni.

Molti libri vanno molto sui dettagli dell'architettura del computer prima di insegnare l'assembly. Questo va bene se vuoi imparare quello, ma dal nostro punto di vista, è come studiare come funziona un motore prima di imparare a guidare.

Detto ciò, i diagrammi nelle parti verso la fine del libro "The Art of 64-bit assembly" che mostrano istruzioni SIMD ed il loro comportamento in modo visuale sono utili: [https://artofasm.randallhyde.com/](https://artofasm.randallhyde.com/)

Un server discord è disponibile per rispondere a domande:
[https://discord.com/invite/Ks5MhUhqfB](https://discord.com/invite/Ks5MhUhqfB)

**Registri**
I registri sono aree nella CPU dove i dati possono essere processati. Le CPU non eseguiranno operazioni direttamente sulla memoria, ma invece i dati verrano prima caricati nei registri, processati e poi scritti nuovamente in memoria. Nel linguaggio assembly, in generale, non è possibile copiare dati da un luogo in memoria ad un altro senza prima far passare quei dati attraverso un registro.

**Registri a Scopo Generale**
Il primo tipo di registro è conosciuto come Registro a Scopo Generale, in inglese *General Purpose Register* (GPR). I GPR sono riferiti come a scopo generale in quanto possono contenere dati, in questo caso un valore a 64 bit, o un indirizzo di memoria (un puntatore). Un valore in un GPR può essere processato attraverso operazioni come addizione, moltiplicazione, spostamento, ecc.

nella maggior parte dei libri di assembly, ci sono capitoli dedicati alle sottigliezze dei GPR, il loro contesto storico ecc. Questo è perchè i GPR sono importanti quando viene alla programmazione di sistemi operativi, all'ingegneria inversa, ecc. Nel codice assembly scritto in FFmpeg, i GPR fanno più da impalcatura che altro e la maggior parte delle volte le loro complessità non sono necessarie e vengono astratte.

**Registri vettoriali**
I registri vettoriali (SIMD), come suggerisce il nome, contengono più elementi di dati. Ci sono vari tipi di registri vettoriali:

* Registri mm - Registri MMX, grandi 64 bit, storici ed ora non usati più di tanto
* Registri xmm - Registri XMM, grandi 128 bit, ad ampia disposizione
* Registri ymm - Registri YMM, grandi 256 bit, ci sono alcune complicazioni usando questi
* Registri ZMM - Registri ZMM, grandi 512 bit, a disponibilità limitata

La maggior parte dei calcoli nella compressione e decompressione video sono basati su numeri interi, quindi ci atterremo a questi. Ecco un esempio di 16 byte in un registro xmm:


| a | b | c | d | e | f | g | h | i | j | k | l | m | n | o | p |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Ma potrebbero essere otto parole (interi a 16 bit)

| a | b | c | d | e | f | g | h |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

O quattro doppie parole (interi a 32 bit)

| a | b | c | d |
| :---- | :---- | :---- | :---- |

O due parole quadruple (interi a 64 bit)

| a | b |
| :---- | :---- |

Riassumendo:

* byte - dati a 8 bit
* parole (**w**ords) - dati a 16 bit
* doppie parole (**d**oublewords) - dati a 32 bit
* parole quadruple (**q**uadwords) - dati a 64 bit
* doppie parole quadruple (**d**ouble **q**uadwords) - dati a 128 bit

I caratteri in grassetto saranno importanti dopo.

**x86inc.asm include**
Vedrai come in molti esempi includeremo il file x86inc.asm. x86inc.asm è un livello d'astrazione leggero usato in FFmpeg, x264 e dav1d per rendere la vita di un programmatore d'assembly più semplice. Aiuta in molti modi, ma tanto per iniziare, una delle cose che fa è etichettare i GPR, r0, r1, r2. Questo significa che non dovrai ricordarti i nomi dei registri. Come menzionato prima, i GPR sono generalmente solo impalcature quindi questo ci semplifica la vita.

**Un semplice estratto du asm scalare**

Diamo un occhio a questo estratto semplice (e molto artificiale) di asm scalare (codice assembly che opera su elementi di dati individuali, uno alla volta, all'interno di ogni Istruzione) per vedere che sta succedendo:

```assembly
mov  r0q, 3  
inc  r0q  
dec  r0q  
imul r0q, 5
```

nella prima linea, il *valore immediato* 3 (un valore memorizzato direttamente nel codice assembly stesso, opposto ad un valore recuperato dalla memoria) è memorizzato nel registro r0 come parola quadrupla. Da notare che nella sintassi Intel, l'operando di origine (il valore o il luogo che fornisce i dati, trovatosi alla destra) è trasferito all'operando di destinazione (il luogo ricevente i dati, trovatosi alla sinistra), simile al comportamento di memcpy. Può anche assere letto come "r0q = 3", dato che l'ordine è lo stesso. il suffisso di r0 "q" designa il registro ad essere usato come parola quadrupla. inc incrementa il valore facendo in modo che r0q contenga 4, dec decrementa il valore nuovamente a 3. imul moltiplica il valore per 5. Quindi alla fine, r0q contiene 15.

È da notare che le istruzioni leggibili dagli umani come mov e inc, assemblate poi in codice macchina dall'assembler, sono chiamate *mnemoniche*. Potresti vedere in rete e su libri mnemoniche rappresentate in maiuscolo con lettere come MOV e INC ma queste sono le stesse delle loro versioni in minuscolo. In FFmpeg, utilizziamo mnemoniche in minuscolo e teniamo il maiuscolo riservato per le macro.

**Comprendere una funzione vettoriale di base**

Ecco la nostra prima funzione SIMD:

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

Attraversiamo il codice linea per linea:

```assembly
%include "x86inc.asm"
```

Questa è una "intestazione" sviluppata nelle comunità di x264, FFmpeg e dav1d per dare aiutanti, nomi e macro predefiniti (come cglobal sotto) per semplificare la scrittura dell'assembly.

```assembly
SECTION .text
```

Questo denota la sezione dove è piazzato il codice che si vuole eseguire. Questo è in contrasto alla sezione .data, dove possono essere inseriti i dati costanti.

```assembly
;static void add_values(uint8_t *src, const uint8_t *src2)  
INIT_XMM sse2
```

La prima riga è un commento (il punto e virgola ";" in asm è come "//" in C) che ci mostra come l'argomento della funzione appare in C. La seconda riga ci mostra come stiamo inizializzando la funzione per usare i registri XMM, usando il set di istruzioni sse2. Questo perchè paddb è un istruzione sse2. Copriremo l'sse2 in ulteriori dettagli nella prossima lezione.

```assembly
cglobal add_values, 2, 2, 2, src, src2
```

Questa è una riga importante in quanto definisce una funzione C di nome "add_values". 

Attraversiamo ogni elemento uno per volta:

* Il parametro dopo mostra cbe ha due argomenti della funzione.
* Il parametro successivo mostra che useremo due GPR come argomenti. In alcuni casi potremo voler usare più GPR, quindi dovremo dire a x86util che ce ne servono di più.
* Il parametro successivo dice a x86util quanti registri XMM useremo.
* I due parametri dopo sono le etichette per gli argomenti della funzione.

Vale la pena notare che codice più vecchio potrebbe non avere etichette per gli argomenti della funzione ma invece potrebbe riferirsi ai GPR usando direttamente r0, r1 ecc.

```assembly
    movu  m0, [srcq]  
    movu  m1, [src2q]
```

movu è l'abbreviazione di movdqw (move double quad unaligned, sposta doppia quadrupla non allineata). Parleremo dell'allineamento in un'altra lezione, ma per ore movu può essere trattato come uno spostamento a 128 bit da [srcq]. Nel caso di mov, le parentesi quadre indicano che l'indirizzo in [srcq] sta venendo dereferenziato, l'equivalente di **src in C.* Questo è conosciuto come un caricamento (load). Da notare che il suffisso "q" si riferisce alla dimensione del puntatore *(*ovvero in C rappresenta *sizeof(*src) == 8 su sistemi a 64 bit, e x86asm è abbastanza intelligente da usare 32 bit su sistemi a 32 bit) ma l'operazione di caricamento sottostante rimane a 128 bit.

È da notare come non ci riferiamo ai registri vettoriali con il loro nome intero, in questo caso xmm0, ma come m0, una forma astratta. Nelle lezioni future vedremo come ciò significa sia possibile scrivere codice una volta sola e fare in modo funzioni su varie dimensioni di registri SIMD.


```assembly
paddb m0, m1
```

paddb (leggilo come *p-add-b*, p aggiunge b) sta aggiungendo ogni byte ad ogni registro come mostrato qui sotto. Il prefisso "P" sta per "Packed" (Impacchettato) ed è usato per identificare istruzioni vettoriali vs istruzioni scalari.



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

Questo è noto come un immagazzinaggio (*store*). I dati sono scritti all'indirizzo nel puntatore srcq.

```assembly
RET
```

Questa è una macro per denotare il ritorno della funzione. Praticamente tutte le funzioni in FFmpeg modificano i dati negli argomenti invece che far ritornare un valore.

Come vedrai nel compito, creeremo puntatori di funzione a funzioni assembly e useremo questi dove possiamo

[Next Lesson](../lesson_02/index.it.md)
