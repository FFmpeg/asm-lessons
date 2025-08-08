**FFmpeg Assembly Language Leçon Deux**

Maintenant que vous avez écrit votre première fonction en assembleur, nous allons vous présenter les branchements (*branch*) et les boucles (*loop*).

Dans un premier temps, nous devons vous introduire les notions de **labels** et de **jumps** (sauts). Regardons l'exemple suivant :

```assembly
mov  r0q, 3
.loop:
    dec  r0q
    jmp .loop
```

L'instruction `jmp` déplace l'exécution du code après `.loop:`. `.loop:` est ce qu'on appelle un **label**. Le **point** (.) préfixant le label indique que ce label est un **label local**. Cela signifie que vous pouvez réutiliser le même nom de label dans plusieurs fonctions sans provoquer de conflits.

Cette portion de code est un exemple de boucle infinie, nous l'améliorerons dans la suite des leçons pour avoir un exemple beaucoup plus réaliste.

Avant de faire une boucle réaliste, nous devons introduire le registre **FLAGS**. Nous n'allons pas rentrer dans les détails complexes des **FLAGS** (car les opérations sur les **GPR** sont principalement des échafaudages) mais sachez qu'il existe plusieurs *flags* comme le **Zero Flag** (**Z** ou **ZF**, pour *indicateur de zéro*), le **Sign Flag** (**SF**, pour *indicateur de signe*), et l'**Overflow Flag** (**OF**, pour *indicateur de débordement*) qui sont mis à jour en fonction du résultat de la plupart des instructions (à l'exception des instructions de type `mov`) sur des données scalaires comme des opérations arithmétiques et des décalages (*shifts*).

Voici un exemple où le compteur `r0q` de la boucle décrémente jusqu'à zéro et où `jg` (*jump if greater than zero*) est utilisé comme condition d'arrêt. L'instruction `dec r0q` met à jour les **FLAGS** en fonction des valeurs de `r0q` après chaque décrémentation et la boucle se répète jusqu'à ce que `r0q` atteigne zéro, moment où la boucle s'arrête.

```assembly
mov  r0q, 3
.loop:
    ; do something
    dec  r0q
    jg  .loop ; jump if greater than zero
```

L'équivalent en **C** est le suivant :

```c
int i = 3;
do
{
   // do something
   i--;
} while(i > 0)
```

Ce code en **C** est un peu contre-intuitif. Normalement, une boucle en **C** s'écrit de cette manière :

```c
int i;
for(i = 0; i < 3; i++) {
    // do something
}
```

Les deux exemples sont quasiment équivalents à la portion en assembleur suivante (il n'y a pas de manière simple de trouver l'équivalent à cette boucle `for`) :

```assembly
xor r0q, r0q
.loop:
    inc r0q
    ; do something
    cmp r0q, 3
    jl  .loop ; jump if (r0q - 3) < 0, i.e (r0q < 3)
```

Plusieurs choses sont à examiner dans cet extrait de code. Dans un premier temps, `xor r0q, r0q` est une manière simple d'assigner un registre à zéro, ce qui est plus rapide sur certains systèmes que `mov r0q, 0`, car aucune valeur immédiate n'est chargée.
Cela peut aussi être utilisé avec des registres **SIMD** avec la ligne `pxor m0, m0` pour mettre à zéro un registre entier. La chose suivante à noter est l'utilisation de `cmp`. `cmp` peut effectivement soustraire le second registre du premier (sans avoir à sauvegarder la valeur quelque part) et met jour les *FLAGS*, mais comme l'indique le commentaire, cela peut être lu avec l'instruction de saut `jl` (saut si inférieur à zéro) pour effectuer un saut si `r0q < 3`.

Notez la présence d'une instruction supplémentaire (`cmp`) dans ce court extrait. Généralement, peu d'instructions équivaut à du code plus rapide, c'est pourquoi l'exemple précédent est préféré. Comme vous le verrez plus tard, il existe plusieurs astuces pour éviter des instructions supplémentaires et faire en sorte que les *FLAGS* soient définis par une opération arithmétique ou une autre opération. Notez que nous n'écrivons pas de l'assembleur pour correspondre exactement aux boucles en C, nous écrivons des boucles en assembleur pour les rendre les plus rapides en assembleur.

