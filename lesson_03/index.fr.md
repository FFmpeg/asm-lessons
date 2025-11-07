**FFmpeg Assembly Language Leçon Trois**

Expliquons un peu plus de jargon et lisez bien cette petite leçon d'histoire.

**Jeux d'instructions**

Vous avez sûrement prêté attention au fait que dans la leçon précédente, nous avons parlé de **SSE2** qui fait partie du jeu d'instructions **SIMD**. Quand une nouvelle génération de CPU sort sur le marché, elle peut être accompagnée de nouvelles instructions et quelquefois de registres de tailles plus grandes. L'histoire du jeu d'instruction x86 est tellement complexe qu'il en existe une version simplifiée (avec plusieurs sous-histoires dans celle-ci) :

  * MMX - Lancée en 1997, la première version **SIMD** chez Intel Processors, registres de 64 bits, version historique.
  * SSE (Streaming SIMD Extensions) - Lancée en 1999, registres de 128 bits.
  * SSE2 - Lancée en 2000, plusieurs nouvelles instructions.
  * SSE3 - Lancée en 2004, premières instructions *horizontales*.
  * SSSE3 (Supplemental SSE3) - Lancée en 2006, ajout de nouvelles instructions dont la plus importante `pshufb`, sans doute l'instruction la plus importante pour le traitement vidéo.
  * SSE4 - Lancée en 2008, ajout de nombreuses nouvelles instructions, notamment les minimums et maximums en mode vectorisé.
  * AVX - Lancée en 2011, ajout de registres à 256 bits (uniquement pour les flottants) et d'une nouvelle syntaxe à trois opérandes.
  * AVX2 - Lancée en 2013, ajout de registres à 256 bits pour les instructions entières.
  * AVX512 - Lancée en 2017, registres à 512 bits, nouvelle instruction de masquage. Leur utilisation dans FFmpeg était très limitée en raison de la diminution de la fréquence du CPU quand de nouvelles instructions étaient utilisées. Permutation complète de 512 bits avec `vpermb`.
  * AVX512ICL - Lancé en 2019, suppression de la réduction de fréquence du processeur.
  * AVX10 - À venir.

Il est important de noter que des jeux d'instructions peuvent être supprimés ou ajoutés d'un CPU à l'autre. Par exemple, **AVX512** a été [supprimé](https://www.igorslab.de/en/intel-deactivated-avx-512-on-alder-lake-but-fully-questionable-interpretation-of-efficiency-news-editorial/), de manière controversée, dans la 12ème génération de CPU Intel. C'est pour cette raison que FFmpeg fait de la détection du CPU à l'exécution. FFmpeg détecte les capacités du processeur sur lequel il s'exécute.

Comme vous l'avez vu dans l'exercice, les pointeurs de fonctions sont par défaut en **C** et sont remplacés par un jeu d'instruction particulier. Cela signifie que la détection est réalisée une fois et ne sera plus jamais requise. Cela contraste avec beaucoup d'applications propriétaires qui programment en dur dans leur code un jeu d'instruction, rendant obsolètes des ordinateurs parfaitement fonctionnels. Cela permet aussi d'activer ou de désactiver des fonctions optimisées à l'exécution. C'est l'un des plus grands avantages de l'open source.

Des programmes comme FFmpeg sont utilisés par des milliards d'appareils autour du monde, certains d'entre eux sont peut-être très âgés. Techniquement, FFmpeg est compatible avec des machines possédant uniquement le jeu d'instruction **SSE**, machines qui datent d'il y a 25 ans.
Heureusement, **x86inc.ams** est capable de vous indiquer si une instruction n'est pas disponible dans un jeu d'instruction particulier.

Pour vous donner une idée des compatibilités, voici la disponibilité des jeux d'instructions selon le [Steam Survey](https://store.steampowered.com/hwsurvey/Steam-Hardware-Software-Survey-Welcome-to-Steam) en novembre 2024 (évidemment biaisé en faveur des joueurs) :

| Jeu d'instruction | Disponibilité |
| :---- | :---- |
| SSE2 | 100% |
| SSE3 | 100% |
| SSSE3 | 99.86% |
| SSE4.1 | 99.80% |
| AVX | 97.39% |
| AVX2 | 94.44% |
| AVX512 (AVX512 et AVX512ICL confondus) | 14.09% |

Pour une application comme FFmpeg avec des milliards d'utilisateurs, même 0.1% d'entre eux représente un grand nombre d'utilisateurs et de rapports de bug en cas de problème. FFmpeg possède une grande infrastructure de tests pour tester chaque combinaison de CPU/OS/compilateurs présentée sur [FATE testsuite](https://fate.ffmpeg.org/?query=subarch:x86_64%2F%2F). Chaque simple *commit* est exécuté sur des centaines de machines pour être certain que rien ne casse.

