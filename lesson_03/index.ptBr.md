**FFmpeg Assembly: Lição Três**

Vamos explicar mais alguns termos técnicos e dar-lhe uma pequena lição de história.

**Conjuntos de Instruções**

Você deve ter visto na lição anterior que falamos sobre SSE2, que é um conjunto de instruções SIMD. Quando uma nova geração de CPU é lançada, ela pode vir com novas instruções e, às vezes, com tamanhos de registradores maiores. A história do conjunto de instruções x86 é muito complexa, então esta é uma história simplificada (existem muitas outras subcategorias):

*   MMX - Lançado em 1997, primeiro SIMD em processadores Intel, registradores de 64 bits, histórico
*   SSE (Streaming SIMD Extensions) - Lançado em 1999, registradores de 128 bits
*   SSE2 - Lançado em 2000, muitas novas instruções
*   SSE3 - Lançado em 2004, primeiras instruções horizontais
*   SSSE3 (Supplemental SSE3) - Lançado em 2006, novas instruções, mas o mais importante é a instrução pshufb shuffle, indiscutivelmente a instrução mais importante no processamento de vídeo
*   SSE4 - Lançado em 2008, muitas novas instruções, incluindo mínimo e máximo empacotados.
*   AVX - Lançado em 2011, registradores de 256 bits (somente float) e nova sintaxe de três operandos
*   AVX2 - Lançado em 2013, registradores de 256 bits para instruções inteiras
*   AVX512 - Lançado em 2017, registradores de 512 bits, novo recurso de máscara de operação. Estes tiveram uso limitado na época no FFmpeg devido à redução da frequência da CPU quando novas instruções eram usadas. Shuffle completo de 512 bits (permute) com vpermb.
*   AVX512ICL - Lançado em 2019, sem mais redução da frequência do clock.
*   AVX10 - Próximo

