**Μάθημα Τρίτο Γλώσσας Assembly του FFmpeg**

Ας εξηγήσουμε λίγη ακόμα ορολογία και να σας κάνουμε ένα σύντομο μάθημα ιστορίας.

**Σύνολα Εντολών (Instruction Sets)**

Μπορεί να είδατε στο προηγούμενο μάθημα ότι μιλήσαμε για το SSE2, το οποίο είναι ένα σύνολο εντολών SIMD. Όταν κυκλοφορεί μια νέα γενιά CPU, μπορεί να συνοδεύεται από νέες εντολές και μερικές φορές μεγαλύτερα μεγέθη καταχωρητών. Η ιστορία του συνόλου εντολών x86 είναι πολύ περίπλοκη, οπότε αυτή είναι μια απλοποιημένη ιστορία (υπάρχουν πολλές περισσότερες υποκατηγορίες):

* MMX - Κυκλοφόρησε το 1997, το πρώτο SIMD σε επεξεργαστές Intel, καταχωρητές 64-bit, ιστορικό
* SSE (Streaming SIMD Extensions) - Κυκλοφόρησε το 1999, καταχωρητές 128-bit
* SSE2 - Κυκλοφόρησε το 2000, πολλές νέες εντολές
* SSE3 - Κυκλοφόρησε το 2004, πρώτες οριζόντιες εντολές (horizontal instructions)
* SSSE3 (Supplemental SSE3) - Κυκλοφόρησε το 2006, νέες εντολές αλλά το πιο σημαντικό η εντολή αναδιάταξης (shuffle) pshufb, αναμφισβήτητα η πιο σημαντική εντολή στην επεξεργασία βίντεο
* SSE4 - Κυκλοφόρησε το 2008, πολλές νέες εντολές συμπεριλαμβανομένων των packed minimum και maximum.
* AVX - Κυκλοφόρησε το 2011, καταχωρητές 256-bit (μόνο για κινητή υποδιαστολή - float) και νέα σύνταξη τριών τελεστέων
* AVX2 - Κυκλοφόρησε το 2013, καταχωρητές 256-bit για εντολές ακεραίων
* AVX512 - Κυκλοφόρησε το 2017, καταχωρητές 512-bit, νέα δυνατότητα μάσκας λειτουργίας (operation mask). Αυτά είχαν περιορισμένη χρήση εκείνη την εποχή στο FFmpeg λόγω της μείωσης της συχνότητας της CPU (downscaling) όταν χρησιμοποιούνταν νέες εντολές. Πλήρης αναδιάταξη (shuffle/permute) 512-bit με την vpermb.
* AVX512ICL - Κυκλοφόρησε το 2019, όχι πλέον μείωση της συχνότητας του ρολογιού.
* AVX10 - Προσεχώς

