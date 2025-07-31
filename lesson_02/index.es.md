# FFmpeg Assembly - Segunda Lección

Ahora que ya escribiste tu primera función en lenguaje ensamblador, vamos a introducir ramas y bucles.

Primero necesitamos presentar la idea de etiquetas (*labels*) y saltos (*jumps*). En el ejemplo artificial de abajo, la instrucción `jmp` transfiere la ejecución al la instrucción debajo de `.loop:`. `.loop:` es lo que se conoce como una *etiqueta*, y el punto al inicio indica que es una *etiqueta local* (*local label*), lo que permite reutilizar el mismo nombre de etiqueta en distintas funciones. Este ejemplo muestra un bucle infinito, pero luego lo haremos más realista.

```assembly
mov  r0q, 3
.loop:
    dec  r0q
    jmp .loop
```

Antes de crear un bucle realista debemos presentar el registro *FLAGS*. No entraremos en muchos detalles sobre *FLAGS* (nuevamente, porque las operaciones con GPR suelen ser solo andamiaje), pero hay varias banderas como Zero-Flag, Sign-Flag y Overflow-Flag que se activan en base al resultado de la mayoría de las instrucciones *no-`mov`* sobre datos escalares (como operaciones aritméticas y desplazamientos).

Aquí hay un ejemplo donde un contador de bucle cuenta hacia abajo hasta cero, y `jg` ("saltar si mayor que cero") actúa como condición del bucle. `dec r0q` modifica *FLAGS* en base al valor de `r0q` y esto puede ser usado para ejecutar el salto.

```assembly
mov  r0q, 3
.loop:
    ; do something
    dec  r0q
    jg  .loop ; jump if greater than zero
```
Esto equivale al siguiente código en C:

```c
int i = 3;
do
{
   // do something
   i--;
} while(i > 0)
```

Este código en C es algo poco común. Normalmente, un bucle se escribe así:

```c
int i;
for(i = 0; i < 3; i++) {
    // do something
}
```
Esto se traduce aproximadamente como (no hay una forma simple de igualar exactamente este `for` en ensamblador):

```assembly
xor r0q, r0q
.loop:
    ; do something
    inc r0q
    cmp r0q, 3
    jl  .loop ; jump if (r0q - 3) < 0, i.e (r0q < 3)
```

Hay varios aspectos para destacar en este fragmento. Primero, el uso de `xor r0q, r0q` es una forma común de poner un registro en cero; en algunos sistemas es más rápido que `mov r0q, 0` ya que, en resumen, no realiza una carga real. También se puede usar con registros SIMD usando `pxor m0, m0` para poner todo el registro en cero. 

Luego, observa el uso de `cmp`. Esta instrucción efectivamente resta el segundo operando del primero (sin guardar el resultado) y ajusta *FLAGS*. Junto con `jl` ("saltar si menor que cero"), se puede leer como un salto si `r0q < 3`.

Nota cómo este ejemplo usa una instrucción adicional (`cmp`) en comparación con el anterior. Generalmente, menos instrucciones significa código más rápido, razón por la cual el primer fragmento es preferido. Como veremos en futuras lecciones, existen varios trucos para evitar estas instrucciones adicionales y aprovechar otras que ya ajustan *FLAGS* al hacer operaciones.

Recuerda: no se escribe assembly para imitar exactamente los bucles en C; se escribe para que sea lo más rápido posible en assembly.

## Saltos comunes

Aquí hay algunos mnemónicos de salto que usarás con frecuencia (*FLAGS* se incluyen como referencia, no es necesario memorizarlos):

| Mnemonic | Description  | FLAGS |
| :---- | :---- | :---- |
| JE/JZ | Jump if Equal/Zero | ZF = 1 |
| JNE/JNZ | Jump if Not Equal/Not Zero | ZF = 0 |
| JG/JNLE | Jump if Greater/Not Less or Equal (signed) | ZF = 0 and SF = OF |
| JGE/JNL | Jump if Greater or Equal/Not Less (signed) | SF = OF |
| JL/JNGE | Jump if Less/Not Greater or Equal (signed) | SF ≠ OF |
| JLE/JNG | Jump if Less or Equal/Not Greater (signed) | ZF = 1 or SF ≠ OF |

