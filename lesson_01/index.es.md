# FFmpeg Assembly - Primera Lección

## Introducción

Bienvenido a la escuela de Assembly de FFmpeg. Acabas de dar el primer paso a la travesía más interesante, desafiante y enriquecedora del mundo de la programación. Estas lecciones te darán un simiento en el modo que el assembly es escrito en FFmpeg y te abrirá los ojos a lo que realmente sucede dentro de tu computadora.

Welcome to the FFmpeg School of Assembly Language. You have taken the first step on the most interesting, challenging, and rewarding journey in programming. These lessons will give you a grounding in the way assembly language is written in FFmpeg and open your eyes to what's actually going on in your computer..

## Conocimientos previos

- Dominio de C, en particular de punteros. Si aún no conoces C, empieza por el libro [The C Programming Language](https://en.wikipedia.org/wiki/The_C_Programming_Language).  
- Matemáticas de nivel secundario (escalres vs. vectores, suma, multiplicación, etc.).

## ¿Qué es el lenguaje ensamblador (Assembly)?

El lenguaje ensamblador —*assembly*, de aquí en adelante— es un lenguaje de programación en el que escribes código que se corresponde directamente con las instrucciones que procesa la CPU. El *assembly*, legible y comprensible por humanos, es, como su nombre lo indica, *ensamblado* en datos binarios conocidos como *machine code* (código máquina), que la CPU puede entender. Es común ver que se lo denomine simplemente *assembly* o, abreviado, *asm*.

La gran mayoría del *assembly* escrito en FFmpeg es lo que se conoce como **SIMD** (*Single Instruction, Multiple Data* o "una sola instrucción, múltiples datos"). A veces también se lo llama *programación vectorial*. Esto significa que una única instrucción opera sobre varios elementos de datos al mismo tiempo. La mayoría de los lenguajes de programación operan sobre un solo elemento a la vez, lo que se conoce como *programación escalar*.

Como habrás adivinado, SIMD es especialmente útil para el procesamiento de imágenes, audio y video, ya que estos contienen grandes cantidades de datos ordenados secuencialmente en memoria. Existen instrucciones especializadas dentro de la CPU que permiten procesar este tipo de datos de manera más eficiente.

En FFmpeg verás que los términos *función en assembly*, *SIMD* y *vectorizar* se usan de manera intercambiable. Todos estos conceptos hacen referencia a lo mismo: escribir una función manualmente en *assembly* para procesar múltiples datos en paralelo. Algunos proyectos también pueden referirse a esto como *núcleos de assembly* (*assembly kernels*).

Todo esto puede sonar complicado, pero es importante recordar que, en FFmpeg, incluso estudiantes de secundaria han escrito código en *assembly*. Como en muchos otros campos, aprender esto es 50 % entender la jerga y 50 % comprender los conceptos fundamentales.

## ¿Por qué escribimos en assembly?

Para hacer el procesamiento multimedia más rápido. Es muy común obtener una mejora de rendimiento de 10 veces o más al escribir código en *assembly*, lo cual es especialmente importante cuando se desea reproducir videos en tiempo real sin interrupciones. También ayuda a ahorrar energía y extender la duración de la batería. Vale la pena destacar que las funciones de codificación y decodificación de video están entre las más utilizadas del mundo, tanto por usuarios finales como por grandes empresas en sus centros de datos. Por lo tanto, incluso una pequeña mejora en rendimiento tiene una gran repercusión.

A menudo verás en internet que la gente utiliza los denominados *intrinsics*, que son funciones similares a C que se corresponden con instrucciones en *assembly*, y que permiten un desarrollo más rápido. En FFmpeg no utilizamos *intrinsics*, sino que escribimos el código ensamblador a mano. Esta es un área polémica, pero los *intrinsics* suelen ser entre un 10 % y un 15 % más lentos que el ensamblador escrito a mano (aunque quienes los defienden podrían no estar de acuerdo), dependiendo del compilador. En FFmpeg, cada pequeño incremento en el rendimiento importa, por eso escribimos directamente en *assembly*. Además, se argumenta que los *intrinsics* son difíciles de leer debido a su uso de la [notación húngara](https://es.wikipedia.org/wiki/Notaci%C3%B3n_h%C3%BAngara).

También podrías encontrar *assembly* en línea (*inline assembly*) —es decir, sin usar *intrinsics*— aún presente en algunas partes de FFmpeg por razones históricas, o en proyectos como el kernel de Linux debido a casos de uso muy específicos. Esto se refiere a cuando el código ensamblador no está en un archivo separado, sino escrito directamente dentro del código C. En proyectos como FFmpeg prevalece la opinión de que este tipo de código es difícil de leer, no está ampliamente soportado por los compiladores y resulta difícil de mantener.

Por último, verás a muchos autoproclamados expertos en línea decir que nada de esto es necesario, y que el compilador puede encargarse automáticamente de toda esta “vectorización”. Al menos con fines de aprendizaje, ignóralos: pruebas recientes en, por ejemplo, el proyecto [dav1d](https://www.videolan.org/projects/dav1d.html) mostraron una mejora de rendimiento de aproximadamente 2x con vectorización automática, mientras que las versiones escritas a mano lograron hasta 8x.

## Sabores del lenguaje ensamblador

Estas lecciones se enfocarán en *assembly* para x86 de 64 bits. Esta variante también es conocida como **amd64**, aunque sigue funcionando en procesadores Intel. Existen otros tipos de *assembly* para otras arquitecturas de CPU, como ARM y RISC-V, y es posible que en el futuro estas lecciones se amplíen para cubrir esas variantes.

Existen dos sabores de sintaxis para *assembly* x86 que verás comúnmente en Internet: **AT&T** e **Intel**. La sintaxis AT&T es más antigua y difícil de leer en comparación con la sintaxis Intel, por lo que utilizaremos esta última.

## Materiales de apoyo

Quizás te sorprenda saber que ni los libros ni recursos en línea como Stack Overflow son particularmente útiles como referencia. Esto se debe en parte a nuestra decisión de usar *assembly* escrito a mano con sintaxis Intel, pero también a que muchos recursos disponibles están enfocados en programación de sistemas operativos o hardware, los cuales normalmente no utilizan SIMD. El *assembly* en FFmpeg está especialmente orientado al procesamiento de imágenes de alto rendimiento, y como verás, representa un enfoque bastante único en la programación en ensamblador. Dicho esto, una vez que completes estas lecciones, te resultará fácil entender otros casos de uso de *assembly*.

Muchos libros profundizan en los detalles de la arquitectura de las computadoras antes de enseñar *assembly*. Esto está bien si eso es lo que deseas aprender, pero desde nuestra perspectiva, es como estudiar cómo funciona un motor antes de aprender a conducir un automóvil.

Dicho esto, los diagramas en las últimas secciones del libro **"The Art of 64-bit Assembly"**, que muestran instrucciones SIMD y su comportamiento de manera visual, son de gran ayuda:  
[https://artofasm.randallhyde.com/](https://artofasm.randallhyde.com/)

Un servidor de Discord está disponible para responder tus preguntas:  
[https://discord.com/invite/Ks5MhUhqfB](https://discord.com/invite/Ks5MhUhqfB)

## Registros

Los registros son áreas de la CPU donde se puede procesar la información. Las CPUs no operan directamente sobre la memoria; en cambio, los datos se cargan en los registros, se procesan y finalmente se vuelven a escribir en la memoria. En *assembly*, generalmente no es posible copiar datos directamente de una ubicación de memoria a otra sin antes pasar esa información por un registro.

## Registros de propósito general

El primer tipo de registro es lo que se conoce como **registro de propósito general** (*General Purpose Register*, o *GPR*). Se les llama "de propósito general" porque pueden contener tanto datos (hasta un valor de 64 bits), como direcciones de memoria (punteros). Un valor en un GPR puede ser procesado mediante operaciones como suma, multiplicación, desplazamientos (*shifting*), etc.

En la mayoría de los libros sobre *assembly*, hay capítulos enteros dedicados a las sutilezas de los GPRs, su contexto histórico, y más. Esto se debe a que los GPRs son especialmente relevantes en programación de sistemas operativos, ingeniería inversa, entre otros. En el código *assembly* que escribimos en FFmpeg, los GPRs funcionan más como una estructura de soporte (*scaffolding*), y la mayoría del tiempo sus complejidades no son necesarias, por lo que podemos abstraernos de ellas.

## Registros vectoriales

Los registros vectoriales (*SIMD registers*), como su nombre lo sugiere, contienen múltiples elementos de datos. Existen varios tipos de registros vectoriales:

- **Registros MM (mm)** — Registros MMX, con un tamaño de 64 bits. Se mantienen por razones históricas y ya no se utilizan mucho en la actualidad.
- **Registros XMM (xmm)** — Registros de 128 bits. Generalmente están disponibles en la mayoría de los sistemas.
- **Registros YMM (ymm)** — Registros de 256 bits. Presentan algunas complicaciones al usarse.
- **Registros ZMM (zmm)** — Registros de 512 bits. Tienen disponibilidad limitada.

La mayoría de los cálculos en compresión y descompresión de video se basan en enteros, así que nos enfocaremos en eso. Aquí tienes un ejemplo de 16 bytes en un registro `xmm`:

| a | b | c | d | e | f | g | h | i | j | k | l | m | n | o | p |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Pero también podría contener ocho *words* (enteros de 16 bits) 

| a | b | c | d | e | f | g | h |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

O cuatro *double words* (Enteros de 32 bits).

| a | b | c | d |
| :---- | :---- | :---- | :---- |

O dos *quadwords* (enteros de 64 bits).

| a | b |
| :---- | :---- |


Para recapitular:

* **b**ytes - 8-bit data  
* **w**ords - 16-bit data  
* **d**oublewords - 32-bit data  
* **q**uadwords - 64-bit data  
* **d**ouble **q**uadwords - 128-bit data

Las letras en negritas serán importantes más tarde.

## Inclusión de x86inc.asm

Verás que en muchos ejemplos se incluye el archivo `x86inc.asm`. Este archivo es una pequeña capa de abstracción utilizada en FFmpeg, x264 y dav1d para facilitarle la vida a quien programa en *assembly*. Ayuda de varias maneras pero, para comenzar, una de sus funciones más útiles es que etiqueta los registros de propósito general (GPRs) como `r0`, `r1`, `r2`, etc. Esto significa que no tienes que memorizar los nombres reales de los registros. Como se mencionó antes, los GPRs suelen funcionar solo como soporte (*scaffolding*), así que esto simplifica bastante las cosas.

## Un ejemplo simple de código asm escalar

Veamos un ejemplo simple (y completamente artificial) de código *assembly* escalar —es decir, código que opera sobre elementos de datos individuales, uno por instrucción— para entender qué está ocurriendo:

```assembly
mov  r0q, 3  
inc  r0q  
dec  r0q  
imul r0q, 5
```

En la primera línea, el *valor inmediato* 3 (un valor almacenado directamente en el propio código *assembly*, en lugar de ser leído desde memoria) se guarda en el registro `r0` como una *quadword*. Ten en cuenta que en la sintaxis de Intel, el operando fuente (el valor o ubicación que provee los datos, ubicado a la derecha) se transfiere al operando destino (la ubicación que recibe los datos, ubicada a la izquierda), de manera similar al comportamiento de `memcpy`. También puedes leerlo como `r0q = 3`, ya que el orden es el mismo. El sufijo `q` en `r0` indica que el registro se está utilizando como *quadword*.

`inc` incrementa el valor, por lo que `r0q` pasa a contener 4. Luego `dec` lo decrementa nuevamente a 3. La instrucción `imul` multiplica el valor por 5. Así que al final, `r0q` contiene 15.

Ten en cuenta que las instrucciones legibles para humanos como `mov` e `inc`, que son ensambladas a código de máquina por el ensamblador, se conocen como *mnemónicos*. Es posible que en internet o en libros los veas escritos en mayúsculas como `MOV` o `INC`, pero son equivalentes a sus versiones en minúsculas. En FFmpeg usamos mnemónicos en minúscula y reservamos las mayúsculas para las macros.


## Comprendiendo una función vectorial báscia

He Aquí nuestra primera función SIMD:

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

Veamos ésto línea por línea:

```assembly
%include "x86inc.asm"
```

Este es un “header” desarrollado en las comunidades de x264, FFmpeg y dav1d para proporcionar funciones auxiliares (helpers), nombres predefinidos y macros (como `cglobal`, que veremos a continuación) que simplifican la escritura de código en *assembly*.


```assembly
SECTION .text
```

Esto indica la sección donde se coloca el código que deseas ejecutar. Esto contrasta con la sección `.data`, donde puedes colocar constantes.

```assembly
;static void add_values(uint8_t *src, const uint8_t *src2)  
INIT_XMM sse2
```

La primera línea es un comentario (el punto y coma `;` en asm es como `//` en C) que muestra cómo se vería el argumento de la función en C. La segunda línea muestra cómo estamos inicializando la función para usar registros XMM, utilizando el conjunto de instrucciones SSE2. Esto se debe a que `paddb` es una instrucción de SSE2. Veremos SSE2 con más detalle en la próxima lección.

```assembly
cglobal add_values, 2, 2, 2, src, src2
```

Esta es una línea importante, ya que define una función de C llamada `add_values`.

Veamos cada elemento uno por uno:

* El parámetro siguiente a `add_values` indica que la función tiene dos argumentos.  
* El parámetro que sigue indica que usaremos dos registros de propósito general (GPRs) para los argumentos. En algunos casos, podríamos querer usar más GPRs, por lo que debemos indicarle a `x86util` que los necesitamos.  
* El parámetro siguiente le dice a `x86util` cuántos registros XMM vamos a usar.  
* Los dos parámetros posteriores son etiquetas para los argumentos de la función.

Vale la pena mencionar que en código más antiguo es posible que no haya etiquetas para los argumentos de la función y, en su lugar, se acceda directamente a los GPRs usando `r0`, `r1`, etc.

```assembly
    movu  m0, [srcq]  
    movu  m1, [src2q]
```

`movu` es una forma abreviada de `movdqu` (*move double quad unaligned*). La alineación (*alignement*) se cubrirá en otra lección, pero por ahora puedes tratar `movu` como una instrucción que mueve 128 bits desde `[srcq]`. En el caso de `mov`, los corchetes indican que se está de-referenciando la dirección en `srcq`, lo equivalente a `*src` en C. Esto es lo que se conoce como una carga (*load*).

Ten en cuenta que el sufijo `q` se refiere al tamaño del puntero (es decir, en C representa que `sizeof(*src) == 8` en sistemas de 64 bits, y `x86asm` es lo suficientemente inteligente como para usar 32 bits en sistemas de 32 bits), pero la carga subyacente sigue siendo de 128 bits.

También vale notar que no nos referimos a los registros vectoriales por su nombre completo (como `xmm0`), sino como `m0`, una forma abstraida. En lecciones futuras verás cómo esto permite escribir código una sola vez y hacerlo funcionar con distintos tamaños de registros SIMD.

```assembly
paddb m0, m1
```

`paddb` (léelo mentalmente como *p-add-b*) está sumando cada byte en cada registro, como se muestra abajo.  
El prefijo `p` significa *packed* (*empaquetado*) y se usa para identificar instrucciones vectoriales en lugar de instrucciones escalares.  
El sufijo `b` indica que se trata de una suma por bytes (suma de valores byte a byte).


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
Esto se conoce como guardado (*store*). La data es escrita nuevamente a la direccion en el puntero `srcq`.

```assembly
RET
```

Esta es una macro que indica el retorno de la función. Prácticamente todas las funciones en ensamblador dentro de FFmpeg modifican los datos en los argumentos en lugar de devolver un valor.

Como verás en el ejercicio, creamos punteros a funciones en ensamblador y los usamos cuando están disponibles.

[Siguiente Lección](../lesson_02/index.es.md)
