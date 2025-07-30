**Lección Dos de Lenguaje Ensamblador con FFmpeg**

Ahora que has escrito tu primera función en lenguaje ensamblador, vamos a introducir las *ramificaciones* (branches) y los *bucles* (loops).

Primero necesitamos presentar la idea de *etiquetas* (labels) y *saltos* (jumps). En el ejemplo artificial siguiente, la instrucción jmp hace que la ejecución salte a la instrucción que sigue a “.loop:”. “.loop:” se conoce como una *etiqueta*, y el punto delante del nombre indica que es una *etiqueta local*, lo que te permite reutilizar el mismo nombre de etiqueta en varias funciones. Este ejemplo, por supuesto, muestra un bucle infinito, pero más adelante lo ampliaremos con algo más realista.

```assembly
mov  r0q, 3
.loop:
    dec  r0q
    jmp .loop
```

Antes de hacer un bucle realista, tenemos que introducir el registro *FLAGS*. No nos detendremos demasiado en los entresijos de *FLAGS* (de nuevo, porque las operaciones con GPR son mayormente estructurales), pero hay varias *flags* como la Zero-Flag, Sign-Flag y Overflow-Flag que se configuran en función del resultado de la mayoría de las instrucciones (excepto mov) sobre datos escalares, como las operaciones aritméticas y los desplazamientos.

Aquí tienes un ejemplo donde el contador del bucle cuenta hacia atrás hasta llegar a cero, y jg (salta si es mayor que cero) es la condición de bucle. dec r0q modifica los FLAGS en función del valor de r0q después de la instrucción, y puedes saltar en base a esos FLAGS.

```assembly
mov  r0q, 3
.loop:
    ; hacer algo
    dec  r0q
    jg  .loop ; saltar si es mayor que cero
```

Esto es equivalente al siguiente código en C:

```c
int i = 3;
do
{
   // hacer algo
   i--;
} while(i > 0)
```

Este código en C es un poco artificial. Normalmente un bucle en C se escribe así:

```c
int i;
for(i = 0; i < 3; i++) {
    // hacer algo
}
```

Esto es aproximadamente equivalente a (no hay una forma directa de emparejar este bucle ```for```):

```assembly
xor r0q, r0q
.loop:
    ; hacer algo
    inc r0q
    cmp r0q, 3
    jl  .loop ; saltar si (r0q - 3) < 0, es decir, si (r0q < 3)
```

Hay varias cosas que destacar en este fragmento. La primera es ```xor r0q, r0q```, que es una forma común de poner un registro a cero, y en algunos sistemas es más rápida que ```mov r0q, 0```, porque, simplificando, no se realiza ninguna carga real. También se puede usar en registros SIMD con ```pxor m0, m0``` para poner a cero todo un registro. Lo siguiente a observar es el uso de cmp. cmp resta efectivamente el segundo registro del primero (sin almacenar el resultado en ningún sitio) y configura los *FLAGS*, pero como se indica en el comentario, se puede leer junto con el salto (jl = saltar si menor que cero) para saltar si ```r0q < 3```.

Fíjate cómo hay una instrucción extra (cmp) en este fragmento. En general, menos instrucciones significa código más rápido, por lo que se prefiere el fragmento anterior. Como verás en futuras lecciones, hay trucos adicionales para evitar esta instrucción extra y hacer que los *FLAGS* se configuren con una operación aritmética u otra instrucción. Observa que no estamos escribiendo ensamblador para que coincida exactamente con los bucles en C, sino para que sean lo más rápidos posible en ensamblador.

Aquí tienes algunos mnemónicos de salto comunes que acabarás usando (*FLAGS* incluidos por completitud, aunque no necesitas saber los detalles para escribir bucles):

| Mnemónico | Descripción | FLAGS |
| :-------- | :------------------------------------------- | :--------------- |
| JE/JZ | Saltar si Igual/Cero | ZF = 1 |
| JNE/JNZ | Saltar si No Igual/No Cero | ZF = 0 |
| JG/JNLE | Saltar si Mayor/No Menor o Igual (con signo) | ZF = 0 y SF = OF |
| JGE/JNL | Saltar si Mayor o Igual/No Menor (con signo) | SF = OF |
| JL/JNGE | Saltar si Menor/No Mayor o Igual (con signo) | SF ≠ OF |
| JLE/JNG | Saltar si Menor o Igual/No Mayor (con signo) | ZF = 1 o SF ≠ OF |

**Constantes**

Veamos algunos ejemplos que muestran cómo usar constantes:

```assembly
SECTION_RODATA

constants_1: db 1,2,3,4
constants_2: times 2 dw 4,3,2,1
```

