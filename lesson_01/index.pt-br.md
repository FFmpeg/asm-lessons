**Lição Um de Linguagem Assembly do FFmpeg**

**Introdução**

Bem-vindo à Escola de Linguagem Assembly do FFmpeg. Você deu o primeiro passo na jornada mais interessante, desafiadora e gratificante da programação. Estas lições fornecerão os fundamentos sobre como assembly é usado no FFmpeg e abrirão seus olhos para o que realmente acontece dentro do seu computador...

**Conhecimentos Necessários**

* Conhecimento em C, especialmente ponteiros. Se você não está familiarizado com C, você encontrará todos os fundamentos no livro [The C Programming Language](https://en.wikipedia.org/wiki/The_C_Programming_Language).
* Matemática do ensino médio (escalar vs vetor, adição, multiplicação, etc...)

**O que é Assembly?**

Assembly é uma linguagem de programação onde o código que você escreve corresponde diretamente a instruções compreensíveis por uma CPU. O assembly legível por humanos é, como o nome indica, *montado* em dados binários, conhecido como *linguagem de máquina*, que a CPU pode entender. O código assembly é frequentemente chamado de "assembly" ou "asm" abreviadamente.

A grande maioria do assembly no FFmpeg é o que chamamos de *SIMD, Single Instruction Multiple Data (instrução única, múltiplos dados)*. SIMD às vezes é referido como programação vetorial. Isso significa que uma instrução particular opera em múltiplos elementos de dados simultaneamente. A maioria das linguagens de programação processa um único elemento de dados por vez, o que é chamado de programação escalar.

Como você pode ter adivinhado, SIMD se presta bem ao processamento de imagens, vídeos e áudio, que contêm uma grande quantidade de dados organizados sequencialmente na memória. Instruções especializadas no processador nos ajudarão a processar esses dados sequenciais.

No FFmpeg, você verá que os termos `função assembly`, `SIMD`, `vetorização` são usados de forma intercambiável. Todos eles se referem à mesma coisa: escrever uma função em assembly manualmente para processar múltiplos elementos de dados de uma só vez. Alguns projetos também podem se referir a `kernels assembly`.

Tudo isso pode parecer complicado, mas é importante lembrar que estudantes do ensino médio escreveram assembly no FFmpeg. Como em qualquer lugar, aprender é 50% jargão e 50% aprendizado real.

**Por que escrevemos em Assembly?**

Para tornar o processamento multimídia rápido. É muito comum ter uma velocidade de processamento pelo menos 10 vezes mais rápida ao escrever em assembly, o que é ainda mais importante quando queremos reproduzir vídeos em tempo real sem travamentos. Isso também economiza energia e estende a vida útil das baterias. É importante destacar que as funções de codificação e decodificação estão entre as funções mais usadas no mundo, tanto por usuários finais quanto por multinacionais em seus data centers. Portanto, mesmo uma pequena melhoria traz muito benefício.

Você verá frequentemente, online, pessoas usando *funções intrínsecas*, funções que se parecem com C mas que são na verdade instruções assembly usadas para permitir desenvolvimento mais rápido. No FFmpeg, não usamos esse tipo de função, escrevemos todo o código assembly manualmente. Este é um ponto de discórdia, mas as funções intrínsecas são cerca de 10 a 15% mais lentas que o equivalente em assembly escrito manualmente (os defensores dessas funções discordariam), tudo depende do compilador. Para o FFmpeg, cada melhoria conta, por isso escrevemos todo o código diretamente em assembly. Um argumento a nosso favor é o uso da `[notação húngara](https://pt.wikipedia.org/wiki/Notação_húngara)` nas funções intrínsecas que complicam sua leitura.

Além disso, você verá referências a *assembly inline*, ou seja, não usando funções intrínsecas, em alguns lugares no FFmpeg por razões históricas, ou em projetos como o kernel Linux em cenários de uso muito específicos. Aqui, o código assembly não está em um arquivo separado, mas escrito diretamente em arquivos com código C. A visão majoritária em projetos como o FFmpeg é que esse código é difícil de ler, não amplamente suportado por compiladores e difícil de manter.

Finalmente, você verá muitos especialistas autoproclamados na internet dizendo que nada disso é necessário e que o compilador pode fazer essa `vetorização` para você. Para fins de aprendizado, ignore-os: testes recentes como os presentes no [projeto dav1d](https://www.videolan.org/projects/dav1d.html) mostraram um ganho de velocidade de cerca de 2x graças a essa vetorização automática, enquanto para versões escritas manualmente, esse ganho subiu para 8x.

**As Variedades de Assembly**

Estas lições se concentrarão no assembly x86 de 64 bits. Também conhecido como amd64, embora continue funcionando em CPUs Intel. Existem tantos tipos de assembly quanto CPUs, como aqueles para ARM ou RISC-V, com possíveis atualizações destes cursos para incluí-los.

Existem dois tipos de sintaxes para assembly x86 que você encontrará online: AT&T e Intel. A primeira é mais antiga e mais difícil de ler comparada à segunda. Nós nos concentraremos nesta última.

**Materiais de Suporte**

Você pode ficar surpreso ao saber que livros ou recursos online como Stack Overflow não são de grande ajuda como referências. Particularmente devido à nossa escolha de usar assembly escrito manualmente com sintaxe Intel. Mas também porque muitos desses recursos online se concentram em programação de sistemas operacionais ou programação para hardware, não usando código SIMD. O assembly do FFmpeg é principalmente focado no processamento de imagens de alto desempenho, e como você verá, com uma abordagem particular à programação assembly. Dito isso, é fácil entender outros casos de uso do assembly uma vez que essas lições sejam concluídas.

Muitos livros detalham muitas arquiteturas de computador diferentes antes de detalhar o assembly. Do nosso ponto de vista, isso é ótimo se é isso que você quer aprender, mas é como querer estudar motores antes de aprender a dirigir um carro.

Dito isso, nas partes seguintes, os diagramas do livro `The Art of 64-bit assembly` mostrando instruções SIMD e seu comportamento em forma visual serão muito úteis: [https://artofasm.randallhyde.com/](https://artofasm.randallhyde.com/)

Um servidor Discord está disponível para responder suas perguntas:
[https://discord.com/invite/Ks5MhUhqfB](https://discord.com/invite/Ks5MhUhqfB)

**Os Registradores**  

Registradores são áreas da CPU onde os dados podem ser processados. As CPUs não intervêm diretamente na memória, os dados são carregados em registradores, processados e depois escritos novamente na memória. Em assembly, de maneira geral, você não pode copiar diretamente os dados de um local de memória para outro sem primeiro passar esses dados por um registrador.

**Registradores de Uso Geral**

O primeiro tipo de registrador que encontraremos é conhecido como Registrador de Uso Geral (GPR). Os GPRs são chamados assim porque podem conter tanto dados, um valor de até 64 bits, quanto um endereço de memória (um ponteiro). Um valor em um GPR pode ser processado por operações como adição, multiplicação, deslocamento, etc...

Na maioria dos livros sobre assembly, muitos capítulos inteiros são dedicados às sutilezas dos GPRs, sua história, etc... Pois os GPRs desempenharam um papel importante na programação de sistemas operacionais, engenharia reversa, etc... No assembly escrito para o FFmpeg, os GPRs são considerados como andaimes e na maioria das vezes, suas complexidades não são necessárias e são abstraídas.

**Registradores Vetoriais**  
Os registradores vetoriais (SIMD), como o nome sugere, contêm múltiplos elementos de dados. Existem diferentes tipos de registradores vetoriais:

* registradores `mm`: registradores `MMX`, de tamanho 64 bits, históricos e pouco usados hoje em dia
* registradores `xmm`: registradores `XMM`, de tamanho 128 bits, amplamente disponíveis
* registradores `ymm`: registradores `YMM`, de tamanho 256 bits, com algumas complicações ao usá-los
* registradores `zmm`: registradores `ZMM`, de tamanho 512 bits, muito pouco disponíveis

A maioria dos cálculos de compressão e descompressão de vídeo são baseados em inteiros, nos limitaremos a eles. Aqui está um exemplo de um registrador `XMM` de 16 bytes:

| a | b | c | d | e | f | g | h | i | j | k | l | m | n | o | p |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Isso também poderia ser oito palavras (inteiros de 16 bits):
| a | b | c | d | e | f | g | h |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |

Ou quatro palavras duplas (inteiros de 32 bits):
| a | b | c | d |
| :---- | :---- | :---- | :---- |


Ou duas palavras quádruplas (inteiros de 64 bits):

| a | b |
| :---- | :---- |


Para recapitular:
* **b**ytes - dados de 8 bits
* **w**ords - dados de 16 bits
* **d**oublewords - dados de 32 bits 
* **q**uadwords - dados de 64 bits
* **d**ouble **q**uadwords - dados de 128 bits.

Os caracteres em negrito serão importantes na sequência.

**Header x86inc.asm**  

Em muitos exemplos, incluímos o arquivo x86inc.asm. x86inc.asm é uma camada de abstração leve usada no FFmpeg, x264 e dav1d para facilitar a vida dos desenvolvedores. Ela ajuda de muitas maneiras, mas uma das coisas úteis que permite é, entre outras, rotular os GPRs `r0`, `r1` e `r2`. Você não precisará se lembrar dos nomes dos registradores. Como mencionado anteriormente, os GPRs são geralmente apenas andaimes, então isso torna as coisas muito mais simples.

**Um Trecho Simples de Assembly Escalar**

Vamos olhar um trecho simples (e totalmente artificial) de código assembly escalar (código assembly que opera em elementos de dados individuais, um por vez, em cada instrução) para ver o que acontece:

```assembly
mov  r0q, 3  
inc  r0q  
dec  r0q  
imul r0q, 5
```

Na primeira linha, o *valor imediato* 3 (um valor armazenado diretamente no código assembly em si, ao contrário de um valor recuperado da memória) é armazenado no registrador `r0` como quadword. Note que na sintaxe Intel, o operando fonte (o valor ou localização fornecendo os dados, localizado à direita) é transferido para o operando de destino (a localização recebendo os dados, localizado à esquerda), um pouco como o comportamento de *memcpy*. Você também pode ler como `r0q = 3`, já que a ordem é a mesma. O sufixo `q` de `r0` designa o registrador como sendo usado como quadword. `inc` incrementa o valor para que `r0q` contenha 4, `dec` decrementa o valor para voltar a 3. `imul` multiplica o valor por 5. No final, `r0q` contém 15.

Note que as instruções legíveis por humanos, como mov e inc, que são montadas em código de máquina pelo assembler, são chamadas *mnemônicos*. Você pode ver online e em livros mnemônicos representados com letras maiúsculas como `MOV` e `INC`, mas eles são os mesmos que as versões em minúsculas. No FFmpeg, usamos mnemônicos em minúsculas e reservamos maiúsculas para macros.

**Entendendo uma Função Vetorial Básica**

Aqui está nossa primeira função SIMD:

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

Vamos revisar o código linha por linha:

```assembly
%include "x86inc.asm"
```

Este `header` desenvolvido nas comunidades x264, FFmpeg e dav1d para fornecer auxiliares, nomes predefinidos e macros (como `cglobal` abaixo) para simplificar a escrita do código assembly.

```assembly
SECTION .text
```
Isso designa a seção onde o código que você quer executar é colocado. Isso contrasta com a seção `.data`, onde você pode colocar dados constantes.

```assembly
;static void add_values(const uint8_t *src, const uint8_t *src2);  
INIT_XMM sse2
```

A primeira linha é um comentário (o ponto e vírgula `;` em asm é o equivalente ao `//` em C) mostrando como é o protótipo da função em C. A segunda linha mostra como inicializamos a função para que ela use um registrador XMM, usando o conjunto de instruções sse2. Isso é necessário porque a instrução paddb faz parte do sse2. Abordaremos o sse2 com mais detalhes na próxima lição.

```assembly
cglobal add_values, 2, 2, 2, src, src2
```

O código anterior mostra uma linha importante: a definição de uma função em C chamada `add_values`.

Vamos revisar cada elemento da linha um por um:

* O parâmetro seguinte indica que a função recebe dois argumentos (o primeiro 2).
* Em seguida, o segundo 2 indica que vamos usar dois GPRs para os argumentos. Em alguns casos, poderíamos querer usar mais GPRs, então precisaríamos indicar ao x86util que precisaremos de mais.
* O parâmetro seguinte indica ao x86util quantos registradores `XMM` vamos usar.
* Os dois últimos parâmetros são os rótulos usados para os argumentos da função `add_values`.

É importante notar que o código mais antigo pode não ter os rótulos como argumentos de funções, mas sim os endereços dos registradores GPR no lugar, usando `r0`, `r1`, etc...

```assembly
    movu  m0, [srcq]  
    movu  m1, [src2q]
```

`movu` é uma abreviação de `movdqu` (*move double quad unaligned*). O alinhamento será abordado em outra lição, mas por enquanto, você pode considerar movu como uma transferência de 128 bits de [srcq]. No caso de mov, os colchetes `[]` significam que o endereço contido em [srcq] é desreferenciado, o que equivale a **src* em C. Isso é chamado de carregamento (load).

O sufixo `q` se refere ao tamanho do ponteiro. Em C, isso corresponde a sizeof(*src) == 8 em sistemas de 64 bits. O assembler x86 é inteligente o suficiente para usar 32 bits em sistemas de 32 bits, mas a operação de carregamento subjacente permanece de 128 bits.

Note que não fazemos referência aos registradores vetoriais por seu nome completo, neste caso `xmm0`, mas sim em uma forma abstrata como `m0`. Nas próximas lições, você verá como essa abordagem permite escrever código uma única vez e fazê-lo funcionar com múltiplos tamanhos de registradores SIMD.

```assembly
paddb m0, m1
```

`paddb` (leia isso *p-add-b*) adiciona cada byte em cada registrador, como ilustrado abaixo. O prefixo `p` significa `packed` e serve para distinguir instruções vetoriais de instruções escalares. O sufixo `b` indica que a operação é realizada no nível de bytes (adição de bytes).

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

Isso é chamado de armazenamento (*store*). Os dados são escritos na memória no endereço apontado por `srcq`.

```assembly
RET
```

`RET` é uma macro indicando que a função termina. Quase todas as funções assembly no FFmpeg modificam os dados passados como argumento em vez de retornar um valor.

Como você verá no exercício, criaremos ponteiros de função para funções assembly e os usaremos quando possível.

[Próxima lição](../lesson_02/index.pt-br.md)