Vale a pena notar que os conjuntos de instruções podem ser removidos, assim como adicionados às CPUs. Por exemplo, o AVX512 foi [removido](https://www.igorslab.de/en/intel-deactivated-avx-512-on-alder-lake-but-fully-questionable-interpretation-of-efficiency-news-editorial/), de forma controversa, nas CPUs Intel de 12ª Geração. É por essa razão que o FFmpeg faz detecção de CPU em tempo de execução. O FFmpeg detecta as capacidades da CPU em que está sendo executado.

Como você viu na tarefa, os ponteiros de função são C por padrão e são substituídos por uma variante de conjunto de instruções específica. Isso significa que a detecção é feita uma vez e nunca mais precisa ser feita. Isso contrasta com muitos aplicativos proprietários que codificam um conjunto de instruções específico, tornando um computador perfeitamente funcional obsoleto. Isso também permite que funções otimizadas sejam ativadas/desativadas em tempo de execução. Este é um dos grandes benefícios do código aberto.

Programas como o FFmpeg são usados em bilhões de dispositivos em todo o mundo, alguns dos quais podem ser muito antigos. O FFmpeg tecnicamente suporta máquinas que suportam apenas SSE, que têm 25 anos! Felizmente, o x86inc.asm é capaz de informar se você usa uma instrução que não está disponível em um determinado conjunto de instruções.

Para lhe dar uma ideia das capacidades do mundo real, aqui está a disponibilidade do conjunto de instruções do [Steam Survey](https://store.steampowered.com/hwsurvey/Steam-Hardware-Software-Survey-Welcome-to-Steam) em novembro de 2024 (isso é obviamente tendencioso para jogadores):

| Conjunto de Instruções | Disponibilidade |
| :---- | :---- |
| SSE2 | 100% |
| SSE3 | 100% |
| SSSE3 | 99.86% |
| SSE4.1 | 99.80% |
| AVX | 97.39% |
| AVX2 | 94.44% |
| AVX512 (Steam não diferencia entre AVX512 e AVX512ICL) | 14.09% |

Para um aplicativo como o FFmpeg, com bilhões de usuários, mesmo 0,1% representa um número muito grande de usuários e relatórios de bugs se algo quebrar. O FFmpeg possui uma extensa infraestrutura de testes para testar as variações de CPU/OS/Compiler em nossa [testsuite FATE](https://fate.ffmpeg.org/?query=subarch:x86_64%2F%2F). Cada commit é executado em centenas de máquinas para garantir que nada quebre.

A Intel fornece um manual detalhado do conjunto de instruções aqui: [https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)

Pode ser complicado pesquisar em um PDF, então há uma alternativa não oficial baseada na web aqui: [https://www.felixcloutier.com/x86/](https://www.felixcloutier.com/x86/)

Há também uma representação visual das instruções SIMD disponível aqui:
[https://www.officedaytime.com/simd512e/](https://www.officedaytime.com/simd512e/)

Parte do desafio da assembly x86 é encontrar a instrução certa para suas necessidades. Em alguns casos, as instruções podem ser usadas de uma forma para a qual não foram originalmente projetadas.

**Astúcia de deslocamento de ponteiro**

Vamos voltar à nossa função original da Lição 1, mas adicionar um argumento `width` à função C.

Usamos `ptrdiff_t` para a variável `width` em vez de `int` para garantir que os 32 bits superiores do argumento de 64 bits sejam zero. Se passássemos diretamente um `int width` na assinatura da função e depois tentássemos usá-lo como um `quad` para aritmética de ponteiros (ou seja, usando `widthq`), os 32 bits superiores do registrador poderiam ser preenchidos com valores arbitrários. Poderíamos corrigir isso estendendo o sinal de `width` com `movsxd` (veja também a macro `movsxdifnidn` em x86inc.asm), mas esta é uma maneira mais fácil.

A função abaixo contém a astúcia de deslocamento de ponteiro:

```assembly
;static void add_values(uint8_t *src, const uint8_t *src2, ptrdiff_t width)
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

Vamos analisar isso passo a passo, pois pode ser confuso:

```assembly
   add srcq, widthq
   add src2q, widthq
   neg widthq
```

A `width` é adicionada a cada ponteiro de forma que cada ponteiro agora aponte para o final do buffer a ser processado. A `width` é então negada.

```assembly
    movu  m0, [srcq+widthq]
    movu  m1, [src2q+widthq]
```

As cargas são então feitas com `widthq` sendo negativo. Assim, na primeira iteração, `[srcq+widthq]` aponta para o endereço original de `srcq`, ou seja, aponta de volta para o início do buffer.

```assembly
    add   widthq, mmsize
    jl .loop
```

`mmsize` é adicionado ao `widthq` negativo, aproximando-o de zero. A condição de loop agora é `jl` (saltar se menor que zero). Este truque significa que `widthq` é usado como um deslocamento de ponteiro **e** como um contador de loop ao mesmo tempo, economizando uma instrução `cmp`. Também permite que o deslocamento do ponteiro seja usado em múltiplas cargas e armazenamentos, bem como o uso de múltiplos dos deslocamentos do ponteiro, se necessário (lembre-se disso para a tarefa).

**Alinhamento**

Em todos os nossos exemplos, temos usado `movu` para evitar o tópico de alinhamento. Muitas CPUs podem carregar e armazenar dados mais rapidamente se os dados estiverem alinhados, ou seja, se o endereço de memória for divisível pelo tamanho do registrador SIMD. Onde possível, tentamos usar cargas e armazenamentos alinhados no FFmpeg usando `mova`.

No FFmpeg, `av_malloc` é capaz de fornecer memória alinhada no heap e a diretiva de pré-processador C `DECLARE_ALIGNED` pode fornecer memória alinhada na stack. Se `mova` for usado com um endereço desalinhado, isso causará uma falha de segmentação e o aplicativo travará. Também é importante ter certeza de que o valor de alinhamento corresponde ao tamanho do registrador SIMD, ou seja, 16 para `xmm`, 32 para `ymm` e 64 para `zmm`.

Veja como alinhar o início da seção RODATA a 64 bytes:

```assembly
SECTION_RODATA 64
```

Observe que isso apenas alinha o início de RODATA. Bytes de preenchimento podem ser necessários para garantir que o próximo rótulo permaneça em um limite de 64 bytes.

**Expansão de faixa**

Outro tópico que evitamos até agora é o estouro (overflow). Isso acontece, por exemplo, quando o valor de um byte ultrapassa 255 após uma operação como adição ou multiplicação. Podemos querer realizar uma operação onde precisamos de um valor intermediário maior que um byte (por exemplo, palavras), ou potencialmente queremos deixar os dados nesse tamanho intermediário maior.

Para bytes sem sinal, é aqui que entram `punpcklbw` (packed unpack low bytes to words) e `punpckhbw` (packed unpack high bytes to words).

Vamos ver como `punpcklbw` funciona. A sintaxe para a versão SSE2 do Manual da Intel é a seguinte:

| PUNPCKLBW xmm1, xmm2/m128 |
| :---- |

Isso significa que sua origem (lado direito) pode ser um registrador `xmm` ou um endereço de memória (`m128` significa um endereço de memória com a sintaxe padrão `[base + scale*index + disp]`) e o destino um registrador `xmm`.

O site officedaytime.com acima tem um bom diagrama mostrando o que está acontecendo:

![O que é isso](image1.png)

Você pode ver que os bytes são intercalados da metade inferior de cada registrador, respectivamente. Mas o que isso tem a ver com a extensão de faixa? Se o registrador `src` for todo zero, isso intercalará os bytes em `dst` com zeros. Isso é o que é conhecido como *extensão de zero* (zero extension), pois os bytes não têm sinal. `punpckhbw` pode ser usado para fazer a mesma coisa para os bytes superiores.

Aqui está um trecho mostrando como isso é feito:

```assembly
pxor      m2, m2 ; zera m2

movu      m0, [srcq]
movu      m1, m0 ; faz uma cópia de m0 em m1
punpcklbw m0, m2
punpckhbw m1, m2
```

`m0` e `m1` agora contêm os bytes originais estendidos a palavras com zeros. Na próxima lição, você verá como as instruções de três operandos em AVX tornam o segundo `movu` desnecessário.

**Extensão de sinal**

Dados com sinal são um pouco mais complicados. Para estender a faixa de um inteiro com sinal, precisamos usar um processo conhecido como [extensão de sinal](https://en.wikipedia.org/wiki/Sign_extension). Isso preenche os MSBs com o bit de sinal. Por exemplo: -2 em `int8_t` é `0b11111110`. Para estendê-lo para `int16_t`, o MSB de 1 é repetido para formar `0b1111111111111110`.

`pcmpgtb` (packed compare greater than byte) pode ser usado para extensão de sinal. Ao fazer a comparação (`0 > byte`), todos os bits no byte de destino são definidos como 1 se o byte for negativo; caso contrário, os bits no byte de destino são definidos como 0. `punpckX` pode ser usado como acima para realizar a extensão de sinal. Se o byte for negativo, o byte correspondente é `0b11111111` e, caso contrário, é `0x00000000`. Intercalando o valor do byte com a saída de `pcmpgtb` resulta em uma extensão de sinal para a palavra.

```assembly
pxor      m2, m2 ; zera m2

movu      m0, [srcq]
movu      m1, m0 ; faz uma cópia de m0 em m1

pcmpgtb   m2, m0
punpcklbw m0, m2
punpckhbw m1, m2
```

Como você pode ver, há uma instrução extra em comparação com o caso sem sinal.

**Empacotamento**

`packuswb` (pack unsigned word to byte) e `packsswb` permitem converter de word para byte. Isso permite intercalar dois registradores SIMD contendo words em um registrador SIMD com um byte. Observe que se os valores excederem a faixa de bytes, eles serão saturados (ou seja, limitados ao maior valor).

**Shuffles**

Shuffles, também conhecidas como permutações, são indiscutivelmente as instruções mais importantes no processamento de vídeo e `pshufb` (packed shuffle bytes), disponível em SSSE3, é a variante mais importante.

Para cada byte, o byte de origem correspondente é usado como um índice do registrador de destino, exceto quando o MSB é definido, o byte de destino é zerado. É análogo ao seguinte código C (embora em SIMD todas as 16 iterações do loop aconteçam em paralelo):

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
pshufb m0, m1 ; shuffle m0 based on m1
```

Observe que -1 (para facilitar a leitura) é usado como índice de shuffle para zerar o byte de saída: -1 como um byte é o campo de bits `0b11111111` (complemento de dois), e assim o MSB (`0x80`) é definido.
