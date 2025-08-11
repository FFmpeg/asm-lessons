**Lezione Linguaggio Assembly FFmpeg Due**  

Ora che hai scritto la tua prima funzione in linguaggio assembly, introdurremo rami e cicli.  

Dobbiamo prima introdurre l'idea di etichette e salti. Nell'esempio artificiale qui sotto, l'istruzione jmp sposta l'istruzione del codice dopo ".loop:". ".loop:" è conosciuta come un'*etichetta*, il fatto che sia prefissata dal punto indica che è un *etichetta locale*, il che effettivamente ti permette di riusare lo stesso nome dell'etichetta attraverso varie funzioni. Questo esempio, ovviamente, mostra un ciclo infinito, ma estenderemo quest'ultimo più tardi a qualcosa di più realistico.

```assembly
mov  r0q, 3
.loop:
    dec  r0q
    jmp .loop
```

Prima di creare un ciclo realistico dobbiamo introdurre il registro *FLAGS* (Bandiere). Non ci soffermeremo troppo sulle complessità di *FLAGS*, (in quanto, di nuovo, le operazioni sui GPR fanno per la maggior parte da impalcatura) ma ci sono varie flags come la Zero-flag, la Sign-flag e la Overflow-flag che sono alzate o abbassate in base al risultato della maggior parte delle operazioni non-mov su dati scalari come operazioni aritmetiche e spostamenti.

Ecco un esempio dove il contatore del ciclo conta alla rovescia fino allo zero e jg (*jump if greater than zero*, salta se più di zero) è la condizione del ciclo. dec r0q imposta il valore delle FLAGs in base al valore di r0q dopo l'istruzione, e puoi quindi saltare in base a loro.

```assembly
mov  r0q, 3
.loop:
    ; fai qualcosa
    dec  r0q
    jg  .loop ; salta se più grande di zero
```

Questo è equivalente al seguente codice C:

```c
int i = 3;
do
{
   // fai qualcosa
   i--;
} while(i > 0)
```

Questo codice C è un po' insolito. Solitamente, un ciclo in C è scritto in questo modo:

```c
int i;
for(i = 0; i < 3; i++) {
    // fai qualcosa
}
```

