**Lição Dois de Linguagem Assembly do FFmpeg**

Agora que você escreveu sua primeira função em assembly, vamos apresentar desvios (*branch*) e laços (*loop*).

Primeiro, precisamos introduzir as noções de **labels** e **jumps** (saltos). Vejamos o seguinte exemplo:

```assembly
mov  r0q, 3
.loop:
    dec  r0q
    jmp .loop
```

A instrução `jmp` move a execução do código após `.loop:`. `.loop:` é o que chamamos de **label**. O **ponto** (.) prefixando o label indica que este label é um **label local**. Isso significa que você pode reutilizar o mesmo nome de label em várias funções sem provocar conflitos.

Esta porção de código é um exemplo de laço infinito, vamos melhorá-lo nas próximas lições para ter um exemplo muito mais realista.

Antes de fazer um laço realista, precisamos introduzir o registrador **FLAGS**. Não vamos entrar nos detalhes complexos dos **FLAGS** (pois as operações nos **GPRs** são principalmente andaimes), mas saiba que existem várias *flags* como a **Zero Flag** (**Z** ou **ZF**, para *indicador de zero*), a **Sign Flag** (**SF**, para *indicador de sinal*), e a **Overflow Flag** (**OF**, para *indicador de estouro*) que são um conjunto de operações baseado na saída da maioria das instruções (exceto instruções do tipo `mov`) em dados escalares como operações aritméticas e deslocamentos (*shifts*).

Aqui está um exemplo onde o contador `r0q` do laço decrementa até zero e onde `jg` (*jump if greater than zero*) é usado como condição de parada. A instrução `dec r0q` atualiza os **FLAGS** com base nos valores de `r0q` após cada decremento e você *volta ao laço* até que `r0q` atinja zero, momento em que o laço para.

```assembly
mov  r0q, 3
.loop:
    ; fazer algo
    dec  r0q
    jg  .loop ; saltar se maior que zero
```

O equivalente em **C** é o seguinte:

```c
int i = 3;
do
{
   // fazer algo
   i--;
} while(i > 0)
```

Este código em **C** é um pouco contraintuitivo. Normalmente, um laço em **C** é escrito desta maneira:

```c
int i;
for(i = 0; i < 3; i++) {
    // fazer algo
}
```

Os dois exemplos são quase equivalentes à porção em assembly seguinte (não há maneira simples de encontrar o equivalente a este laço ```for```):

```assembly
xor r0q, r0q
.loop:
    inc r0q
    ; fazer algo
    cmp r0q, 3
    jl  .loop ; saltar se (r0q - 3) < 0, i.e (r0q < 3)
```

Várias coisas devem ser examinadas neste trecho de código. Primeiro, ```xor r0q, r0q``` é uma maneira simples de atribuir zero a um registrador, o que é mais rápido em alguns sistemas que ```mov r0q, 0```, porque não há nenhum carregamento de registradores.
Isso também pode ser usado com registradores **SIMD** com a linha ```pxor m0, m0``` para zerar um registrador inteiro. A próxima coisa a notar é o uso de ```cmp```. ```cmp``` pode efetivamente subtrair o segundo registrador do primeiro (sem ter que salvar o valor em algum lugar) e atualiza as *FLAGS*, mas como indica o comentário, isso pode ser lido com a instrução de salto ```jl``` (salto se inferior a zero) para efetuar um salto se ```r0q < 3```.

