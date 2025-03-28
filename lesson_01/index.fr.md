**FFmpeg Assembly Language Leçon Un**

**Introduction**

Bienvenue dans la FFmpeg School of Assembly Language. Vous avez fait le premier pas dans le voyage le plus intéressant, le plus stimulant et le plus gratifiant de la programmation. Ces leçons vous donneront des bases sr la façon sur la manière l'assembleur est utilisée dns FFmpeg et vous ouvriront les yeux sur ce qui se passe réellement dans votre ordinateur...

**Connaissances requises**

* Connaissance en C, particulièrement les pointeurs. Si le C ne vous est pas familier, vous trouverez toutes les bases dans le livre [The C Programming Language](https://en.wikipedia.org/wiki/The_C_Programming_Language).
* Mathématiques de lycée (scalaire vs vecteur, addition, multiplication, etc...)

**Qu'est ce que l'assembleur?**

L'assembleur est un langage de programmation où le code que vous écrivez correponds direction à des instructions compréhensible par un CPU. L'assembleur lisible par l'Homme est, comme son nom l'indique, *assemblé* en données binaires, connu sous le nom de *langage machine*, que le CPU peut comprendre. Le code assembleur est souvent appelé "assembleur" ou "asm" en ambrégé.

La grand majorité de l'assembleur de FFmpeg est ce que l'on appelle *SIMD, Single Instruction Multiple Data (instruction unique, données multiples)*. SIMD est parfois désigné sous le terme de programmation vectorielle. Cela signifie qu'une instruction particulière opère sur plusieurs éléments de données simultanément. La plupart des langages de programmation traitent un seul élément de données à la fois, ce que l'on appelle la programmation scalaire.

Comme vous l'avez peut-être deviné, SIMD se prête bien au traitement d'images, des vidéos et de l'audio, qui contiennent une grande quantité de données organisées séquentiellement en mémoire. Des instructions spécialisées dans le processeur nous aideront à traiter ces données séquentielles.

Dans FFmpeg, vous verrez que les termes `fonction en assembleur`, `SIMD`, `vectorisation` sont utilisés de manière interchangeable. Ils désignent tous la même chose: écrire une fonction en assembleur à la main pour traiter plusieurs éléments de données en une seule fois. Certains projets peuvent aussi faire référence à des `noyaux en assembleur`.

Tout cela peut sembler compliqué, mais il est important de rappeler que des lycééns ont écrit de l'assembleur dans FFmpeg. Comme partout, l'apprentissage c'est 50% du jargon et 50% d'apprentissage réel.

**Pourquoi écrivons en assembleur ?**

Pour rendre le traitement multimédia rapide. Il est très courant d'avoir une vitesse de traitement au moins 10 fois plus rapide en écrivant en assembleur, ce qui est d'autant plus important quand nous voulons lire des vidéos en temps réel sans saccade. Cela permet aussi d'économiser de l'énergie et d'étendre la durée de vie des batteries. Il est important de souligner que les fonctions d'encodage et de décodage sont parmi les fonctions les plus utilisés au monde, aussi bien par les utilisateurs finaux que par les multi-nationales dans leurs data-centers. Donc, même une petit amélioration apporte beaucoup.

Vous verrez souvent, en ligne, des gens utiliser des *fonctions intrinsèques*, des fonctions ressemblants à du C mais qui sont en fait des instructions en assembleur utilisées pour pemettre un développement plus rapide. Dans FFmpeg, nous n'utilisons pas ce genre de fonctions, nous écrivons tout le code en assembleur à la main. C'est un point de discorde, mais les fonctions intrinsèques sont environ 10 à 15% plus lente que l'équivalent en assembleur écrit à la main (les partisans de ces fonctions ne seraient pas d'accord), tout dépend du compilateur. Pour FFmpeg, chaque amélioration compte, c'est pourquoi nous écrivons tout le code directement en assembleur. Un argument en notre faveur est l'utilisation de la `[notation hongroise](https://fr.wikipedia.org/wiki/Notation_hongroise)` dans les fonctions intrinsèques qui compliquent leur lecture.