Αξίζει να σημειωθεί ότι τα σύνολα εντολών μπορούν να αφαιρεθούν καθώς και να προστεθούν στους επεξεργαστές (CPU). Για παράδειγμα, το AVX512 [αφαιρέθηκε](https://www.igorslab.de/en/intel-deactivated-avx-512-on-alder-lake-but-fully-questionable-interpretation-of-efficiency-news-editorial/), με αμφιλεγόμενο τρόπο, στους επεξεργαστές Intel 12ης Γενιάς. Γι' αυτόν τον λόγο, το FFmpeg κάνει ανίχνευση CPU κατά το χρόνο εκτέλεσης (runtime). Το FFmpeg ανιχνεύει τις δυνατότητες του επεξεργαστή στον οποίο εκτελείται.

Όπως είδατε στην εργασία, οι δείκτες συναρτήσεων (function pointers) είναι από προεπιλογή C και αντικαθίστανται με μια συγκεκριμένη παραλλαγή συνόλου εντολών. Αυτό σημαίνει ότι η ανίχνευση γίνεται μία φορά και δεν χρειάζεται να ξαναγίνει ποτέ. Αυτό έρχεται σε αντίθεση με πολλές ιδιόκτητες εφαρμογές οι οποίες κωδικοποιούν ένα συγκεκριμένο σύνολο εντολών, καθιστώντας έναν απόλυτα λειτουργικό υπολογιστή παρωχημένο. Αυτό επιτρέπει επίσης την ενεργοποίηση/απενεργοποίηση βελτιστοποιημένων συναρτήσεων κατά το χρόνο εκτέλεσης. Αυτό είναι ένα από τα μεγάλα οφέλη του ανοιχτού κώδικα.

Προγράμματα όπως το FFmpeg χρησιμοποιούνται σε δισεκατομμύρια συσκευές σε όλο τον κόσμο, μερικές από τις οποίες μπορεί να είναι πολύ παλιές. Το FFmpeg τεχνικά υποστηρίζει μηχανήματα που υποστηρίζουν μόνο SSE, τα οποία είναι 25 ετών! Ευτυχώς, το x86inc.asm είναι σε θέση να σας πει εάν χρησιμοποιείτε μια εντολή που δεν είναι διαθέσιμη σε ένα συγκεκριμένο σύνολο εντολών.

Για να σας δώσουμε μια ιδέα για τις πραγματικές δυνατότητες, ακολουθεί η διαθεσιμότητα του συνόλου εντολών από την [Έρευνα του Steam](https://store.steampowered.com/hwsurvey/Steam-Hardware-Software-Survey-Welcome-to-Steam) από τον Νοέμβριο του 2024 (αυτό είναι προφανώς μεροληπτικό προς τους παίκτες):

| Σύνολο Εντολών | Διαθεσιμότητα |
| :---- | :---- |
| SSE2 | 100% |
| SSE3 | 100% |
| SSSE3 | 99.86% |
| SSE4.1 | 99.80% |
| AVX | 97.39% |
| AVX2 | 94.44% |
| AVX512 (Το Steam δεν διαχωρίζει μεταξύ AVX512 και AVX512ICL) | 14.09% |

Για μια εφαρμογή όπως το FFmpeg με δισεκατομμύρια χρήστες, ακόμη και το 0,1% είναι ένας πολύ μεγάλος αριθμός χρηστών και αναφορών σφαλμάτων εάν κάτι πάει στραβά. Το FFmpeg διαθέτει εκτεταμένη υποδομή δοκιμών για τον έλεγχο των παραλλαγών CPU/ΛΣ/Μεταγλωττιστή στη [σουίτα δοκιμών FATE](https://fate.ffmpeg.org/?query=subarch:x86_64%2F%2F). Κάθε commit εκτελείται σε εκατοντάδες μηχανήματα για να διασφαλιστεί ότι τίποτα δεν χαλάει.

Η Intel παρέχει ένα λεπτομερές εγχειρίδιο συνόλου εντολών εδώ:
[https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)

Μπορεί να είναι δυσκίνητο να ψάχνετε σε ένα PDF, οπότε υπάρχει μια ανεπίσημη εναλλακτική λύση που βασίζεται στο διαδίκτυο εδώ: [https://www.felixcloutier.com/x86/](https://www.felixcloutier.com/x86/)

Υπάρχει επίσης μια οπτική αναπαράσταση των εντολών SIMD διαθέσιμη εδώ: [https://www.officedaytime.com/simd512e/](https://www.officedaytime.com/simd512e/)

Μέρος της πρόκλησης της assembly x86 είναι η εύρεση της σωστής εντολής για τις ανάγκες σας. Σε ορισμένες περιπτώσεις, οι εντολές μπορούν να χρησιμοποιηθούν με τρόπο που δεν προορίζονταν αρχικά.

**Τεχνάσματα με τις μετατοπίσεις δεικτών (Pointer offset trickery)**

Ας επιστρέψουμε στην αρχική μας συνάρτηση από το Μάθημα 1, αλλά ας προσθέσουμε ένα όρισμα width (πλάτος) στη συνάρτηση C.

Χρησιμοποιούμε ptrdiff_t για τη μεταβλητή width αντί για int για να διασφαλίσουμε ότι τα ανώτερα 32-bit του 64-bit ορίσματος είναι μηδέν. Αν περνούσαμε απευθείας ένα int width στην υπογραφή της συνάρτησης και στη συνέχεια προσπαθούσαμε να το χρησιμοποιήσουμε ως τετραπλή λέξη (quadword) για αριθμητική δεικτών (δηλαδή χρησιμοποιώντας το `widthq`), τα ανώτερα 32-bit του καταχωρητή θα μπορούσαν να γεμίσουν με αυθαίρετες τιμές. Θα μπορούσαμε να το διορθώσουμε αυτό κάνοντας επέκταση προσήμου (sign extending) στο width με την εντολή `movsxd` (δείτε επίσης τη μακροεντολή `movsxdifnidn` στο x86inc.asm), αλλά αυτός είναι ένας ευκολότερος τρόπος.

Η παρακάτω συνάρτηση περιέχει το τέχνασμα με τη μετατόπιση του δείκτη:

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

Ας το δούμε βήμα προς βήμα καθώς μπορεί να προκαλέσει σύγχυση:

```assembly
   add srcq, widthq
   add src2q, widthq
   neg widthq
```

Το πλάτος (width) προστίθεται σε κάθε δείκτη έτσι ώστε κάθε δείκτης να δείχνει πλέον στο τέλος του buffer που πρόκειται να επεξεργαστεί. Στη συνέχεια, το πλάτος αναιρείται (negated).

```assembly
    movu  m0, [srcq+widthq]
    movu  m1, [src2q+widthq]
```

Οι φορτώσεις (loads) γίνονται στη συνέχεια με το widthq να είναι αρνητικό. Έτσι, στην πρώτη επανάληψη, το [srcq+widthq] δείχνει στην αρχική διεύθυνση του srcq, δηλαδή δείχνει πίσω στην αρχή του buffer.

```assembly
    add   widthq, mmsize
    jl .loop
```

Το mmsize προστίθεται στο αρνητικό widthq φέρνοντάς το πιο κοντά στο μηδέν. Η συνθήκη του βρόχου είναι τώρα jl (άλμα αν είναι μικρότερο από το μηδέν). Αυτό το τέχνασμα σημαίνει ότι το widthq χρησιμοποιείται ως μετατόπιση δείκτη **και** ως μετρητής βρόχου ταυτόχρονα, εξοικονομώντας μια εντολή cmp. Επιτρέπει επίσης τη χρήση της μετατόπισης του δείκτη σε πολλαπλές φορτώσεις και αποθηκεύσεις, καθώς και τη χρήση πολλαπλασίων των μετατοπίσεων του δείκτη εάν χρειαστεί (να το θυμάστε αυτό για την εργασία).

**Ευθυγράμμιση (Alignment)**

Σε όλα τα παραδείγματά μας χρησιμοποιούσαμε την εντολή movu για να αποφύγουμε το θέμα της ευθυγράμμισης. Πολλοί επεξεργαστές (CPU) μπορούν να φορτώσουν και να αποθηκεύσουν δεδομένα ταχύτερα εάν τα δεδομένα είναι ευθυγραμμισμένα, δηλαδή εάν η διεύθυνση μνήμης είναι διαιρετή με το μέγεθος του καταχωρητή SIMD. Όπου είναι δυνατόν, προσπαθούμε να χρησιμοποιούμε ευθυγραμμισμένες φορτώσεις και αποθηκεύσεις στο FFmpeg χρησιμοποιώντας την εντολή mova.

Στο FFmpeg, η av_malloc είναι σε θέση να παρέχει ευθυγραμμισμένη μνήμη στο σωρό (heap) και η οδηγία προεπεξεργαστή C DECLARE_ALIGNED μπορεί να παρέχει ευθυγραμμισμένη μνήμη στη στοίβα (stack). Εάν η mova χρησιμοποιηθεί με μια μη ευθυγραμμισμένη διεύθυνση, θα προκαλέσει σφάλμα κατάτμησης (segmentation fault) και η εφαρμογή θα καταρρεύσει. Είναι επίσης σημαντικό να είστε βέβαιοι ότι η τιμή ευθυγράμμισης αντιστοιχεί στο μέγεθος του καταχωρητή SIMD, δηλαδή 16 με xmm, 32 για ymm και 64 για zmm.

Δείτε πώς μπορείτε να ευθυγραμμίσετε την αρχή του τμήματος RODATA σε 64-byte:

```assembly
SECTION_RODATA 64
```

Σημειώστε ότι αυτό ευθυγραμμίζει απλώς την αρχή του RODATA. Μπορεί να χρειαστούν bytes συμπλήρωσης (padding bytes) για να διασφαλιστεί ότι η επόμενη ετικέτα παραμένει σε ένα όριο 64-byte.

**Επέκταση εύρους (Range expansion)**

Ένα άλλο θέμα που έχουμε αποφύγει μέχρι τώρα είναι η υπερχείλιση (overflowing). Αυτό συμβαίνει, για παράδειγμα, όταν η τιμή ενός byte ξεπερνά το 255 μετά από μια πράξη όπως η πρόσθεση ή ο πολλαπλασιασμός. Μπορεί να θέλουμε να εκτελέσουμε μια πράξη όπου χρειαζόμαστε μια ενδιάμεση τιμή μεγαλύτερη από ένα byte (π.χ. λέξεις - words), ή ενδεχομένως να θέλουμε να αφήσουμε τα δεδομένα σε αυτό το μεγαλύτερο ενδιάμεσο μέγεθος.

Για τα μη προσημασμένα bytes (unsigned bytes), εδώ είναι που μπαίνουν στο παιχνίδι οι εντολές punpcklbw (packed unpack low bytes to words - συσκευασμένη αποσυσκευασία χαμηλών bytes σε λέξεις) και punpckhbw (packed unpack high bytes to words - συσκευασμένη αποσυσκευασία υψηλών bytes σε λέξεις).

Ας δούμε πώς λειτουργεί η punpcklbw. Η σύνταξη για την έκδοση SSE2 από το Εγχειρίδιο της Intel είναι η εξής:

| PUNPCKLBW xmm1, xmm2/m128 |
| :---- |

Αυτό σημαίνει ότι η πηγή της (δεξιά πλευρά) μπορεί να είναι ένας καταχωρητής xmm ή μια διεύθυνση μνήμης (m128 σημαίνει μια διεύθυνση μνήμης με την τυπική σύνταξη [base + scale*index + disp]) και ο προορισμός ένας καταχωρητής xmm.

Η ιστοσελίδα officedaytime.com που αναφέρθηκε παραπάνω έχει ένα καλό διάγραμμα που δείχνει τι συμβαίνει:

![What is this](image1.png)

Μπορείτε να δείτε ότι τα bytes είναι πλεγμένα (interleaved) από το κάτω μισό του κάθε καταχωρητή αντίστοιχα. Αλλά τι σχέση έχει αυτό με την επέκταση εύρους; Αν ο καταχωρητής πηγής (src) είναι εξ ολοκλήρου μηδενικά, αυτό πλέκει τα bytes του καταχωρητή προορισμού (dst) με μηδενικά. Αυτό είναι γνωστό ως *επέκταση με μηδενικά* (zero extension), καθώς τα bytes είναι χωρίς πρόσημο (unsigned). Η punpckhbw μπορεί να χρησιμοποιηθεί για να κάνει το ίδιο πράγμα για τα υψηλά bytes (high bytes).

Ακολουθεί ένα απόσπασμα κώδικα που δείχνει πώς γίνεται αυτό:

```assembly
pxor      m2, m2 ; μηδενισμός του m2

movu      m0, [srcq]
movu      m1, m0 ; δημιουργία αντιγράφου του m0 στο m1
punpcklbw m0, m2
punpckhbw m1, m2
```

Οι καταχωρητές ```m0``` και ```m1``` περιέχουν τώρα τα αρχικά bytes επεκταμένα με μηδενικά σε λέξεις (words). Στο επόμενο μάθημα θα δείτε πώς οι εντολές τριών τελεστέων (three-operand instructions) στο AVX καθιστούν περιττή τη δεύτερη movu.

**Επέκταση προσήμου (Sign extension)**

Τα προσημασμένα δεδομένα (signed data) είναι λίγο πιο περίπλοκα. Για να επεκτείνουμε το εύρος ενός προσημασμένου ακεραίου, πρέπει να χρησιμοποιήσουμε μια διαδικασία γνωστή ως [επέκταση προσήμου](https://en.wikipedia.org/wiki/Sign_extension). Αυτή συμπληρώνει τα πιο σημαντικά bits (MSBs) με το bit προσήμου. Για παράδειγμα: το -2 σε int8_t είναι 0b11111110. Για να το επεκτείνουμε με πρόσημο σε int16_t, το MSB του 1 επαναλαμβάνεται για να γίνει 0b1111111111111110.

Η εντολή ```pcmpgtb``` (packed compare greater than byte) μπορεί να χρησιμοποιηθεί για επέκταση προσήμου. Κάνοντας τη σύγκριση (0 > byte), όλα τα bits στο byte προορισμού τίθενται σε 1 εάν το byte είναι αρνητικό, διαφορετικά τα bits στο byte προορισμού τίθενται σε 0. Η punpckX μπορεί να χρησιμοποιηθεί όπως παραπάνω για την εκτέλεση της επέκτασης προσήμου. Εάν το byte είναι αρνητικό, το αντίστοιχο byte είναι 0b11111111 και διαφορετικά είναι 0x00000000. Η εναλλαγή (interleaving) της τιμής του byte με την έξοδο της pcmpgtb εκτελεί ως αποτέλεσμα μια επέκταση προσήμου σε λέξη (word).

```assembly
pxor      m2, m2 ; μηδενισμός του m2

movu      m0, [srcq]
movu      m1, m0 ; δημιουργία αντιγράφου του m0 στο m1

pcmpgtb   m2, m0
punpcklbw m0, m2
punpckhbw m1, m2
```

Όπως μπορείτε να δείτε, υπάρχει μια επιπλέον εντολή σε σύγκριση με την περίπτωση των μη προσημασμένων (unsigned).

**Συσκευασία (Packing)**

Οι εντολές packuswb (pack unsigned word to byte - συσκευασία μη προσημασμένης λέξης σε byte) και packsswb σας επιτρέπουν να μεταβείτε από λέξη (word) σε byte. Σας επιτρέπουν να πλέξετε (interleave) δύο καταχωρητές SIMD που περιέχουν λέξεις σε έναν καταχωρητή SIMD με bytes. Σημειώστε ότι εάν οι τιμές υπερβούν το εύρος του byte, θα κορεστούν (δηλαδή θα περιοριστούν στη μεγαλύτερη τιμή - clamped at the largest value).

**Αναδιατάξεις (Shuffles)**

Οι αναδιατάξεις, επίσης γνωστές ως μεταθέσεις (permutes), είναι αναμφισβήτητα η πιο σημαντική εντολή στην επεξεργασία βίντεο και η pshufb (packed shuffle bytes - συσκευασμένη αναδιάταξη bytes), διαθέσιμη στο SSSE3, είναι η πιο σημαντική παραλλαγή.

Για κάθε byte, το αντίστοιχο byte προέλευσης χρησιμοποιείται ως δείκτης του καταχωρητή προορισμού, εκτός όταν το MSB (πιο σημαντικό bit) είναι ενεργοποιημένο, οπότε το byte προορισμού μηδενίζεται. Είναι ανάλογο με τον ακόλουθο κώδικα C (αν και στο SIMD όλες οι 16 επαναλήψεις του βρόχου συμβαίνουν παράλληλα):

```c
for(int i = 0; i < 16; i++) {
    if(src[i] & 0x80)
        dst[i] = 0;
    else
        dst[i] = dst[src[i]]
}
```
Ακολουθεί ένα απλό παράδειγμα assembly:

```assembly
SECTION_DATA 64

shuffle_mask: db 4, 3, 1, 2, -1, 2, 3, 7, 5, 4, 3, 8, 12, 13, 15, -1

section .text

movu m0, [srcq]
movu m1, [shuffle_mask]
pshufb m0, m1 ; αναδιάταξη του m0 με βάση το m1
```

Σημειώστε ότι το -1 για ευκολία στην ανάγνωση χρησιμοποιείται ως δείκτης αναδιάταξης για να μηδενίσει το byte εξόδου: το -1 ως byte είναι το πεδίο bit 0b11111111 (συμπλήρωμα ως προς δύο), και επομένως το MSB (0x80) είναι ενεργοποιημένο.

[image1]: <data:image/png;base64,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>