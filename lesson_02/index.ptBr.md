**Lição Dois de Linguagem Assembly FFmpeg**

Agora que você escreveu sua primeira função em linguagem assembly, introduziremos agora os saltos (branches) e loops.

Precisamos primeiro introduzir a ideia de rótulos (labels) e saltos (jumps). No exemplo artificial abaixo, a instrução `jmp` move a instrução de código para depois de “.loop:”. “.loop:” é conhecido como um *rótulo*, com o prefixo de ponto no rótulo significando que é um *rótulo local*, permitindo efetivamente que você reutilize o mesmo nome de rótulo em várias funções. Este exemplo, é claro, mostra um loop infinito, mas o estenderemos mais tarde para algo mais realista.

```assembly
mov  r0q, 3
.loop:
    dec  r0q
    jmp .loop
```

Antes de criar um loop realista, precisamos introduzir o registrador *FLAGS*. Não nos aprofundaremos muito nas complexidades de *FLAGS* (novamente porque as operações GPR são em grande parte um andaime), mas existem várias *flags* como Zero-Flag, Sign-Flag e Overflow-Flag que são definidas com base na saída da maioria das instruções não-mov em dados escalares, como operações aritméticas e shifts.

Aqui está um exemplo onde o contador de loop decresce até zero e `jg` (jump if greater than zero) é a condição do loop. `dec r0q` define as *FLAGS* com base no valor de `r0q` após a instrução e você pode saltar com base nelas.

```assembly
mov  r0q, 3
.loop:
    ; fazer algo
    dec  r0q
    jg  .loop ; saltar se maior que zero
```

Isso é equivalente ao seguinte código C:

```c
int i = 3;
do
{
   // fazer algo
   i--;
} while(i > 0)
```

Este código C é um pouco antinatural. Geralmente um loop em C é escrito assim:

```c
int i;
for(i = 0; i < 3; i++) {
    // fazer algo
}
```

Isso é aproximadamente equivalente a (não há uma maneira simples de igualar este loop `for`):

```assembly
xor r0q, r0q
.loop:
    ; fazer algo
    inc r0q
    cmp r0q, 3
    jl  .loop ; saltar se (r0q - 3) < 0, ou seja (r0q < 3)
```

Há várias coisas a serem apontadas neste trecho. Primeiro, `xor r0q, r0q` que é uma maneira comum de zerar um registrador, que em alguns sistemas é mais rápida do que `mov r0q, 0`, porque, em termos simples, não há nenhuma carga real acontecendo. Também pode ser usado em registradores SIMD com `pxor m0, m0` para zerar um registrador inteiro. A próxima coisa a notar é o uso de `cmp`. `cmp` efetivamente subtrai o segundo registrador do primeiro (sem armazenar o valor em nenhum lugar) e define as *FLAGS*, mas, conforme o comentário, pode ser lido junto com o salto, (`jl` = jump if less than zero) para saltar se `r0q < 3`.

Observe como há uma instrução extra (`cmp`) neste trecho. De modo geral, menos instruções significam código mais rápido, razão pela qual o trecho anterior é preferido. Como você verá em futuras lições, há mais truques usados para evitar essa instrução extra e fazer com que as *FLAGS* sejam definidas por aritmética ou outra operação. Observe como não estamos escrevendo assembly para corresponder exatamente aos loops C, escrevemos loops para torná-los o mais rápidos possível em assembly.

Aqui estão alguns mnemônicos de salto comuns que você acabará usando (as *FLAGS* estão lá para completude, mas você não precisa saber os detalhes para escrever loops):

| Mnemônico | Descrição | FLAGS |
| :---- | :---- | :---- |
| JE/JZ | Saltar se Igual/Zero | ZF = 1 |
| JNE/JNZ | Saltar se Não Igual/Não Zero | ZF = 0 |
| JG/JNLE | Saltar se Maior/Não Menor ou Igual (assinado) | ZF = 0 e SF = OF |
| JGE/JNL | Saltar se Maior ou Igual/Não Menor (assinado) | SF = OF |
| JL/JNGE | Saltar se Menor/Não Maior ou Igual (assinado) | SF ≠ OF |
| JLE/JNG | Saltar se Menor ou Igual/Não Maior (assinado) | ZF = 1 ou SF ≠ OF |

**Constantes**

Vamos ver alguns exemplos mostrando como usar constantes:

```assembly
SECTION_RODATA

constants_1: db 1,2,3,4
constants_2: times 2 dw 4,3,2,1
```

