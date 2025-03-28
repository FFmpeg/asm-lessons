**FFmpeg Assembly Language Leçon Deux**

Maintenant que vous avez écrit votre première fonction en assembleur, nous allons vous présenter les branchements (*branch*) et les boucles (*loop*).

Dans un premier temps, nous devons vous introduire les notions de **labels** et de **jumps** (sauts). Regardons l'exemple suivant :

```assembly
mov  r0q, 3
.loop:
    dec  r0q
    jmp .loop
```

L'instruction `jmp` déplace l'exécution du code après `.loop:`. `.loop:` est ce qu'on appelle un **label**. Le **point** (.) préfixant le label indique que ce label est un **label local**. Cela signifie que vous pouvez réutiliser le même nom de label dans plusieurs fonctions sans provoque de conflits.

Cette portion de code est un exemple de boucle infinie, nous l'améliorerons dans la suite des leçons pour avoir un exemple beaucoup plus réaliste.

Avant de faire une boucle réaliste, nous devons introduire le registre **FLAGS**. Nous n'allons pas rentrer dans les détails complexes des **FLAGS** (car les opérations sur les **GPR** sont principalement des échafaudages) mais sachez qu'il existe plusieurs *flags* comme le **Zero Flag** (**Z** ou **ZF**, pour *indicateur de zéro*), le **Sign Flag** (**SF**, pour *indicateur de signe*), et l'**Overflow Flag** (**OF**, pour *indicateur de débordement*) qui sont un ensemble d'opérations basé sur la sortie de la majorité des instructions (à l'exception des instructions de type `mov`) sur des données scalaires comme des opérations arithmétiques et des décalages (*shifts*).

Voici une exemple où le compteur `r0q` de la boucle décrémente jusqu'à zéro et où `jg` (*jump if greater than zero*) est utilisé comme condition d'arrêt. L'instruction `dec r0q` met à jour les **FLAGS** en fonction des valeurs de `r0q` après chaque décrémentation et vous *rebouclez* jusqu'à ce que `r0q` atteigne zéro, moment où la boucle s'arrête.

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

Les deux exemples sont quasiment équivalents à la portion en assembleur suivante (il n'y a pas de manière simple de trouver l'équivalent à cette boucle ```for```):

```assembly
xor r0q, r0q
.loop:
    inc r0q
    ; do something
    cmp r0q, 3
    jl  .loop ; jump if (r0q - 3) < 0, i.e (r0q < 3)
```

Plusieurs choses sont à examiner sur cette extrait de code. Dans un premier temps, ```xor r0q, r0q``` est une manière simple d'assigner un registre à zéro, ce qui est plus rapide sur certains système que ```mov r0q, 0```, parce qu'il n'y a aucun chargement de registres.
Cela peut aussi être utilisé avec des registres **SIMD** avec la ligne ```pxor m0, m0``` pour mettre à zéro un registre entier. La chose suivante à noter est l'utilisation de ```cmp```. ```cmp``` peut effectivement soustraier le second registre du premier (sans avoir à sauvegarder la valeur quelque part) et met jour les *FLAGS*, mais comme l'indique le commentaire, cela peut être lu avec l'instruction de saut ```jl``` (saut si inférieur à zéro) pour effctuer un sat si ```r0q < 3```.

Notez la présence d'une instruction supplémentaire (````cmp```) dans ce court extrait. Généralement, peu d'instructions équivaut à du code plus rapide, c'est pourquoi l'exemple précédent est préféré. Comme vous le verrez plus tard, il existe plusieurs astuces pour éviter des instructions supplémentaires et faire en sort que les *FLAGS* soient définis par une opération arithmétique ou une autre opération. Notez que nous n'écrivons pas de l'assembleur pour correspondre exactement aux boucles en C, nous écrivons des boucles en assembleur pour les rendre les plus rapide en assembleur.

Voici quelques mnémoniques de saut que vous finirez par utiliser (les registres *FLAGS* sont présentés là pour tout couvrir, mais vous n'aurez pas à connaître pour écrire des boucles) :

| Mnémonique | Signification | Description | FLAGS |
| :---- | :---- | :---- | :---- |
| JE/JZ | Jump if Equal/Zero | Saut si Egal / Zéro | ZF = 1 |
| JNE/JNZ | Jump if Not Equal/Not Zero | Saut si Non Egalité / Non Zéro |  ZF = 0 |
| JG/JNLE | Jump if Greater/Not Less or Equal (signed) | Saut si Supériorité / Non Infériorité ou Egalité (signé) |  ZF = 0 and SF = OF |
| JGE/JNL | Jump if Greater or Equal/Not Less (signed) | Saut si Supériorité ou Egalité / Non Infériorité (signé) | SF = OF |
| JL/JNGE | Jump if Less/Not Greater or Equal (signed) | Saut si Infériorité / Non Supériorité ou Egalité (signé) |  SF ≠ OF |
| JLE/JNG | Jump if Less or Equal/Not Greater (signed) | Saut si Infériorité ou Egalité / Non Supériorité (signé) | ZF = 1 or SF ≠ OF |

**Constantes**

Plongeons dans quelques exemples sur l'utilisation des constantes:

```assembly
SECTION_RODATA

constants_1: db 1,2,3,4
constants_2: times 2 dw 4,3,2,1
```

* SECTION_RODATA indique qu'il s'agit d'une section en lecture seule. (C'est une macro car les différents formats de fichier de sortie utilisés par les systèmes d'exploitation les déclare différemment.)
* le label *constants_1* est défini comme étant un ```db``` (*declared bytes*), c'est équivalent à uint8_t constants_1[4] = {1, 2, 3, 4};
* constants_2 utilise la macro ```times 2``` pour répété la définition de mot (```dw``` pour *declared word*), c'est équivalent à uint16_t constants_2[8] = {4, 3, 2, 1, 4, 3, 2, 1};

Ces labels, que l'assembleur convertit en addresse mémoire, peuvent être utiliser pour les chargements (mais pas pour des stockages car ils sont en lecture seule). Quelques instructions acceptent une addresse mémoire comme opérande, ce qui permet de les utiliser sans chargement explicite dans un registre (il y a des avantages et des inconvénients à ceci).

**Offsets (déplacements)**

Les offsets (*déplacements*) sont les distances (en bytes) entre deux éléments consécutifs en mémoire. L'offset est déterminé par la **taille de chaque élément** dans la structure de donnée.

Maintenant que nous sommes capables d'écrire des boucles, il est temps de récupérer des données. Mais il existe des différences par rapport au C. Regardons la boucle suivante en C:

```c
uint32_t data[3];
int i;
for(i = 0; i < 3; i++) {
    data[i];
}
```

L'offset de 4 vytes entre chaque élément de données est pré-calculé par le compilateur. Mais en l'écrivant à la main en assembleur, vous devrez calculer cet offset vous-même.

Regardons à la syntaxe du calcul des addresses mémoires. Cela s'applique à tout type d'addresses mémoires:

```assembly
[base + scale*index + disp]
```

* base - Il s'agit d'un GPR (généralement un pointeur provenant d'un argument d'une fonction en C).
* scale - Entier valant 1, 2, 4, 8. La valeur par défaut est 1.
* index - un registre GPR (le compteur d'une boucle).
* disp - Entier (stocké jusqu'à au plus 32 bits). disp est un cécalage dans le données.

x86asm fournit la constante **mmsize**, qui vous indique la taille du registre SIMD avec leque vous travaillez.

Voici un simple (et pas très logique) exemple d'illustratio du chargement avec des déplacements différents:

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

Notez comment dans  ```movu m1, [srcq+2*r1q+3+mmsize]```  l'assemble pré-calculera le déplacement correct à utiliser. Dans la prochaine leçon, nous vous présenterons une astuce pour éviter d'avoir ```add``` et ```dec``` dans la boucle, en les remplacements par un unique ```add```.

**LEA**

Maintenant que vous maitrisez les offsets, vous êtes capable d'utiliser l'instruction ```lea``` (_**L**oad **E**ffective **A**ddress*_). Avec une seule instruction, vous êtes capable de faire une mumtiplication et une addition, ce qui est alors plus rapide que l'utilisation de plusieurs instructions. Il y a, bien sûr, des limitations sur les données que vous pouvez multiplier et ajouter mais cela n'empêche pas ```lea``` d'être une instruction puissante.

```assembly
lea r0q, [base + scale*index + disp]
```

Contrairement à son nom, **LEA** peut être utilisée autant pour des opérations arithmétiques que pour des calculs d'addresses mémoires. Vous pouvez faire quelque chose d'aussi compliqué que:

```assembly
lea r0q, [r1q + 8*r2q + 5]
```

Il est important de noter que cela n'affecte par le contenu de ````r1q``` et ```r2q```. Cela n'impacte pas non plus les registres *FLAGS* (par conséquent, vous ne pouvez pas effectuer de saut sur la sortie). L'utilisation des **LEA** évite toutes ces instructions et les régistres temporaires (ce code n'a pas d'équivalent car ```add``` modifie les *FLAGS*):

```assembly
movq r0q, r1q
movq r3q, r2q
sal  r3q, 3 ; shift arithmetic left 3 = * 8
add  r3q, 5
add  r0q, r3q
```

Vous verrez l'utilisation de ```lea``` dans l'utilisation de beaucoup d'addresses avant des boulces ou pour faire des calculs comme le précédent. Notez bien, que vous ne pouvez pas faire tous types de multiplications et d'additions, mais les multiplications par 1, 2, 4 ou 8 et les additions d'un décalage fixe sont des opérations courantes.

Dans l'exercice, vous aurez à charger une constante et en addiioner les valeurs à un vecteur **SIMD** dans une boucle.

[Leçon suivante](../lesson_03/index.fr.md)