Voici quelques mnémoniques de saut que vous finirez par utiliser (les registres *FLAGS* sont présentés là pour tout couvrir, mais vous n'aurez pas à tout connaître pour écrire des boucles) :

| Mnémonique | Signification | Description | FLAGS |
| :---- | :---- | :---- | :---- |
| JE/JZ | **J**ump if **E**qual/**Z**ero | Saut si Egal / Zéro | ZF = 1 |
| JNE/JNZ | **J**ump if **N**ot **E**qual/**N**ot **Z**ero | Saut si Non Égal / Non Zéro |  ZF = 0 |
| JG/JNLE | **J**ump if **G**reater/**N**ot **L**ess or **E**qual (signed) | Saut si Supérieur / Non Inférieur ou Égal (signé) |  ZF = 0 et SF = OF |
| JGE/JNL | **J**ump if **G**reater or **E**qual/**N**ot **L**ess (signed) | Saut si Supérieur ou Égal / Non Inférieur (signé) | SF = OF |
| JL/JNGE | **J**ump if **L**ess/**N**ot **G**reater or **E**qual (signed) | Saut si Inférieur / Non Supérieur ou Égal (signé) |  SF ≠ OF |
| JLE/JNG | **J**ump if **L**ess or **E**qual/**N**ot **G**reater (signed) | Saut si Inférieur ou Égal / Non Supérieur (signé) | ZF = 1 ou SF ≠ OF |

**Constantes**

Plongeons dans quelques exemples sur l'utilisation des constantes :

```assembly
SECTION_RODATA

constants_1: db 1,2,3,4
constants_2: times 2 dw 4,3,2,1
```

* SECTION_RODATA indique qu'il s'agit d'une section en lecture seule. (C'est une macro car les différents formats de fichier de sortie utilisés par les systèmes d'exploitation les déclare différemment.)
* le label *constants_1* est défini comme étant un `db` (*declared bytes*), c'est équivalent à uint8_t constants_1[4] = {1, 2, 3, 4};
* constants_2 utilise la macro `times 2` pour répéter la définition de mot (`dw` pour *declared word*), c'est équivalent à uint16_t constants_2[8] = {4, 3, 2, 1, 4, 3, 2, 1};

Ces labels, que l'assembleur convertit en adresses mémoire, peuvent être utilisés pour les chargements (mais pas pour des stockages car ils sont en lecture seule). Quelques instructions acceptent une adresse mémoire comme opérande, ce qui permet de les utiliser sans chargement explicite dans un registre (il y a des avantages et des inconvénients à ceci).

**Offsets (déplacements)**

Les offsets (*déplacements*) sont les distances (en bytes) entre deux éléments consécutifs en mémoire. L'offset est déterminé par la **taille de chaque élément** dans la structure de données.

Maintenant que nous sommes capables d'écrire des boucles, il est temps de récupérer des données. Mais il existe des différences par rapport au C. Regardons la boucle suivante en C :

```c
uint32_t data[3];
int i;
for(i = 0; i < 3; i++) {
    data[i];
}
```

L'offset de 4 bytes entre chaque élément de données est pré-calculé par le compilateur. Mais en l'écrivant à la main en assembleur, vous devrez calculer cet offset vous-même.

Regardons la syntaxe du calcul d'adresses mémoire. Cela s'applique à tout type d'adresses mémoires :

```assembly
[base + scale*index + disp]
```

* base - Il s'agit d'un GPR (généralement un pointeur provenant d'un argument d'une fonction en C).
* scale - Entier valant 1, 2, 4, 8. La valeur par défaut est 1.
* index - un registre GPR (le compteur d'une boucle).
* disp - Entier (stocké jusqu'à au plus 32 bits). disp est un décalage dans les données.

x86asm fournit la constante **mmsize**, qui vous indique la taille du registre SIMD avec lequel vous travaillez.

Voici un simple (et pas très logique) exemple d'illustration du chargement avec des déplacements différents :

```assembly
;static void simple_loop(const uint8_t *src)
INIT_XMM sse2
cglobal simple_loop, 1, 2, 2, src
     movq r1q, 3
.loop:
     movu m0, [srcq]
     movu m1, [srcq+2*r1q+3+mmsize]

     ; do some things

     add srcq, mmsize
dec r1q
jg .loop

RET
```

Notez comment dans  `movu m1, [srcq+2*r1q+3+mmsize]`  l'assembleur pré-calculera le déplacement correct à utiliser. Dans la prochaine leçon, nous vous présenterons une astuce pour éviter d'avoir `add` et `dec` dans la boucle, en les remplaçant par un unique `add`.

**LEA**

Maintenant que vous maîtrisez les offsets, vous êtes capable d'utiliser l'instruction ```lea``` (_**L**oad **E**ffective **A**ddress_). Avec une seule instruction, vous êtes capable de faire une multiplication et une addition, ce qui est alors plus rapide que l'utilisation de plusieurs instructions. Il y a, bien sûr, des limitations sur les données que vous pouvez multiplier et ajouter mais cela n'empêche pas `lea` d'être une instruction puissante.

```assembly
lea r0q, [base + scale*index + disp]
```

Contrairement à son nom, **LEA** peut être utilisée autant pour des opérations arithmétiques que pour des calculs d'adresses mémoires. Vous pouvez faire quelque chose d'aussi compliqué que :

```assembly
lea r0q, [r1q + 8*r2q + 5]
```

Il est important de noter que cela n'affecte pas le contenu de `r1q` et `r2q`. Cela n'impacte pas non plus les registres *FLAGS* (par conséquent, vous ne pouvez pas effectuer de saut sur la sortie). L'utilisation des **LEA** évite toutes ces instructions et les régistres temporaires (ce code n'a pas d'équivalent car `add` modifie les *FLAGS*) :

```assembly
movq r0q, r1q
movq r3q, r2q
sal  r3q, 3 ; shift arithmetic left 3 = * 8
add  r3q, 5
add  r0q, r3q
```

Vous verrez `lea` utilisée pour manipuler de nombreuses adresses avant des boucles ou pour faire des calculs comme le précédent. Notez bien que vous ne pouvez pas faire tous types de multiplications et d'additions, mais les multiplications par 1, 2, 4 ou 8 et les additions d'un décalage fixe sont des opérations courantes.

Dans l'exercice, vous aurez à charger une constante et d'en additionner les valeurs à un vecteur **SIMD** dans une boucle.

[Leçon suivante](../lesson_03/index.fr.md)