Note a presença de uma instrução adicional (````cmp```) neste curto trecho. Geralmente, poucas instruções equivalem a código mais rápido, por isso o exemplo anterior é preferido. Como você verá mais tarde, existem várias técnicas para evitar instruções adicionais e fazer com que as *FLAGS* sejam definidas por uma operação aritmética ou outra operação. Note que não escrevemos assembly para corresponder exatamente aos laços em C, escrevemos laços em assembly para torná-los os mais rápidos em assembly.

Aqui estão alguns mnemônicos de salto que você acabará usando (os registradores *FLAGS* são apresentados aqui para cobrir tudo, mas você não precisará conhecê-los para escrever laços):

| Mnemônico | Significado | Descrição | FLAGS |
| :---- | :---- | :---- | :---- |
| JE/JZ | Jump if Equal/Zero | Salto se Igual / Zero | ZF = 1 |
| JNE/JNZ | Jump if Not Equal/Not Zero | Salto se Não Igual / Não Zero |  ZF = 0 |
| JG/JNLE | Jump if Greater/Not Less or Equal (signed) | Salto se Maior / Não Menor ou Igual (com sinal) |  ZF = 0 e SF = OF |
| JGE/JNL | Jump if Greater or Equal/Not Less (signed) | Salto se Maior ou Igual / Não Menor (com sinal) | SF = OF |
| JL/JNGE | Jump if Less/Not Greater or Equal (signed) | Salto se Menor / Não Maior ou Igual (com sinal) |  SF ≠ OF |
| JLE/JNG | Jump if Less or Equal/Not Greater (signed) | Salto se Menor ou Igual / Não Maior (com sinal) | ZF = 1 ou SF ≠ OF |

**Constantes**

Vamos mergulhar em alguns exemplos sobre o uso de constantes:

```assembly
SECTION_RODATA

constants_1: db 1,2,3,4
constants_2: times 2 dw 4,3,2,1
```

* SECTION_RODATA indica que esta é uma seção somente leitura. (É uma macro porque os diferentes formatos de arquivo de saída usados pelos sistemas operacionais os declaram de forma diferente.)
* o label *constants_1* é definido como sendo um ```db``` (*declared bytes*), é equivalente a uint8_t constants_1[4] = {1, 2, 3, 4};
* constants_2 usa a macro ```times 2``` para repetir a definição de palavra (```dw``` para *declared word*), é equivalente a uint16_t constants_2[8] = {4, 3, 2, 1, 4, 3, 2, 1};

Esses labels, que o assembler converte em endereços de memória, podem ser usados para carregamentos (mas não para armazenamentos, pois são somente leitura). Algumas instruções aceitam um endereço de memória como operando, o que permite usá-los sem carregamento explícito em um registrador (há vantagens e desvantagens nisso).

**Offsets (deslocamentos)**

Os offsets (*deslocamentos*) são as distâncias (em bytes) entre dois elementos consecutivos na memória. O offset é determinado pelo **tamanho de cada elemento** na estrutura de dados.

Agora que somos capazes de escrever laços, é hora de recuperar dados. Mas existem diferenças em relação ao C. Vejamos o seguinte laço em C:

```c
uint32_t data[3];
int i;
for(i = 0; i < 3; i++) {
    data[i];
}
```

O offset de 4 bytes entre cada elemento de dados é pré-calculado pelo compilador. Mas ao escrever manualmente em assembly, você precisará calcular esse offset você mesmo.

Vejamos a sintaxe do cálculo de endereços de memória. Isso se aplica a todos os tipos de endereços de memória:

```assembly
[base + scale*index + disp]
```

* base - Este é um GPR (geralmente um ponteiro vindo de um argumento de uma função em C).
* scale - Inteiro valendo 1, 2, 4, 8. O valor padrão é 1.
* index - um registrador GPR (o contador de um laço).
* disp - Inteiro (armazenado até no máximo 32 bits). disp é um deslocamento nos dados.

x86asm fornece a constante **mmsize**, que indica o tamanho do registrador SIMD com o qual você está trabalhando.

Aqui está um exemplo simples (e não muito lógico) de ilustração do carregamento com diferentes deslocamentos:

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

Note como em ```movu m1, [srcq+2*r1q+3+mmsize]``` o assembler pré-calculará o deslocamento correto a usar. Na próxima lição, apresentaremos uma técnica para evitar ter ```add``` e ```dec``` no laço, substituindo-os por um único ```add```.

**LEA**

Agora que você domina os offsets, você é capaz de usar a instrução ```lea``` (_**L**oad **E**ffective **A**ddress*_). Com uma única instrução, você é capaz de fazer uma multiplicação e uma adição, o que é então mais rápido que o uso de várias instruções. Há, claro, limitações sobre os dados que você pode multiplicar e adicionar, mas isso não impede ```lea``` de ser uma instrução poderosa.

```assembly
lea r0q, [base + scale*index + disp]
```

Ao contrário de seu nome, **LEA** pode ser usada tanto para operações aritméticas quanto para cálculos de endereços de memória. Você pode fazer algo tão complicado quanto:

```assembly
lea r0q, [r1q + 8*r2q + 5]
```

É importante notar que isso não afeta o conteúdo de ````r1q``` e ```r2q```. Isso também não impacta os registradores *FLAGS* (portanto, você não pode efetuar saltos na saída). O uso de **LEA** evita todas essas instruções e os registradores temporários (este código não tem equivalente porque ```add``` modifica as *FLAGS*):

```assembly
movq r0q, r1q
movq r3q, r2q
sal  r3q, 3 ; deslocamento aritmético à esquerda 3 = * 8
add  r3q, 5
add  r0q, r3q
```

Você verá o uso de ```lea``` no uso de muitos endereços antes de laços ou para fazer cálculos como o anterior. Note bem que você não pode fazer todos os tipos de multiplicações e adições, mas as multiplicações por 1, 2, 4 ou 8 e as adições de um deslocamento fixo são operações comuns.

No exercício, você terá que carregar uma constante e adicionar os valores a um vetor **SIMD** em um laço.

[Próxima lição](../lesson_03/index.pt-br.md)