Aussi, vous verrez des référence à de l'*assembleur en ligne* (ou *assembleur inline*), c'est-à-dire n'utilisant pas de fonctions intrinsèques, à quelques endroits dans FFmpeg pour des raisons historiques, ou dans des projets comme le noyau Linux dans des scénarios d'utilisations très spécifiques. Ici, le code assembleur n'est pas dans un fichier séparé mais écrit directement dans des fichiers avec du code en C. Le point de vue majoritaire dans des projets comme FFmpeg est que ce code est difficile à lire, pas largement supporté par les compilateurs et difficile à maintenir.

Enfin, vous verrez beaucou d'experts auto-proclamés sur Internet disant que rien de tout cela est nécessaire et que le compilateur peut effectuer cette `vectorisation` pour vous. Dans une optique d'apprentissage, ignorez-les : des tests récents comme ceux présents sur le [projet dav1d](https://www.videolan.org/projects/dav1d.html) ont montrés un gain de vitesse d'environ x2 grâce à cette vectorisation automatique, tandis que pour des versions écrites à la main, ce gain montait à x8.

**Les variétés d'assembleurs**

Ces leçons se concentreront sur l'assembleur x86 64 bits. Aussi connu sous le nom d'amd64, bien qu'il continue de fonctionner sur les CPUs Intel. Il existe autant de types d'assembleurs que de CPUs comme ceux pour ARM ou RISC-V avec potentiellement des mises à jour de ces cours pour les inclure.

Il existe deux types de syntaxes pour l'assembleur x86 que vous trouverez en ligne : AT&T et Intel. La première est plus ancienne et plus difficile à lire comparé à la seconde. Nous nous intéresserons à cette dernière.

**Matériels supportés**

Vous serez peut-être surpris d'entendre que des livres ou des ressources en ligne comme Stack Overflow ne sont pas d'une grande aide comme références. Particulièrement à cause de notre choix d'utiliser de l'assembleur écrit à la main avec la syntaxe Intel. Mais aussi parce que beaucoup de ces ressources en ligne se concentrent sur de la programmation de systèmes d'exploitation ou de la programmation pour du hardware, n'utilisant pas du code SIMD. L'assembleur de FFmpeg est majoritairement axé autout du traitement d'images haute performance, et comme vous le verrez, avec une approche particulière de la programmation en assembleur. Ceci étant dit, il est facile de comprendre d'autres cas d'utilisation de l'assembleur une fois ces leçons terminées.

Beaucoup de livres détaillent beaucoup différentes architectures d'ordinateur avant de détailler l'assembleur. De notre point de vue, c'est très bien si c'est ce que vous voulez apprendre, mais cela revient à vouloir étudier les moteurs avant d'apprendre à conduire une voiture.

Une fois ceci dit, dans les parties suivantes, les diagrammes du livre `The Art of 64-bit assembly` montrant les instructions SIMD et leur comportement sous forme visuelle vous seront très utiles : [https://artofasm.randallhyde.com/](https://artofasm.randallhyde.com/)

Un serveur Discord est disponible pour répondre à vos questions:
[https://discord.com/invite/Ks5MhUhqfB](https://discord.com/invite/Ks5MhUhqfB)

**Les registres**  

Les registres sont des zones du CPU où les données peuvent être traitées. Les CPUs n'interviennent pas directement sur la mémoire, les données sont plutôt chargés dans des registres, traitées, puis écrites à nouveau en mémoire. En assembleur, de manière générale, vous ne pouvez pas copier directement les données d'un emplacement mémoire à un autre endroit sans d'abord passer ces données dans un registre.

**Registres à usage général**

Le premier type de registre que nous allons rencontrer est connu sous le nom de Registre à Usage Générale (GPR). Les GPR sont appelés ainsi car ils peuvent contenir soit des données, une valeur allant jusqu'à 64-bits, soit une adresse mémoire (un pointeur). Une valeur dans un GPR peut être traitée par des opérations telles que l'addition, la multiplication, le décalage, etc...

Dans la plupart des livres sur l'assembleur, de nombreux chapitres entiers sont concsacrés aux subtilités des GPR, leur histoire, etc... Car les GPR ont joués un rôle important dans la programmation de systèmes d'exploitation, le reverse engineering, etc... Dans l'assembleur écrit pour FFmpeg, les GPR sont considérés comme des échafaudages et la plupart du temps, leur compléxités ne sont pas nécessaires et sont abstraites.

**Registres vectoriels**  
Les registrs vectoriels (SIMD), comme leur nom le suggère, contiennent plusieurs éléments de données. Il existe différents types de registres vectoriels:

* registres `mm` : des registres `MMX`, de taille 64-bits, historique et peu utilisés de nos jours
* registres `xmm` : des registres `XMM`, de taille 128 bits, largement disponible
* registres `ymm` : des registres `YMM`, de tailles 256 bits, avec quelques complications lors de leur utilisation
* registres `zmm` : des registres `ZMM`, de tailles 512 bits, très peu disponible

La plupart des calculs de compression et de décompression vidéo sont basés sur des entiers, nous nous y limiterons. Voici un exemple d'un registre `XMM` de 16 octets:

| a | b | c | d | e | f | g | h | i | j | k | l | m | n | o | p |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Cela pourrait aussi être huit mots (entiers de 16 bits):
| a | b | c | d | e | f | g | h |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Ou quatre double mots (entiers de 32 bits):
| a | b | c | d |
| :---- | :---- | :---- | :---- |


Ou deux quadruples mots (entiers de 64 bits):

| a | b |
| :---- | :---- |


Pour récapituler :
* **b**ytes - données de 8 bits
* **w**ords - données de 16 bits
* **d**oublewords - données de 32 bits 
* **q**uadwords - données de 64 bits
* **d**ouble **q**uadwords - données de 128 bits.

Les caractères en gras seront important dans la suite.

**Header x86inc.asm**  

Dans beaucoup d'exemples, nous incluons le fichier x86inc.asm. x86inc.asm est une couche d'abstraction légère utilisée dans FFmpeg, x264 et dav1d pour faciliter la vie des développeurs. Elle aide de nombreuses manières, mais l'une des choses utiles qu'elle permet est entre autre d'étiquetter les GPR `r0`, `r1` et `r2`. Vous n'aurez pas à vous souvenir des noms des registres. Comme mentionné précédemment, les GPR sont généralement juste des échafaudages, donc cela rend les choses beaucoup plus simples.

**Un extrait simple d'assembleur scalaire**

Regardons un extrait simple (et totalement artificiel) de code assembleur scalaire (code assembleur qui opère sur des éléments de données individuels, un à la fois, dans chaque instruction) pour voir ce qui se passe :

```assembly
mov  r0q, 3  
inc  r0q  
dec  r0q  
imul r0q, 5
```

À la première ligne, la *valeur immédiate* 3 (une valeur stockée directement dans le code assembleur lui-même, contrairement à une valeur récupérée depuis la mémoire) est stockée dans le registre `r0` sous forme de quadword. Notez qu'en syntaxe Intel, l'opérande source (la valeur ou l'emplacement fournissant les données, situé à droite) est transféré vers l'opérande de destination (l'emplacement recevant les données, situé à gauche), un peu comme le comportement de *memcpy*. Vous pouvez aussi le lire comme `r0q = 3`, puisque l'ordre est le même. Le suffixe `q` de `r0` désigne le registre comme étant utilisé comme quadword. `inc` incrémente la valeur pour que `r0q` contienne 4, `dec` décrémente la valeur pour revenir à 3. `imul` multiplie la valeur par 5. À la fin, `r0q` contient donc 15.

Notez que les instructions lisibles par l'humain, comme mov et inc, qui sont assemblées en code machine par l'assembleur, sont appelées *mnémoniques*. Vous pouvez voir en ligne et dans les livres des mnémoniques représentées avec des lettres majuscules comme `MOV` et `INC`, mais elles sont les mêmes que les versions en minuscules. Dans FFmpeg, nous utilisons des mnémoniques en minuscules et réservons les majuscules pour les macros.

**Comprendre une fonction vectorielle de base**

Voici notre première fonction SIMD :

```assembly
%include "x86inc.asm"

SECTION .text

;static void add_values(const uint8_t *src, const uint8_t *src2)  
INIT_XMM sse2  
cglobal add_values, 2, 2, 2, src, src2   
    movu  m0, [srcq]  
    movu  m1, [src2q]

    paddb m0, m1

    movu  [srcq], m0

    RET
```

Passons en revue le code ligne par ligne :

```assembly
%include "x86inc.asm"
```

Ce `header` développé dans les communautés x264, FFmpeg et dav1d pour fournir des axiliaires, des noms prédéfinis et des macros (comme `cglobal` ci-dessous) pour simpliflier l'écriture du code assembleur.

```assembly
SECTION .text
```
Cela désigne la section où le code que vous voulez exécuter est placé. Cela contraste avec la section `.data`, où vous pouvez placer des données constantes.

```assembly
;static void add_values(const uint8_t *src, const uint8_t *src2);  
INIT_XMM sse2
```

La première ligne est un commentaire (le point-virgule `;` en asm est l'équivalent du `//` en C) montrant à quoi ressemble le prototype de la fonction en C. La seconde ligne montre comment nous initialisons la fonction pour qu'elle utilise un registre XMM, en utilisant le jeu d'instruction sse2. Cela est nécessaire car l'instruction paddb fait partie de sse2. Nous aborderons sse2 plus en détail dans la prochaine leçon.

```assembly
cglobal add_values, 2, 2, 2, src, src2
```

Le code précédent montre une ligne importante : la définition d'une fonction en C appelée `add_values`.

Passons en revue chaque élément de la ligne un par un :

* Le paramètre suivant indique que la fonction prend deux argument (le premier 2).
* Ensuite, le deuxième 2 indique que nous allons utiliser deux GPR pour les arguments. Dans certains cas, nous pourrions vouloir utiliser plus de GPR, donc nous devrons indiquer à x86util que nous en aurons besoin de plus.
* Le paramètre suivant indique à x86util combien de registres `XMM` nous allons utiliser.
* Les deux derniers paramètres sont les labels utilisés pour les arguments de la fonction `add_values`.

Il est important de noter que le code plus ancien peut ne pas avoir les labels comme arguments de fonctions mais plutôt les adresses des registres GPR à la place, en utilisant `r0`, `r1`, etc...

```assembly
    movu  m0, [srcq]  
    movu  m1, [src2q]
```

`movu` est une abréviation de `movdqu` (*move double quad unaligned*). L’alignement sera abordé dans une autre leçon, mais pour l’instant, vous pouvez considérer movu comme un transfert de 128 bits depuis [srcq]. Dans le cas de mov, les crochets `[]` signifient que l’adresse contenue dans [srcq] est déréférencée, ce qui équivaut à **src* en C. C’est ce qu’on appelle un chargement (load).

Le suffixe `q` fait référence à la taille du pointeur. En C, cela correspond à sizeof(*src) == 8 sur les systèmes 64 bits. L’assembleur x86 est suffisamment intelligent pour utiliser 32 bits sur les systèmes 32 bits, mais l’opération de chargement sous-jacente reste de 128 bits.

Remarquez que nous ne faisons pas référence aux registres vectoriels par leur nom complet, dans ce cas `xmm0`, mais plutôt sous une forme abstraite comme `m0`. Dans les prochaines leçons, vous verrez comment cette approche permet d’écrire du code une seule fois et de le faire fonctionner avec plusieurs tailles de registres SIMD.

```assembly
paddb m0, m1
```

`paddb` (lisez ceci *p-add-b*) additionne chaque octet dans chaque registre, comme illustré ci-dessous. Le préfixe `p` signifie `packed` et sert à distinguer les instructions vectorielles des instructions scalaires. Le suffixe `b` indique que l'opération est effectuée au niveau des octets (addition d’octets).

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

C'est ce qu'on appelle un stockage (*store*). Les données sont écrites en mémoire à l'adresse pointée par `srcq`.

```assembly
RET
```

`RET` est une macro indiquant que la fonction se termine. Presque toutes les fonctions en assembleur dans FFmpeg modifient les données passées en argument au lieu de retourner une valeur.

Comme vous le verrez dans l'exercice, nous créerons des pointeurs de fonction vers des fonctions en assembleur et les utiliserons lorsque cela est possible.

[Leçon suivante](../lesson_02/index.fr.md)