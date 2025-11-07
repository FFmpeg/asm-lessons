**Lição Três de Linguagem Assembly do FFmpeg**

Vamos explicar um pouco mais de jargão e leia bem esta pequena lição de história.

**Conjuntos de Instruções**

Você certamente prestou atenção ao fato de que na lição anterior, falamos sobre **SSE2** que faz parte do conjunto de instruções **SIMD**. Quando uma nova geração de CPU é lançada no mercado, ela pode vir acompanhada de novas instruções e às vezes registradores de tamanhos maiores. A história do conjunto de instruções x86 é tão complexa que existe uma versão simplificada (com várias subhistórias dentro dela):

* MMX - Lançado em 1997, a primeira versão **SIMD** nos Processadores Intel, registradores de 64 bits, versão histórica.
* SSE (Streaming SIMD Extensions) - Lançado em 1999, registradores de 128 bits.
* SSE2 - Lançado em 2000, várias novas instruções.
* SSE3 - Lançado em 2004, primeiras instruções *horizontais*.
* SSSE3 (Supplemental SSE3) - Lançado em 2006, adição de novas instruções incluindo a mais importante ```pshufb```, sem dúvida a instrução mais importante para processamento de vídeo
* SSE4 - Lançado em 2008, adição de muitas novas instruções, notavelmente mínimos e máximos em modo vetorizado.
* AVX - Lançado em 2011, adição de registradores de 256 bits (apenas para pontos flutuantes) e uma nova sintaxe de três operandos.
* AVX2 - Lançado em 2013, adição de registradores de 256 bits para registradores para instruções inteiras.
* AVX512 - Lançado em 2017, registradores de 512 bits, nova instrução de mascaramento. Seu uso no FFmpeg era muito limitado devido à diminuição da frequência da CPU quando novas instruções eram usadas. Permutação completa de 512 bits com ```vpermb```.
* AVX512ICL - Lançado em 2019, remoção da redução de frequência do processador.
* AVX10 - Por vir.