Intel fournit un manuel de jeu d'instructions détaillé ici : [https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)

Cela peut être laborieux de chercher sur un PDF, une alternative en ligne est présente ici : [https://www.felixcloutier.com/x86/](https://www.felixcloutier.com/x86/)

Il y a aussi une représentation visuelle des instructions SIMD disponible ici : [https://www.officedaytime.com/simd512e/](https://www.officedaytime.com/simd512e/)

Une partie du défi de l'utilisation de l'assembleur x86 est de trouver la bonne instruction dont vous avez besoin. Dans certains cas, des instructions peuvent être utilisées dans des cas de figure pour lesquelles elles n'ont pas été faites originellement.

**Astuces sur les décalages de pointeurs**

Revenons à notre fonction originale de la Leçon 1, mais ajoutons un argument de largeur à la fonction en C.

Nous utilisons `ptrdiff_t` pour la variable de largeur au lieu de `int` afin de garantir que les 32 bits supérieurs de l’argument 64 bits sont mis à zéro. Si nous passions directement une largeur en `int` dans la signature de la fonction, puis tentions de l’utiliser comme un `quad` pour l’arithmétique des pointeurs (c’est-à-dire en utilisant `widthq`), les 32 bits supérieurs du registre pourraient contenir des valeurs arbitraires. Nous pourrions corriger cela en étendant le signe de `width` avec `movsxd` (voir aussi la macro `movsxdifnidn` dans **x86inc.asm)**, mais cette méthode est plus simple.

La fonction ci-dessous contient l’astuce des décalages de pointeur :

```assembly
;static void add_values(const uint8_t *src, const uint8_t *src2, ptrdiff_t width)
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

Passons en revue cela étape par étape, car cela peut être déroutant :

```assembly
   add srcq, widthq
   add src2q, widthq
   neg widthq
```

La largeur `widthq` est ajoutée à chaque pointeur de sorte que chaque pointeur pointe maintenant vers la fin du tampon à traiter. Le signe de `widthq` est ensuite inversé.

```assembly
    movu  m0, [srcq+widthq]
    movu  m1, [src2q+widthq]
```

Les chargements sont ensuite effectués avec `widthq` étant négatif. Ainsi, lors de la première itération, `[srcq+widthq]` pointe vers l'adresse originale de `srcq`, c'est-à-dire qu'il pointe vers le début du tampon.

```assembly
    add   widthq, mmsize
    jl .loop
```

`mmsize` est ajouté à `widthq` négatif, le rapprochant ainsi de zéro. La condition de boucle est maintenant `jl` (*jump if less than zero*). Cette astuce permet à `widthq` d’être utilisé à la fois comme décalage de pointeur et comme compteur de boucle, économisant ainsi une instruction `cmp`. Elle permet également d’utiliser le décalage du pointeur dans plusieurs chargements et stockages, ainsi que d'utiliser des multiples des décalages de pointeur si nécessaire (retenez cela pour l'exercice).

**Alignement**

Dans tous nos exemples, nous avons utilisé `movu` pour éviter le sujet de l’alignement. De nombreux processeurs peuvent charger et stocker des données plus rapidement si celles-ci sont alignées, c’est-à-dire si l’adresse mémoire est divisible par la taille du registre SIMD. Lorsque c’est possible, nous essayons d’utiliser des chargements et des stockages alignés dans FFmpeg en utilisant `mova`.

Dans FFmpeg, `av_malloc` peut fournir une mémoire alignée sur le tas, et la directive de préprocesseur C `DECLARE_ALIGNED` peut fournir une mémoire alignée sur la pile. Si `mova` est utilisé avec une adresse non alignée, cela entraînera une faute de segmentation et l’application plantera. Il est également important de s’assurer que la valeur d’alignement correspond à la taille du registre SIMD, c’est-à-dire 16 pour `xmm`, 32 pour `ymm` et 64 pour `zmm`.

Voici comment aligner le début de la section RODATA sur 64 octets :

```assembly
SECTION_RODATA 64
```

Notez que cela aligne uniquement le début de la section RODATA. Des octets de remplissage peuvent être nécessaires pour s'assurer que le label suivant reste sur une limite de 64 octets.

**Décompactage**

Un autre sujet que nous avons évité jusqu'à présent est le débordement. Cela se produit, par exemple, lorsque la valeur d'un octet dépasse 255 après une opération comme l'addition ou la multiplication. Nous pouvons vouloir effectuer une opération où nous avons besoin d'une valeur intermédiaire plus grande qu'un octet (par exemple, des mots), ou potentiellement nous voulons laisser les données dans cette taille intermédiaire plus grande.

Pour les octets non signés, c'est là que les instructions `punpcklbw` (décompacter des octets en mots dans le bas d'un paquet) et `punpckhbw` (décompacter des octets en mots dans le haut d'un paquet) interviennent.

Voyons comment fonctionne `punpcklbw`. La syntaxe pour la version SSE2 dans le manuel Intel est la suivante :

| PUNPCKLBW xmm1, xmm2/m128 |
| :---- |

Cela signifie que sa source (côté droit) peut être un registre `xmm` ou une adresse mémoire (`m128` signifie une adresse mémoire avec la syntaxe standard **[base + échelle\*index + déplacement]**), et la destination est un registre `xmm`.

Le site web officedaytime.com ci-dessus a un bon diagramme montrant ce qui se passe :

![What is this](image1.png)

Vous pouvez voir que les octets sont entrelacés à partir de la moitié inférieure de chaque registre respectivement. Mais quel rapport cela a-t-il avec l'extension de plage ? Si le registre source est entièrement nul, cela entrelace les octets dans le registre de destination avec des zéros. C'est ce qu'on appelle l'*extension par zéro*, car les octets sont sans signe. `punpckhbw` peut être utilisé pour faire la même chose pour les octets supérieurs.

Voici un extrait montrant comment cela est fait :

```assembly
pxor      m2, m2 ; zero out m2

movu      m0, [srcq]
movu      m1, m0 ; make a copy of m0 in m1
punpcklbw m0, m2
punpckhbw m1, m2
```

`m0` et `m1` contiennent maintenant les octets d'origine étendus à zéro en mots. Dans la prochaine leçon, vous verrez comment les instructions à trois opérandes en AVX rendent le deuxième `movu` inutile.

**Extension de signe**

Les données signées sont un peu plus compliquées. Pour étendre la plage d'un entier signé, nous devons utiliser un processus connu sous le nom d'[extension de signe](https://en.wikipedia.org/wiki/Sign_extension). Cela consiste à ajouter des bits de signe dans les bits les plus significatifs. Par exemple : `-2` en `int8_t` est `0b11111110`. Pour l'étendre à `int16_t`, le bit de signe `1` est répété pour obtenir `0b1111111111111110`.

`pcmpgtb` (*packed compare greater than byte*) peut être utilisé pour l'extension de signe. En effectuant la comparaison (0 \> octet), tous les bits dans l'octet de destination sont définis sur 1 si l'octet est négatif, sinon les bits dans l'octet de destination sont définis sur 0. `punpckX` peut être utilisé comme ci-dessus pour effectuer l'extension de signe. Si l'octet est négatif, l'octet correspondant est `0b11111111` et sinon il est `0x00000000`. L'entrelacement de la valeur de l'octet avec la sortie de `pcmpgtb` effectue une extension de signe en mot.

```assembly
pxor      m2, m2 ; zero out m2

movu      m0, [srcq]
movu      m1, m0 ; make a copy of m0 in m1

pcmpgtb   m2, m0
punpcklbw m0, m2
punpckhbw m1, m2
```

Comme vous pouvez le voir, il y a une instruction supplémentaire par rapport au cas non signé.

**Packing**

`packuswb` (*pack unsigned word to byte*) et `packsswb` vous permettent de passer du mot au byte. Ils vous permettent d'entrelacer deux registres SIMD contenant des mots en un seul registre SIMD avec des bytes. Notez que si les valeurs dépassent la plage des bytes, elles seront saturées (c'est-à-dire *clampées* à la valeur maximale).

**Shuffles**

Les mélanges (*shuffles*), également appelés permutations, sont sans doute l'instruction la plus importante dans le traitement vidéo et `pshufb` (*packed shuffle bytes*), disponible dans SSSE3, est la variante la plus importante.

Pour chaque byte, le byte source correspondant est utilisé comme un index dans le registre de destination, sauf lorsque le bit de poids fort (MSB) est activé, le byte de destination est mis à zéro. C'est l'analogue du code C suivant (bien que dans SIMD, les 16 itérations de boucle se produisent en parallèle) :

```c
for(int i = 0; i < 16; i++) {
    if(src[i] & 0x80)
        dst[i] = 0;
    else
        dst[i] = dst[src[i]]
}
```

Voici un exemple simple en assembleur:

```assembly
SECTION_DATA 64

shuffle_mask: db 4, 3, 1, 2, -1, 2, 3, 7, 5, 4, 3, 8, 12, 13, 15, -1

section .text

movu m0, [srcq]
movu m1, [shuffle_mask]
pshufb m0, m1 ; shuffle m0 based on m1
```

Notez que -1, pour une lecture facile, est utilisé comme l'indice de mélange pour mettre à zéro le byte de sortie : `-1` en tant que byte est le champ binaire `0b11111111` (complément à deux), et ainsi le bit de poids fort (MSB) (`0x80`) est activé.
