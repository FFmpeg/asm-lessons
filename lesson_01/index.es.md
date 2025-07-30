**Lección Uno de Lenguaje Ensamblador FFmpeg**

**Introducción**

Bienvenido a la Escuela de Lenguaje Ensamblador de FFmpeg. Has dado el primer paso en el viaje más interesante, desafiante y gratificante de la programación. Estas lecciones te proporcionarán una base sobre cómo se utiliza el ensamblador en FFmpeg y te abrirán los ojos a lo que realmente ocurre en tu ordenador.

**Conocimientos necesarios**

* Conocimientos de C, en particular de punteros. Si no sabes C, estudia el libro [The C Programming Language](https://en.wikipedia.org/wiki/The_C_Programming_Language)
* Matemáticas de secundaria (escalares vs vectores, suma, multiplicación, etc.)

**¿Qué es el lenguaje ensamblador?**

El lenguaje ensamblador es un lenguaje de programación en el que se escribe código que se corresponde directamente con las instrucciones que procesa una CPU. El lenguaje ensamblador legible por humanos, como su nombre indica, se *ensamble* en datos binarios, conocidos como *código máquina*, que la CPU puede entender. Es posible que veas el código en lenguaje ensamblador denominado como *ensamblador* o *asm* para abreviar.

La gran mayoría del código ensamblador en FFmpeg es lo que se conoce como *SIMD, Single Instruction Multiple Data (instrucción única, datos múltiples)*. SIMD se conoce a veces como programación vectorial. Esto significa que una instrucción concreta opera sobre varios elementos de datos al mismo tiempo. La mayoría de los lenguajes de programación operan sobre un elemento de datos a la vez, lo que se conoce como programación secuencial.

Como habrás adivinado, SIMD se presta bien al procesamiento de imágenes, vídeo y audio, que tienen muchos datos ordenados secuencialmente en la memoria. Hay instrucciones especializadas disponibles en la CPU para ayudarnos a procesar datos secuenciales.

En FFmpeg, verás los términos «función de ensamblador», «SIMD» y «vectorizar» utilizados indistintamente. Todos ellos se refieren a lo mismo: escribir una función en lenguaje ensamblador a mano para procesar múltiples elementos de datos de una sola vez. Algunos proyectos también pueden referirse a ellos como «núcleos de ensamblador». 

Todo esto puede parecer complicado, pero es importante recordar que en FFmpeg, los estudiantes de secundaria han escrito código ensamblador. Como con todo, el aprendizaje es un 50 % jerga y un 50 % aprendizaje real.

**¿Por qué escribimos en lenguaje ensamblador?**  
Para acelerar el procesamiento multimedia. Es muy habitual conseguir una mejora de velocidad de 10 veces o más al escribir código ensamblador, lo que resulta especialmente importante cuando se desea reproducir vídeos en tiempo real sin interrupciones. Además, ahorra energía y prolonga la duración de la batería. Cabe señalar que las funciones de codificación y descodificación de vídeo son algunas de las más utilizadas en el mundo, tanto por los usuarios finales como por las grandes empresas en sus centros de datos. Por lo tanto, incluso una pequeña mejora se acumula rápidamente.

A menudo se ve en Internet que la gente utiliza *intrínsecos*, que son funciones similares a las de C que se asignan a instrucciones de ensamblador para permitir un desarrollo más rápido. En FFmpeg no utilizamos intrínsecos, sino que escribimos el código ensamblador a mano. Se trata de un tema controvertido, pero los intrínsecos suelen ser entre un 10 y un 15 % más lentos que el ensamblador escrito a mano (los defensores de los intrínsecos no estarían de acuerdo), dependiendo del compilador. Para FFmpeg, cada bit de rendimiento adicional ayuda, por lo que escribimos directamente en código ensamblador. También se argumenta que las intrínsecas son difíciles de leer debido al uso de la «[notación húngara] (https://en.wikipedia.org/wiki/Hungarian_notation)».

También es posible que veas *ensamblador en línea* (es decir, sin usar intrínsecos) en algunos lugares de FFmpeg por razones históricas, o en proyectos como el kernel de Linux debido a casos de uso muy específicos. Aquí es donde el código ensamblador no se encuentra en un archivo separado, sino que se escribe en línea con el código C. La opinión predominante en proyectos como FFmpeg es que este código es difícil de leer, no es ampliamente compatible con los compiladores y es imposible de mantener.

Por último, verás a muchos autoproclamados expertos en Internet diciendo que nada de esto es necesario y que el compilador puede hacer toda esta «vectorización» por ti. Al menos con fines de aprendizaje, ignóralos: pruebas recientes en, por ejemplo, [el proyecto dav1d](https://www.videolan.org/projects/dav1d.html) mostraron una aceleración de aproximadamente 2 veces gracias a esta vectorización automática, mientras que las versiones escritas a mano podían alcanzar 8 veces.

**Variantes del lenguaje ensamblador**  
Estas lecciones se centrarán en el lenguaje ensamblador x86 de 64 bits. También se conoce como amd64, aunque sigue funcionando en CPU Intel. Existen otros tipos de ensamblador para otras CPU, como ARM y RISC-V, y es posible que en el futuro estas lecciones se amplíen para abarcarlos.

Hay dos tipos de sintaxis de ensamblador x86 que verás en Internet: AT&T e Intel. La sintaxis AT&T es más antigua y más difícil de leer en comparación con la sintaxis Intel. Por lo tanto, utilizaremos la sintaxis Intel.

**Materiales de apoyo**  
Quizás te sorprenda saber que los libros o recursos en línea como Stack Overflow no son especialmente útiles como referencias. Esto se debe en parte a nuestra decisión de utilizar ensamblador escrito a mano con sintaxis Intel. Pero también a que muchos recursos en línea se centran en la programación de sistemas operativos o hardware, y suelen utilizar código no SIMD. El ensamblador FFmpeg se centra especialmente en el procesamiento de imágenes de alto rendimiento y, como verás, se trata de un enfoque particularmente único de la programación en ensamblador. Dicho esto, una vez completadas estas lecciones, te resultará fácil comprender otros casos de uso del ensamblador.

Muchos libros profundizan en numerosos detalles de la arquitectura informática antes de enseñar ensamblador. Esto está bien si es lo que quieres aprender, pero desde nuestro punto de vista, es como estudiar ingeniería antes de aprender a conducir un coche.

Dicho esto, los diagramas de las últimas partes del libro «The Art of 64-bit assembly» que muestran las instrucciones SIMD y su comportamiento de forma visual son útiles: [https://artofasm.randallhyde.com/](https://artofasm.randallhyde.com/)

Hay un servidor Discord disponible para responder preguntas:  
[https://discord.com/invite/Ks5MhUhqfB](https://discord.com/invite/Ks5MhUhqfB)

**Registros**  
Los registros son áreas de la CPU donde se pueden procesar los datos. Las CPU no operan directamente sobre la memoria, sino que los datos se cargan en los registros, se procesan y se vuelven a escribir en la memoria. En lenguaje ensamblador, por lo general, no se pueden copiar datos directamente de una ubicación de memoria a otra sin pasar primero esos datos por un registro.

**Registros de propósito general**  
El primer tipo de registro es el conocido como registro de propósito general (GPR). Los GPR se denominan de propósito general porque pueden contener datos, en este caso hasta un valor de 64 bits, o una dirección de memoria (un puntero). Un valor en un GPR se puede procesar mediante operaciones como suma, multiplicación, desplazamiento, etc. 

En la mayoría de los libros sobre ensamblador, hay capítulos enteros dedicados a las sutilezas de los GPR, sus antecedentes históricos, etc. Esto se debe a que los GPR son importantes cuando se trata de programación de sistemas operativos, ingeniería inversa, etc. En el código ensamblador escrito en FFmpeg, los GPR son más bien como andamios y, en la mayoría de los casos, su complejidad no es necesaria y se abstrae.

**Registros vectoriales**  
Los registros vectoriales (SIMD), como su nombre indica, contienen múltiples elementos de datos. Existen varios tipos de registros vectoriales:

* Registros mm: registros MMX, de 64 bits, históricos y que ya no se utilizan mucho.
* Registros xmm: registros XMM, de 128 bits, ampliamente disponibles.
* Registros ymm: registros YMM, de 256 bits, con algunas complicaciones a la hora de utilizarlos.   
* Registros zmm: registros ZMM, de 512 bits, disponibilidad limitada

La mayoría de los cálculos en la compresión y descompresión de vídeo se basan en números enteros, por lo que nos ceñiremos a eso. Aquí hay un ejemplo de 16 bytes en un registro xmm:

| a | b | c | d | e | f | g | h | i | j | k | l | m | n | o | p |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Pero podrían ser ocho palabras (enteros de 16 bits).

| a | b | c | d | e | f | g | h |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

O cuatro palabras dobles (enteros de 32 bits)

| a | b | c | d |
| :---- | :---- | :---- | :---- |

O dos palabras cuádruples (enteros de 64 bits):

| a | b |
| :---- | :---- |

En resumen:


* **b**ytes: datos de 8 bits  
* **w**ords: datos de 16 bits  
* **d**oublewords: datos de 32 bits  
* **q**uadwords: datos de 64 bits  
* **d**ouble **q**uadwords: datos de 128 bits

Los caracteres en negrita serán importantes más adelante.

**x86inc.asm include**  
Verás que en muchos ejemplos incluimos el archivo x86inc.asm. X86inc.asm es una capa de abstracción ligera que se utiliza en FFmpeg, x264 y dav1d para facilitar el trabajo de los programadores de ensamblador. Ayuda de muchas maneras, pero, para empezar, una de las cosas útiles que hace es etiquetar los GPR, r0, r1, r2. Esto significa que no tienes que recordar ningún nombre de registro. Como se ha mencionado anteriormente, los GPR son generalmente solo andamios, por lo que esto facilita mucho las cosas.

**Un fragmento sencillo de código ensamblador escalar**

Veamos un fragmento sencillo (y muy artificial) de código ensamblador escalar (código ensamblador que opera sobre elementos de datos individuales, uno a uno, dentro de cada instrucción) para ver qué está pasando:

```assembly
mov  r0q, 3  
inc  r0q  
dec  r0q  
imul r0q, 5
```

En la primera línea, el *valor inmediato* 3 (un valor almacenado directamente en el propio código ensamblador, en contraposición a un valor obtenido de la memoria) se almacena en el registro r0 como una palabra cuádruple. Ten en cuenta que, en la sintaxis de Intel, el operando de origen (el valor o la ubicación que proporciona los datos, situado a la derecha) se transfiere al operando de destino (la ubicación que recibe los datos, situada a la izquierda), de forma muy similar al comportamiento de memcpy. También puedes leerlo como «r0q = 3», ya que el orden es el mismo. El sufijo «q» de r0 designa que el registro se utiliza como una palabra cuádruple. inc incrementa el valor para que r0q contenga 4, dec decrementa el valor de nuevo a 3. imul multiplica el valor por 5. Así, al final, r0q contiene 15. 

Ten en cuenta que las instrucciones legibles para los humanos, como mov e inc, que el ensamblador convierte en código máquina, se conocen como *mnemónicos*. Es posible que veas en Internet y en libros mnemónicos representados con letras mayúsculas, como MOV e INC, pero son iguales que las versiones en minúsculas. En FFmpeg, utilizamos mnemónicos en minúsculas y reservamos las mayúsculas para las macros.

**Comprender una función vectorial básica**

Esta es nuestra primera función SIMD:

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

Repasemos línea por línea:

```assembly
%include "x86inc.asm"
```

Se trata de un «encabezado» o *header*  desarrollado en las comunidades x264, FFmpeg y dav1d para proporcionar *helpers*, nombres predefinidos y macros (como cglobal más abajo) que simplifican la escritura de ensamblador.

```assembly
SECTION .text
```

Esto indica la sección donde se coloca el código que se desea ejecutar. Esto contrasta con la sección .data, donde se pueden colocar datos constantes.

```assembly
;static void add_values(uint8_t *src, const uint8_t *src2)  
INIT_XMM sse2
```

La primera línea es un comentario (el punto y coma «;» en asm es como «//» en C) que muestra cómo es el argumento de la función en C. La segunda línea muestra cómo inicializamos la función para utilizar los registros XMM, utilizando el conjunto de instrucciones sse2. Esto se debe a que paddb es una instrucción sse2. Trataremos sse2 con más detalle en la próxima lección.

```assembly
cglobal add_values, 2, 2, 2, src, src2
```

Esta es una línea importante, ya que define una función C llamada «add_values». 

Repasemos cada elemento uno por uno:

* El primer parámetro muestra que la función tiene dos argumentos. 
* El parámetro siguiente muestra que utilizaremos dos GPR para los argumentos. En algunos casos, es posible que queramos utilizar más GPR, por lo que debemos indicar a x86util que necesitamos más.   
* El parámetro siguiente le indica a x86util cuántos registros XMM vamos a utilizar.
* Los dos parámetros siguientes son etiquetas para los argumentos de la función.

Cabe señalar que es posible que el código más antiguo no tenga etiquetas para los argumentos de la función, sino que se dirija directamente a los GPR utilizando r0, r1, etc.

```assembly
    movu  m0, [srcq]  
    movu  m1, [src2q]
```

movu es la abreviatura de movdqu (move double quad unaligned). La alineación se tratará en otra lección, pero por ahora movu puede considerarse como un movimiento de 128 bits desde [srcq]. En el caso de mov, los corchetes significan que la dirección en [srcq] se está desreferenciando, lo equivalente a **src en C.* Esto es lo que se conoce como una carga. Ten en cuenta que el sufijo «q» se refiere al tamaño del puntero * (*es decir, en C representa *sizeof(*src) == 8 en sistemas de 64 bits, y x86asm es lo suficientemente inteligente como para usar 32 bits en sistemas de 32 bits), pero la carga subyacente es de 128 bits.

Tenga en cuenta que no nos referimos a los registros vectoriales por su nombre completo, en este caso xmm0, sino como m0, una forma abstracta. En futuras lecciones verá cómo esto significa que puede escribir código una vez y hacer que funcione en múltiples tamaños de registros SIMD.

```assembly
paddb m0, m1
```

paddb (léase en voz alta como *p-add-b*) suma cada byte de cada registro, tal y como se muestra a continuación. El prefijo «p» significa «empaquetado» o «packed» y se utiliza para identificar las instrucciones vectoriales frente a las instrucciones escalares. El sufijo «b» indica que se trata de una suma por bytes (suma de bytes).

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

Esto es lo que se conoce como «store». Los datos se vuelven a escribir en la dirección del puntero srcq.

```assembly
RET
```

Esta es una macro para indicar los retornos de la función. Prácticamente todas las funciones de ensamblador en FFmpeg modifican los datos de los argumentos en lugar de devolver un valor.

Como verás en la tarea, creamos punteros de función a funciones de ensamblador y los utilizamos cuando están disponibles.

[Siguiente lección](../lesson_02/index.es.md)
