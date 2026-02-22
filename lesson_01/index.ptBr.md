**FFmpeg Lição Um de Linguagem Assembly**

**Introdução**

Bem-vindo à Escola de Linguagem Assembly do FFmpeg. Você deu o primeiro passo na jornada mais interessante, desafiadora e recompensadora da programação. Estas lições fornecerão uma base na forma como a linguagem assembly é escrita no FFmpeg e abrirão seus olhos para o que realmente está acontecendo em seu computador.

**Conhecimento Necessário**

* Conhecimento de C, em particular ponteiros. Se você não sabe C, estude o livro [The C Programming Language](https://en.wikipedia.org/wiki/The_C_Programming_Language)
* Matemática do Ensino Médio (escalar vs vetor, adição, multiplicação etc)

**O que é linguagem assembly?**

Linguagem assembly é uma linguagem de programação onde você escreve código que corresponde diretamente às instruções que uma CPU processa. A linguagem assembly legível por humanos é, como o nome sugere, *montada* em dados binários, conhecidos como *código de máquina*, que a CPU pode entender. Você pode ver o código de linguagem assembly ser referido como “assembly” ou “asm” para abreviar.

A vasta maioria do código assembly no FFmpeg é o que se conhece como *SIMD, Single Instruction Multiple Data* (Instrução Única, Múltiplos Dados). SIMD é às vezes referido como programação vetorial. Isso significa que uma instrução particular opera em múltiplos elementos de dados ao mesmo tempo. A maioria das linguagens de programação opera em um elemento de dados por vez, conhecido como programação escalar.

Como você deve ter adivinhado, SIMD se presta bem ao processamento de imagens, vídeo e áudio, que possuem muitos dados ordenados sequencialmente na memória. Existem instruções especializadas disponíveis na CPU para nos ajudar a processar dados sequenciais.

No FFmpeg, você verá os termos “função assembly”, “SIMD” e “vetor(izar)” usados ​​de forma intercambiável. Todos eles se referem à mesma coisa: Escrever uma função em linguagem assembly manualmente para processar múltiplos elementos de dados de uma só vez. Alguns projetos também podem se referir a eles como “kernels assembly”.

Tudo isso pode parecer complicado, mas é importante lembrar que no FFmpeg, estudantes do ensino médio já escreveram código assembly. Como em tudo, aprender é 50% jargão e 50% aprendizado real.

**Por que escrevemos em linguagem assembly?**
Para tornar o processamento multimídia rápido. É muito comum obter uma melhoria de velocidade de 10x ou mais ao escrever código assembly, o que é especialmente importante ao querer reproduzir vídeos em tempo real sem travamentos. Também economiza energia e estende a vida útil da bateria. Vale a pena ressaltar que as funções de codificação e decodificação de vídeo são algumas das funções mais usadas na Terra, tanto por usuários finais quanto por grandes empresas em seus datacenters. Então, mesmo uma pequena melhoria se soma rapidamente.

Você frequentemente verá, online, pessoas usando *intrinsics*, que são funções semelhantes a C que mapeiam para instruções assembly para permitir um desenvolvimento mais rápido. No FFmpeg, não usamos intrinsics, mas sim escrevemos código assembly manualmente. Esta é uma área de controvérsia, mas intrinsics são tipicamente cerca de 10-15% mais lentas do que assembly escrito à mão (defensores de intrinsics discordariam), dependendo do compilador. Para o FFmpeg, cada bit de desempenho extra ajuda, razão pela qual escrevemos código assembly diretamente. Há também um argumento de que intrinsics são difíceis de ler devido ao uso da “[Notação Húngara](https://en.wikipedia.org/wiki/Hungarian_notation)”.

Você também pode ver *inline assembly* (ou seja, sem usar intrinsics) permanecendo em alguns lugares no FFmpeg por razões históricas, ou em projetos como o Linux Kernel devido a casos de uso muito específicos lá. Isso é quando o código assembly não está em um arquivo separado, mas escrito em linha com o código C. A opinião predominante em projetos como o FFmpeg é que esse código é difícil de ler, não amplamente suportado por compiladores e de difícil manutenção.

Por fim, você verá muitos autoproclamados especialistas online dizendo que nada disso é necessário e que o compilador pode fazer toda essa “vetorização” para você. Pelo menos para fins de aprendizado, ignore-os: testes recentes em, por exemplo, [o projeto dav1d](https://www.videolan.org/projects/dav1d.html) mostraram um ganho de velocidade de cerca de 2x com essa vetorização automática, enquanto as versões escritas à mão podiam atingir 8x.

**Tipos de linguagem assembly**
Estas lições focarão na linguagem assembly x86 de 64 bits. Esta também é conhecida como amd64, embora ainda funcione em CPUs Intel. Existem outros tipos de assembly para outras CPUs como ARM e RISC-V e, potencialmente no futuro, estas lições serão estendidas para cobrir essas.

Existem dois tipos de sintaxe assembly x86 que você verá online: AT&T e Intel. A Sintaxe AT&T é mais antiga e mais difícil de ler em comparação com a sintaxe Intel. Portanto, usaremos a sintaxe Intel.

**Materiais de apoio**
Você pode se surpreender ao saber que livros ou recursos online como o Stack Overflow não são particularmente úteis como referências. Isso se deve em parte à nossa escolha de usar assembly escrito à mão com sintaxe Intel. Mas também porque muitos recursos online são focados em programação de sistema operacional ou programação de hardware, geralmente usando código não-SIMD. O assembly do FFmpeg é particularmente focado no processamento de imagens de alto desempenho e, como você verá, é uma abordagem particularmente única para a programação assembly. Dito isso, é fácil entender outros casos de uso de assembly depois de concluir estas lições.

Muitos livros entram em muitos detalhes de arquitetura de computador antes de ensinar assembly. Isso é bom se é isso que você quer aprender, mas do nosso ponto de vista, é como estudar motores antes de aprender a dirigir um carro.

Dito isso, os diagramas nas partes posteriores do livro “The Art of 64-bit assembly” mostrando instruções SIMD e seu comportamento de forma visual são úteis: [https://artofasm.randallhyde.com/](https://artofasm.randallhyde.com/)

Um servidor Discord está disponível para responder perguntas:
[https://discord.com/invite/Ks5MhUhqfB](https://discord.com/invite/Ks5MhUhqfB)

**Registradores**
Registradores são áreas na CPU onde os dados podem ser processados. As CPUs não operam diretamente na memória, mas os dados são carregados em registradores, processados e gravados de volta na memória. Em linguagem assembly, geralmente, você não pode copiar dados diretamente de um local de memória para outro sem primeiro passar esses dados por um registrador.

**Registradores de Propósito Geral**
O primeiro tipo de registrador é o que se conhece como Registrador de Propósito Geral (GPR). Os GPRs são chamados de propósito geral porque podem conter dados, neste caso até um valor de 64 bits, ou um endereço de memória (um ponteiro). Um valor em um GPR pode ser processado através de operações como adição, multiplicação, deslocamento, etc.

Na maioria dos livros sobre assembly, há capítulos inteiros dedicados às sutilezas dos GPRs, o histórico, etc. Isso ocorre porque os GPRs são importantes quando se trata de programação de sistema operacional, engenharia reversa, etc. No código assembly escrito no FFmpeg, os GPRs são mais como andaimes e na maioria das vezes suas complexidades não são necessárias e são abstraídas.

**Registradores vetoriais**
Registradores vetoriais (SIMD), como o nome sugere, contêm múltiplos elementos de dados. Existem vários tipos de registradores vetoriais:

* mm registers - Registradores MMX, de 64 bits, históricos e pouco usados atualmente
* xmm registers - Registradores XMM, de 128 bits, amplamente disponíveis
* ymm registers - Registradores YMM, de 256 bits, algumas complicações ao usá-los
* zmm registers - Registradores ZMM, de 512 bits, disponibilidade limitada

A maioria dos cálculos em compressão e descompressão de vídeo são baseados em inteiros, então vamos nos ater a isso. Aqui está um exemplo de 16 bytes em um registrador xmm:

| a | b | c | d | e | f | g | h | i | j | k | l | m | n | o | p |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Mas poderia ser oito words (inteiros de 16 bits)

| a | b | c | d | e | f | g | h |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Ou quatro double words (inteiros de 32 bits)

| a | b | c | d |
| :---- | :---- | :---- | :---- |

Ou dois quadwords (inteiros de 64 bits):

| a | b |
| :---- | :---- |

Para recapitular:

* **b**ytes - dados de 8 bits
* **w**ords - dados de 16 bits
* **d**oublewords - dados de 32 bits
* **q**uadwords - dados de 64 bits
* **d**ouble **q**uadwords - dados de 128 bits

Os caracteres em negrito serão importantes mais tarde.

**x86inc.asm include**
Você verá em muitos exemplos que incluímos o arquivo x86inc.asm. X86inc.asm é uma camada de abstração leve usada no FFmpeg, x264 e dav1d para facilitar a vida de um programador assembly. Ajuda de muitas maneiras, mas para começar, uma das coisas úteis que faz é rotular GPRs, r0, r1, r2. Isso significa que você não precisa memorizar nenhum nome de registrador. Como mencionado antes, os GPRs são geralmente apenas andaimes, então isso torna a vida muito mais fácil.

**Um simples trecho escalar de asm**

Vamos ver um trecho simples (e muito artificial) de asm escalar (código assembly que opera em itens de dados individuais, um por vez, dentro de cada instrução) para ver o que está acontecendo:

```assembly
mov  r0q, 3
inc  r0q
dec  r0q
imul r0q, 5
```

Na primeira linha, o *valor imediato* 3 (um valor armazenado diretamente no próprio código assembly, ao contrário de um valor buscado da memória) está sendo armazenado no registrador r0 como um quadword. Note que na sintaxe Intel, o operando de origem (o valor ou local que fornece os dados, localizado à direita) é transferido para o operando de destino (o local que recebe os dados, localizado à esquerda), muito parecido com o comportamento de memcpy. Você também pode lê-lo como “r0q = 3”, já que a ordem é a mesma. O sufixo “q” de r0 designa o registrador como sendo usado como um quadword. inc incrementa o valor para que r0q contenha 4, dec decrementa o valor de volta para 3. imul multiplica o valor por 5. Então, no final, r0q contém 15.

Observe que as instruções legíveis por humanos, como mov e inc, que são montadas em código de máquina pelo montador, são conhecidas como *mnemônicos*. Você pode ver online e em livros mnemônicos representados com letras maiúsculas como MOV e INC, mas estas são as mesmas que as versões em minúsculas. No FFmpeg, usamos mnemônicos em minúsculas e mantemos as maiúsculas reservadas para macros.

**Compreendendo uma função vetorial básica**

Aqui está nossa primeira função SIMD:

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

Vamos analisar linha por linha:

```assembly
%include "x86inc.asm"
```

Este é um “cabeçalho” desenvolvido nas comunidades x264, FFmpeg e dav1d para fornecer ajudantes, nomes predefinidos e macros (como cglobal abaixo) para simplificar a escrita assembly.

```assembly
SECTION .text
```

Isso denota a seção onde o código que você deseja executar é colocado. Isso contrasta com a seção .data, onde você pode colocar dados constantes.

```assembly
;static void add_values(uint8_t *src, const uint8_t *src2)
INIT_XMM sse2
```

A primeira linha é um comentário (o ponto e vírgula “;” em asm é como “//” em C) mostrando a aparência do argumento da função em C. A segunda linha mostra como estamos inicializando a função para usar registradores XMM, usando o conjunto de instruções sse2. Isso ocorre porque paddb é uma instrução sse2. Abordaremos sse2 em mais detalhes na próxima lição.

```assembly
cglobal add_values, 2, 2, 2, src, src2
```

Esta é uma linha importante, pois define uma função C chamada “add_values”.

Vamos analisar cada item um por vez:

* O próximo parâmetro mostra que ele tem dois argumentos de função.
* O parâmetro seguinte mostra que usaremos dois GPRs para os argumentos. Em alguns casos, podemos querer usar mais GPRs, então precisamos dizer ao x86util que precisamos de mais.
* O parâmetro seguinte informa ao x86util quantos registradores XMM usaremos.
* Os dois parâmetros seguintes são rótulos para os argumentos da função.

Vale a pena notar que o código mais antigo pode não ter rótulos para os argumentos da função, mas em vez disso endereçar GPRs diretamente usando r0, r1 etc.

```assembly
    movu  m0, [srcq]
    movu  m1, [src2q]
```

movu é uma abreviação para movdqu (move double quad unaligned). O alinhamento será abordado em outra lição, mas por enquanto movu pode ser tratado como um movimento de 128 bits de [srcq]. No caso de mov, os colchetes significam que o endereço em [srcq] está sendo desreferenciado, o equivalente a `*src` em C. Isso é o que se conhece como uma carga (load). Observe que o sufixo “q” refere-se ao tamanho do ponteiro (*i.e.* em C, ele representa `sizeof(*src) == 8` em sistemas de 64 bits, e x86asm é inteligente o suficiente para usar 32 bits em sistemas de 32 bits), mas a carga subjacente é de 128 bits.

Observe que não nos referimos a registradores vetoriais pelo nome completo, neste caso xmm0, mas como m0, uma forma abstraída. Em futuras lições, você verá como isso significa que você pode escrever o código uma vez e fazer com que ele funcione em vários tamanhos de registradores SIMD.

```assembly
paddb m0, m1
```

paddb (leia isso mentalmente como *p-add-b*) está adicionando cada byte em cada registrador, conforme mostrado abaixo. O prefixo “p” significa “packed” e é usado para identificar instruções vetoriais versus instruções escalares. O sufixo “b” mostra que esta é uma adição bytewise (adição de bytes).

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

Isso é o que se conhece como um store (armazenamento). Os dados são gravados de volta no endereço no ponteiro srcq.

```assembly
RET
```

Esta é uma macro para denotar que a função retorna. Praticamente todas as funções assembly no FFmpeg modificam os dados nos argumentos, em vez de retornar um valor.

Como você verá na atribuição, criamos ponteiros de função para funções assembly e os usamos onde disponíveis.

[Próxima Lição](../lesson_02/index.md)