## Constantes

Veamos algunos ejemplos de cómo usar constantes:

```assembly
SECTION_RODATA

constants_1: db 1,2,3,4
constants_2: times 2 dw 4,3,2,1
```

* `SECTION_RODATA` indica que esta es una sección de solo lectura. (Es una macro porque los distintos formatos de salida usados por los sistemas operativos lo declaran de forma diferente)
* `constants_1:` define una etiqueta y declara bytes (`db`), equivalente a `uint8_t constants_1[4] = {1, 2, 3, 4};`
* `constants_2:` utiliza el macro `times 2` para repetir las palabras (`dw`), equivalente a `uint16_t constants_2[8] = {4, 3, 2, 1, 4, 3, 2, 1};`

Estas etiquetas son convertidas por el ensamblador en direcciones de memoria, y pueden usarse en operaciones de lectura (*loads*), pero no de escritura (*stores*) ya que son *read-only*. Algunas instrucciones aceptan directamente una dirección de memoria como operando, sin necesidad de cargarla antes en un registro (lo cual tiene ventajas y desventajas).


## Offsets

Los *offsets* son la distancia (en bytes) entre elementos consecutivos en memoria. El *offset* depende del **tamaño de cada elemento** en la estructura de datos.

Ahora que podemos escribir bucles, es hora de acceder a los datos. No obstante, hay algunas diferencias respecto a C. Considera este bucle en C:

```c
uint32_t data[3];
int i;
for(i = 0; i < 3; i++) {
    data[i];
}
```

El compilador de C calcula automáticamente el *offset* de 4 bytes entre elementos, pero al escribir ensamblador a mano debes calcular estos *offsets* tú mismo.

Veamos la sintaxis general para calcular direcciones de memoria. Esto aplica a todos los tipos de direcciones en memoria:

```assembly
[base + scale*index + disp]
```

* `base`: un GPR (generalmente un puntero pasado como argumento desde C)
* `scale`: puede ser 1, 2, 4 u 8. Por defecto es 1.
* `index`: otro GPR (usualmente un contador de bucle)
* `disp`: un entero (hasta 32 bits). Es un desplazamiento dentro del arreglo o estructura.

x86asm provee la constante `mmsize`, que representa el tamaño del registro SIMD con el que estás trabajando.

Ejemplo simple (aunque sin sentido práctico) para ilustrar cómo cargar desde *offsets* personalizados:

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


Nota cómo en `movu m1, [srcq+2*r1q+3+mmsize]`, el ensamblador calculará automáticamente el desplazamiento correcto. En la próxima lección veremos un truco para evitar tener que usar `add` y `dec` dentro del bucle, reemplazándolos por una sola instrucción.

## LEA

Ya que entiendes los *offsets*, puedes usar `lea` (*Load Effective Address*). Esta instrucción permite realizar multiplicaciones y sumas en una sola operación, lo cual es más rápido que usar varias instrucciones por separado. Hay limitaciones, claro, sobre qué puedes multiplicar y sumar, pero eso no impide que `lea` sea una poderosa instrucción.

```assembly
lea r0q, [base + scale*index + disp]
```
A pesar del nombre, `lea` también puede usarse para operaciones aritméticas normales, no solo para direcciones. Por ejemplo:

```assembly
lea r0q, [r1q + 8*r2q + 5]
```

Esto no modifica los contenidos de `r1q` ni `r2q`. Tampoco afecta las *FLAGS* (de modo que no puedes hacer saltos condicionales basados en su resultado). Usar `lea` evita instrucciones adicionales y registros temporales. Este código no es equivalente, porque `add` sí cambia los *FLAGS*:

```assembly
movq r0q, r1q
movq r3q, r2q
sal  r3q, 3 ; shift arithmetic left 3 = * 8
add  r3q, 5
add  r0q, r3q
```

Verás que `lea` se usa mucho para preparar direcciones antes de un bucle o realizar cálculos como el anterior. Claro que no permite todo tipo de multiplicaciones y sumas, pero multiplicar por 1, 2, 4, 8 y agregar un *offset* fijo es algo común.


En la asignación tendrás que cargar una constante y sumarla a un vector SIMD dentro de un bucle.

[Siguiente Lección](../lesson_03/index.es.md)