* `SECTION_RODATA` especifica que esta é uma seção de dados somente leitura. (Esta é uma macro porque diferentes formatos de arquivo de saída que os sistemas operacionais usam declaram isso de forma diferente)
* `constants_1`: O rótulo `constants_1` é definido como `db` (declare byte) - ou seja, equivalente a `uint8_t constants_1[4] = {1, 2, 3, 4};`
* `constants_2`: Isso usa a macro `times 2` para repetir as palavras declaradas - ou seja, equivalente a `uint16_t constants_2[8] = {4, 3, 2, 1, 4, 3, 2, 1};`

Esses rótulos, que o montador converte em um endereço de memória, podem então ser usados em cargas (mas não em armazenamentos, pois são somente leitura). Algumas instruções aceitam um endereço de memória como operando, de modo que podem ser usadas sem cargas explícitas em um registrador (há prós e contras nisso).

**Offsets**

Offsets são a distância (em bytes) entre elementos consecutivos na memória. O offset é determinado pelo **tamanho de cada elemento** na estrutura de dados.

Agora que somos capazes de escrever loops, é hora de buscar dados. Mas há algumas diferenças em comparação com C. Vamos olhar o seguinte loop em C:

```c
uint32_t data[3];
int i;
for(i = 0; i < 3; i++) {
    data[i];
}
```

O offset de 4 bytes entre os elementos de dados é pré-calculado pelo compilador C. Mas ao escrever assembly manualmente, você precisa calcular esses offsets sozinho.

Vamos olhar a sintaxe para cálculos de endereço de memória. Isso se aplica a todos os tipos de endereços de memória:

```assembly
[base + scale*index + disp]
```

* `base` - Este é um GPR (geralmente um ponteiro de um argumento de função C)
* `scale` - Pode ser 1, 2, 4, 8. 1 é o padrão
* `index` - Este é um GPR (geralmente um contador de loop)
* `disp` - Este é um inteiro (até 32-bit). Deslocamento é um offset nos dados

x86asm fornece a constante `mmsize`, que permite saber o tamanho do registrador SIMD com o qual você está trabalhando.

Aqui está um exemplo simples (e sem sentido) para ilustrar o carregamento a partir de offsets personalizados:

```assembly
;static void simple_loop(const uint8_t *src)
INIT_XMM sse2
cglobal simple_loop, 1, 2, 2, src
     movq r1q, 3
.loop:
     movu m0, [srcq]
     movu m1, [srcq+2*r1q+3+mmsize]

     ; fazer algumas coisas

     add srcq, mmsize
dec r1q
jg .loop

RET
```

Observe como em `movu m1, [srcq+2*r1q+3+mmsize]` o montador pré-calculará a constante de deslocamento correta a ser usada. Na próxima lição, mostraremos um truque para evitar ter que fazer `add` e `dec` no loop, substituindo-os por um único `add`.

**LEA**

Agora que você entende os offsets, você pode usar `lea` (Load Effective Address). Isso permite que você realize multiplicação e adição com uma única instrução, o que será mais rápido do que usar múltiplas instruções. Existem, é claro, limitações sobre pelo que você pode multiplicar e adicionar, mas isso não impede que `lea` seja uma instrução poderosa.

```assembly
lea r0q, [base + scale*index + disp]
```

Ao contrário do nome, `LEA` pode ser usada para aritmética normal, bem como para cálculos de endereço. Você pode fazer algo tão complicado quanto:

```assembly
lea r0q, [r1q + 8*r2q + 5]
```

Observe que isso não afeta o conteúdo de `r1q` e `r2q`. Também não afeta as *FLAGS* (então você não pode saltar com base na saída). Usar `LEA` evita todas essas instruções e registradores temporários (este código não é equivalente porque `add` altera as *FLAGS*):

```assembly
movq r0q, r1q
movq r3q, r2q
sal  r3q, 3 ; deslocamento aritmético à esquerda 3 = * 8
add  r3q, 5
add  r0q, r3q
```

Você verá `lea` sendo muito usada para configurar endereços antes de loops ou para realizar cálculos como os acima. Observe, é claro, que você não pode fazer todos os tipos de multiplicação e adição, mas multiplicações por 1, 2, 4, 8 e adição de um offset fixo são comuns.

Na tarefa, você terá que carregar uma constante e adicionar os valores a um vetor SIMD em um loop.

[Próxima Lição](../lesson_03/index.md)