É importante notar que conjuntos de instruções podem ser removidos ou adicionados de uma CPU para outra. Por exemplo, **AVX512** foi [removido](https://www.igorslab.de/en/intel-deactivated-avx-512-on-alder-lake-but-fully-questionable-interpretation-of-efficiency-news-editorial/), de forma controversa, na 12ª geração de CPUs Intel. É por isso que o FFmpeg faz detecção de CPU em tempo de execução. O FFmpeg detecta as capacidades do processador no qual está sendo executado.

Como você viu no exercício, os ponteiros de função são por padrão em **C** e são substituídos por um conjunto de instruções particular. Isso significa que a detecção é realizada uma vez e nunca mais será necessária. Isso contrasta com muitas aplicações proprietárias que programam rigidamente em seu código um conjunto de instruções tornando obsoletos computadores perfeitamente funcionais. Isso também permite ativar ou desativar funções otimizadas em tempo de execução. Este é um dos maiores benefícios do código aberto.

Programas como o FFmpeg são usados por bilhões de dispositivos ao redor do mundo, alguns deles podem ser muito antigos. Tecnicamente, o FFmpeg é compatível com máquinas que possuem apenas o conjunto de instruções **SSE**, máquinas que datam de 25 anos atrás.
Felizmente, **x86inc.asm** é capaz de indicar se uma instrução não está disponível em um conjunto de instruções particular.

Para dar uma ideia das compatibilidades, aqui está a disponibilidade dos conjuntos de instruções segundo o [Steam Survey](https://store.steampowered.com/hwsurvey/Steam-Hardware-Software-Survey-Welcome-to-Steam) em novembro de 2024 (obviamente tendencioso em favor dos jogadores):

| Conjunto de Instruções | Disponibilidade |
| :---- | :---- |
| SSE2 | 100% |
| SSE3 | 100% |
| SSSE3 | 99.86% |
| SSE4.1 | 99.80% |
| AVX | 97.39% |
| AVX2 | 94.44% |
| AVX512 (AVX512 e AVX512ICL combinados) | 14.09% |

Para uma aplicação como o FFmpeg com bilhões de usuários, mesmo 0.1% deles representa um grande número de usuários e relatórios de bugs em caso de problemas. O FFmpeg possui uma grande infraestrutura de testes para testar cada combinação de CPU/OS/compiladores apresentada no [FATE testsuite](https://fate.ffmpeg.org/?query=subarch:x86_64%2F%2F). Cada commit único é executado em centenas de máquinas para ter certeza de que nada quebra.

A Intel fornece um manual de conjunto de instruções detalhado aqui: [https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)

Pode ser trabalhoso pesquisar em um PDF, uma alternativa online está presente aqui: [https://www.felixcloutier.com/x86/](https://www.felixcloutier.com/x86/)

Há também uma representação visual das instruções SIMD disponível aqui: [https://www.officedaytime.com/simd512e/](https://www.officedaytime.com/simd512e/)

Parte do desafio do uso do assembly x86 é encontrar a instrução certa que você precisa. Em alguns casos, instruções podem ser usadas em situações para as quais não foram originalmente feitas.

**Truques sobre Deslocamentos de Ponteiros**

Vamos voltar à nossa função original da Lição 1, mas adicionar um argumento de largura à função em C.

Usamos `ptrdiff_t` para a variável de largura em vez de `int` para garantir que os 32 bits superiores do argumento de 64 bits sejam zerados. Se passássemos diretamente uma largura em `int` na assinatura da função e depois tentássemos usá-la como um `quad` para aritmética de ponteiros (ou seja, usando `widthq`), os 32 bits superiores do registrador poderiam conter valores arbitrários. Poderíamos corrigir isso estendendo o sinal de width com `movsxd` (veja também a macro `movsxdifnidn` em **x86inc.asm**), mas este método é mais simples.

A função abaixo contém o truque dos deslocamentos de ponteiro:

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

Vamos revisar isso passo a passo, pois pode ser confuso:

```assembly
   add srcq, widthq
   add src2q, widthq
   neg widthq
```

A largura `widthq` é adicionada a cada ponteiro de modo que cada ponteiro agora aponta para o final do buffer a processar. O sinal de `widthq` é então invertido.

```assembly
    movu  m0, [srcq+widthq]
    movu  m1, [src2q+widthq]
```

Os carregamentos são então feitos com `widthq` sendo negativo. Assim, na primeira iteração, `[srcq+widthq]` aponta para o endereço original de `srcq`, ou seja, aponta para o início do buffer.

```assembly
    add   widthq, mmsize
    jl .loop
```

`mmsize` é adicionado a `widthq` negativo, aproximando-o assim de zero. A condição do laço agora é `jl` (*jump if less than zero*). Este truque permite que `widthq` seja usado tanto como deslocamento de ponteiro quanto como contador de laço, economizando assim uma instrução `cmp`. Também permite usar o deslocamento do ponteiro em múltiplos carregamentos e armazenamentos, bem como usar múltiplos dos deslocamentos de ponteiro se necessário (lembre-se disso para o exercício).

**Alinhamento**

Em todos os nossos exemplos, usamos `movu` para evitar o assunto do alinhamento. Muitos processadores podem carregar e armazenar dados mais rapidamente se eles estiverem alinhados, ou seja, se o endereço de memória for divisível pelo tamanho do registrador SIMD. Quando possível, tentamos usar carregamentos e armazenamentos alinhados no FFmpeg usando `mova`.

No FFmpeg, `av_malloc` pode fornecer memória alinhada no heap, e a diretiva de pré-processador C DECLARE_ALIGNED pode fornecer memória alinhada na pilha. Se `mova` for usado com um endereço não alinhado, isso resultará em uma falha de segmentação e a aplicação travará. Também é importante garantir que o valor de alinhamento corresponda ao tamanho do registrador SIMD, ou seja, 16 para xmm, 32 para `ymm` e 64 para `zmm`.

Aqui está como alinhar o início da seção RODATA em 64 bytes:

```assembly
SECTION_RODATA 64
```

Note que isso alinha apenas o início da seção RODATA. Bytes de preenchimento podem ser necessários para garantir que o label seguinte permaneça em um limite de 64 bytes.

**Desempacotamento**

Outro assunto que evitamos até agora é o estouro. Isso ocorre, por exemplo, quando o valor de um byte excede 255 após uma operação como adição ou multiplicação. Podemos querer realizar uma operação onde precisamos de um valor intermediário maior que um byte (por exemplo, palavras), ou potencialmente queremos deixar os dados nesse tamanho intermediário maior.

Para bytes não assinados, é aí que as instruções `punpcklbw` (desempacotar bytes em palavras na parte baixa de um pacote) e `punpckhbw` (desempacotar bytes em palavras na parte alta de um pacote) entram.

Vamos ver como funciona `punpcklbw`. A sintaxe para a versão SSE2 no manual da Intel é a seguinte:

| PUNPCKLBW xmm1, xmm2/m128 |
| :---- |

Isso significa que sua fonte (lado direito) pode ser um registrador xmm ou um endereço de memória (m128 significa um endereço de memória com a sintaxe padrão **[base + escala*índice + deslocamento]**), e o destino é um registrador xmm.

O site officedaytime.com acima tem um bom diagrama mostrando o que acontece:

![O que é isso](image1.png)

Você pode ver que os bytes são entrelaçados a partir da metade inferior de cada registrador respectivamente. Mas que relação isso tem com a extensão de faixa? Se o registrador fonte estiver totalmente zerado, isso entrelaça os bytes no registrador de destino com zeros. Isso é chamado de *extensão por zero*, porque os bytes são sem sinal. punpckhbw pode ser usado para fazer a mesma coisa para os bytes superiores.

Aqui está um trecho mostrando como isso é feito:

```assembly
pxor      m2, m2 ; zerar m2

movu      m0, [srcq]
movu      m1, m0 ; fazer uma cópia de m0 em m1
punpcklbw m0, m2
punpckhbw m1, m2
```

```m0``` e ```m1``` agora contêm os bytes originais estendidos a zero em palavras. Na próxima lição, você verá como as instruções de três operandos em AVX tornam o segundo ```movu``` desnecessário.

**Extensão de Sinal**

Dados com sinal são um pouco mais complicados. Para estender a faixa de um inteiro com sinal, precisamos usar um processo conhecido como [extensão de sinal](https://en.wikipedia.org/wiki/Sign_extension). Isso consiste em adicionar bits de sinal nos bits mais significativos. Por exemplo: ```-2``` em ```int8_t``` é ```0b11111110```. Para estendê-lo para ```int16_t```, o bit de sinal ```1``` é repetido para obter ```0b1111111111111110```.

```pcmpgtb``` (packed compare greater than byte) pode ser usado para extensão de sinal. Ao fazer a comparação (0 > byte), todos os bits no byte de destino são definidos como 1 se o byte for negativo, caso contrário os bits no byte de destino são definidos como 0. ```punpckX``` pode ser usado como acima para realizar a extensão de sinal. Se o byte for negativo, o byte correspondente é ```0b11111111``` e caso contrário é ```0x00000000```. O entrelaçamento do valor do byte com a saída de ```pcmpgtb``` realiza uma extensão de sinal em palavra.

```assembly
pxor      m2, m2 ; zerar m2

movu      m0, [srcq]
movu      m1, m0 ; fazer uma cópia de m0 em m1

pcmpgtb   m2, m0
punpcklbw m0, m2
punpckhbw m1, m2
```

Como você pode ver, há uma instrução adicional em comparação com o caso sem sinal.

**Empacotamento**

```packuswb``` (empacotar palavra sem sinal em byte) e ```packsswb``` permitem que você passe de palavra para byte. Eles permitem entrelaçar dois registradores SIMD contendo palavras em um único registrador SIMD com um byte. Note que se os valores excederem a faixa de bytes, eles serão saturados (ou seja, limitados ao valor máximo).

**Embaralhamentos**

Os embaralhamentos (*shuffles*), também chamados de permutações, são indiscutivelmente a instrução mais importante no processamento de vídeo e ```pshufb``` (packed shuffle bytes), disponível em SSSE3, é a variante mais importante.

Para cada byte, o byte fonte correspondente é usado como um índice no registrador de destino, exceto quando o bit mais significativo (MSB) está ativado, o byte de destino é zerado. Este é o análogo do seguinte código C (embora em SIMD, as 16 iterações do laço ocorrem em paralelo):

```c
for(int i = 0; i < 16; i++) {
    if(src[i] & 0x80)
        dst[i] = 0;
    else
        dst[i] = dst[src[i]]
}
```

Aqui está um exemplo simples em assembly:

```assembly
SECTION_DATA 64

shuffle_mask: db 4, 3, 1, 2, -1, 2, 3, 7, 5, 4, 3, 8, 12, 13, 15, -1

section .text

movu m0, [srcq]
movu m1, [shuffle_mask]
pshufb m0, m1 ; embaralhar m0 baseado em m1
```

Note que -1, para fácil leitura, é usado como o índice de embaralhamento para zerar o byte de saída: ```-1``` como byte é o campo binário ```0b11111111``` (complemento de dois), e assim o bit mais significativo (MSB) (0x80) está ativado.

[image1]: <data:image/png;base64,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>