* SECTION_RODATA especifica que esta es una sección de datos de solo lectura. (Es un macro porque diferentes formatos de archivo de salida usados por los sistemas operativos lo declaran de forma diferente)
* constants_1: La etiqueta constants_1 se define como ```db``` (declarar byte), es decir, equivalente a uint8_t constants_1[4] = {1, 2, 3, 4};
* constants_2: Esto usa el macro ```times 2``` para repetir las palabras declaradas, es decir, equivalente a uint16_t constants_2[8] = {4, 3, 2, 1, 4, 3, 2, 1};

Estas etiquetas, que el ensamblador convierte en direcciones de memoria, pueden usarse en cargas (pero no en escrituras, ya que son de solo lectura). Algunas instrucciones aceptan una dirección de memoria como operando, por lo que pueden usarse sin necesidad de cargar explícitamente en un registro (esto tiene ventajas e inconvenientes).

**Offsets (Desplazamientos)**

Los desplazamientos son la distancia (en bytes) entre elementos consecutivos en memoria. El desplazamiento lo determina el **tamaño de cada elemento** en la estructura de datos.

Ahora que sabemos escribir bucles, es hora de obtener datos. Pero hay algunas diferencias en comparación con C. Observemos el siguiente bucle en C:

```c
uint32_t data[3];
int i;
for(i = 0; i < 3; i++) {
    data[i];
}
```

El desplazamiento de 4 bytes entre elementos de data lo calcula de antemano el compilador de C. Pero cuando escribes ensamblador a mano, debes calcular estos desplazamientos tú mismo.

Veamos la sintaxis para el cálculo de direcciones de memoria. Esto se aplica a todo tipo de direcciones:

```assembly
[base + scale*index + disp]
```

* base – Es un GPR (normalmente un puntero pasado como argumento desde C)
* scale – Puede ser 1, 2, 4, 8. El valor por defecto es 1
* index – Es un GPR (normalmente un contador de bucle)
* disp – Es un entero (hasta 32 bits). El desplazamiento es un offset dentro de los datos

x86asm proporciona la constante mmsize, que indica el tamaño del registro SIMD con el que estás trabajando.

Aquí tienes un ejemplo simple (y sin sentido práctico) para ilustrar cómo cargar usando desplazamientos personalizados:

```assembly
;static void simple_loop(const uint8_t *src)
INIT_XMM sse2
cglobal simple_loop, 1, 2, 2, src
     movq r1q, 3
.loop:
     movu m0, [srcq]
     movu m1, [srcq+2*r1q+3+mmsize]

     ; hacer algunas cosas

     add srcq, mmsize
dec r1q
jg .loop

RET
```

Fíjate cómo en ```movu m1, [srcq+2*r1q+3+mmsize]```, el ensamblador precalculará la constante de desplazamiento adecuada a usar. En la próxima lección te mostraremos un truco para evitar tener que hacer add y dec dentro del bucle, reemplazándolos por una sola instrucción add.

**LEA**

Ahora que entiendes los desplazamientos, puedes usar lea (*Load Effective Address*). Esto te permite realizar multiplicaciones y sumas en una sola instrucción, lo cual será más rápido que usar múltiples instrucciones. Por supuesto, hay limitaciones sobre con qué puedes multiplicar y qué puedes sumar, pero eso no impide que lea sea una instrucción muy potente.

```assembly
lea r0q, [base + scale*index + disp]
```

Contrario a lo que sugiere su nombre, LEA se puede usar para aritmética normal además del cálculo de direcciones. Puedes hacer algo tan complicado como:

```assembly
lea r0q, [r1q + 8*r2q + 5]
```

Ten en cuenta que esto no afecta al contenido de r1q ni de r2q. Tampoco afecta a los *FLAGS* (así que no puedes saltar en función del resultado). Usar LEA evita todas estas instrucciones y registros temporales (este código no es equivalente porque add sí modifica los *FLAGS*):

```assembly
movq r0q, r1q
movq r3q, r2q
sal  r3q, 3 ; desplazamiento aritmético a la izquierda en 3 = * 8
add  r3q, 5
add  r0q, r3q
```

Verás lea usado a menudo para preparar direcciones antes de bucles o realizar cálculos como el anterior. Ten en cuenta, por supuesto, que no puedes hacer cualquier tipo de multiplicación y suma, pero multiplicaciones por 1, 2, 4, 8 y la suma de un desplazamiento fijo son muy comunes.

En el ejercicio tendrás que cargar una constante y sumar los valores a un vector SIMD dentro de un bucle.

[Lección siguiente](../lesson_03/index.es.md)