Questo è grossolanamente uguale a (non c'è un modo smplice di paragonare questo ciclo ```for``` )

```assembly
xor r0q, r0q
.loop:
    ; fai qualcosa
    inc r0q
    cmp r0q, 3
    jl  .loop ; salta se (r0q - 3) < 0, ovvero (r0q < 3)
```

Ci sono alcune cose da notare in questo frammento di codice. Prima tra tutte è ```xor r0q, r0q``` ovvero un modo usuale per impostare un registro a zero, il chè in alcuni sistemi è più veloce di ```mov r0q, 0```, in quanto, semplicemente, non si sta effettivamente caricando nulla. Può anche essere usato su registri SIMD attraverso ```pxor m0, m0``` per azzerare un intero registro. La prossima cosa da notare è l'uso di cmp. Praticamente, cpm sottrae il secondo registro dal primo (senza memorizzare il risultato da nessuna parte) e imposta *FLAGS*, ma come scritto nel commento, può essere letto insieme al salto, (jl = *jump if less than zero*, salta se meno di zero) per saltare se ```r0q < 3```.

Da notare come c'è un istruzione in più in questo frammento. Generalmente, meno istruzioni significano codice più veloce, il che è il motivo per cui il frammento precedente è preferibile a questo. Come vedrai nelle lezioni future, ci sono più trucchi per evitare di scrivere istruzioni extra e avere *FLAGS* impostato da aritmetica o da altre operazioni. Da notare anche come non stiamo scrivendo assembly per corrispondere esattamente a cicli in C, ma stiamo scrivendo i cicli facendo in modo che siano il più veloce possibile in assembly.

A seguito alcune mnemoniche per i salti che andrai ad usare (ci sono anche *FLAGS* per completezza, ma non devi saperne le specifiche per scrivere cicli):

| Mnemonica | Descrizione  | FLAGS |
| :---- | :---- | :---- |
| JE/JZ | Salta se uguale/Zero | ZF = 1 |
| JNE/JNZ | Salta se Non uguale/Non zero | ZF = 0 |
| JG/JNLE | Salta se maggiore/non minore o uguale (con segno) | ZF = 0 and SF = OF |
| JGE/JNL | Salta se maggiore o uguale/Non minore (con segno) | SF = OF |
| JL/JNGE | Salta se minore/Non maggiore o uguale (con segno) | SF ≠ OF |
| JLE/JNG | Salta se non minore o uguale/Not maggiore (con segno) | ZF = 1 or SF ≠ OF |

**Costanti**

Vediamo degli esempi che ci mostrano come usare le costanti:

```assembly
SECTION_RODATA

constants_1: db 1,2,3,4
constants_2: times 2 dw 4,3,2,1
```

* SECTION_RODATA specifica che questa è una sezione di dati a sola lettura (*Read Only Data*). (Questa è una macro siccome i differenti formati di file di output usati dai sistemi operativi dichiarano questo in modo differente)
* constants\_1: L'etichetta constants\_1 è definita come ```db``` (dichiara byte) - è equivalente a uint8\_t constants\_1\[4\] = {1, 2, 3, 4};
* constants\_2: Questa usa la macro ```times 2``` per ripetere le parole dichiarate - è equivalente a uint16\_t constants\_2\[8\] = {4, 3, 2, 1, 4, 3, 2, 1};

Questa etichette, convertite poi in un indirizzo di memoria dall'assembler, possono essere usate in caricamenti (ma non scritture, in quanto sono a sola lettura). Alcune istruzioni prendono come operando un indirizzo di memoria quindi possono essere usate senza caricamenti espliciti in un registro (ci sono pro e contro nel fare questo).

**Offsets** (Sfalsamenti)  

Gli sfalsamenti sono la distanza (in byte) tra elementi consecutivi in memoria Lo sfalsamento è determinato dalla **dimensione di ogni elemento** nella struttura di dati.

Ora che sei in grado di scrivere cicli, è arrivata l'ora di andare a recuperare dati. Ci sono però alcune differenze rispetto al C. Osserviamo il seguente ciclo in C:

```c
uint32_t data[3];
int i;
for(i = 0; i < 3; i++) {
    data[i];
}
```

Lo sfalsamento di 4 byte tra gli elementi di data è precalcolato dal compilatore C. Quando però si scrive assembly a mano, bisogna anche calcolare lo sfalsamento a mano.

Guardiamo alla sintassi per il calcolo dell'indirizzo di memoria. Questo si applica a tutti i tipi di indirizzi di memoria.

```assembly
[base + scala*indice + spiazzamento]
```

* base - Questo è un GPR (solitamente un puntatore da un argomento di una funzione C)
* scala - Questa può essere 1, 2, 4, 8. 1 è il predefinito
* indice - Questo è un GPR (solitamente un contatore di ciclo)
* spiazzamento - Questo è un numero intero (fino a 32 bit). Lo spiazzamento è uno sfalsamento all'intero dei dati

x86asm provvede la costante mmsize, la quale ti fa sapere la dimensione del registro SIMD con cui stai lavorando.

Ecco un esempio semplice (e insensato) per illustrare il caricamento da sfalsamenti personalizzati:

```assembly
;static void simple_loop(const uint8_t *src)
INIT_XMM sse2
cglobal simple_loop, 1, 2, 2, src
     movq r1q, 3
.loop:
     movu m0, [srcq]
     movu m1, [srcq+2*r1q+3+mmsize]

     ; fai qualcosa

     add srcq, mmsize
dec r1q
jg .loop

RET
```

Da notare come su ```movu m1, [srcq+2*r1q+3+mmsize]``` l'assembler pre-calcolerà lo spiazzamento costante corretto da usare. Nella prossima lezione ti mostreremo un trucchetto per evitare il fare un add e un dec nel ciclo, rimpiazzandoli con un singolo add.

**LEA**  

Ora che comprendi gli sfalsamenti sarai in grado di usare lea (*Load Effective Address*, Carica Indirizzo Effettivo). Questo ti permette di eseguire moltiplicazione e addizione in una sola istruzione, che sarà più veloce rispetto all'uso di più istruzioni. Ci sono, ovviamente, limitazioni per cosa puoi effettivamente moltiplicare ed aggiungere ma ciò non influisce sul fatto che lea sia un istruzione molto potente.

```assembly
lea r0q, [base + scale*index + disp]
```

Contrario al suo nome, LEA può essere usato per normale aritmetica e calcolo d'indirizzo. Puoi fare qualcosa tanto complicato quanto:

```assembly
lea r0q, [r1q + 8*r2q + 5]
```

È da notare come questo non abbia effetto sui contenuti di r1q e r2q. Non ha effetto nemmeno su *FLAGS* (quindi non puoi saltare in base al risultato dell'operazione). Usando LEA si evitano tutte queste istruzioni e registri tempranei (questo codice non è equivalente in quanto add cambia *FLAGS*)

```assembly
movq r0q, r1q
movq r3q, r2q
sal  r3q, 3 ; shift arithmetic left 3 = * 8
add  r3q, 5
add  r0q, r3q
```

Vedrai come lea è molto usato per impostare indirizzi prima di cicli o per eseguire calcoli come qui sopra. Ovviamente è da notare che non è possibile eseguire ogni tipo di moltiplicazione e addizione, ma moltiplicazioni per 1, 2, 4, 8 e addizioni di uno sfalsamento fisso sono comuni.

Nel compito dovrai caricare una costante ed addizionare i valori ad un vettore SIMD in un ciclo

[Prossima Lezione](../lesson_03/index.it.md)
