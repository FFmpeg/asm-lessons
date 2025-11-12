**FFmpeg 어셈블리 3강**

몇 가지 추가 용어를 설명하고, 짧은 역사 이야기를 해보겠습니다.

**명령어 집합 (Instruction Sets)**

이전 강의에서 언급했듯이, SSE2는 SIMD 명령어 집합 중 하나입니다. 새로운 CPU 세대가 출시될 때마다 새로운 명령어나 더 큰 레지스터 크기가 추가되기도 합니다. x86 명령어 집합의 역사는 매우 복잡하므로, 여기서는 단순화된 형태로 설명합니다. (세부 하위 분류는 훨씬 더 많습니다)

* MMX - 1997년 출시, 인텔 프로세서에서 최초의 SIMD, 64비트 레지스터, 현재는 역사적인 기술
* SSE (Streaming SIMD Extensions) - 1999년 출시, 128비트 레지스터
* SSE2 - 2000년 출시, 다수의 새로운 명령어 추가
* SSE3 - 2004년 출시, 최초의 수평(horizontal) 명령어 포함
* SSSE3 (Supplemental SSE3) - 2006년 출시, 새로운 명령어 추가, 특히 pshufb 셔플 명령어 도입, 비디오 처리에서 가장 중요한 명령어 중 하나로 평가됨
* SSE4 - 2008년 출시, 최소/최대 연산 등 다수의 새로운 명령어 포함
* AVX - 2011년 출시, 256비트 레지스터(부동소수점 전용), 3-피연산자 문법 도입
* AVX2 - 2013년 출시, 정수 명령어용 256비트 레지스터
* AVX512 - 2017년 출시, 512비트 레지스터, 새로운 연산 마스크 기능 추가. 당시 FFmpeg에서는 새로운 명령어 사용 시 CPU 클럭이 하락하는 문제로 제한적으로 사용됨. vpermb를 통한 완전한 512비트 셔플(퍼뮤트) 지원.
* AVX512ICL - 2019년 출시, 클럭 하락 문제 해결.
* AVX10 - 출시 예정.

명령어 집합은 추가될 뿐만 아니라 제거될 수도 있습니다. 예를 들어, AVX512는 인텔 12세대 CPU에서 [제거되었습니다](https://www.igorslab.de/en/intel-deactivated-avx-512-on-alder-lake-but-fully-questionable-interpretation-of-efficiency-news-editorial/). 이 때문에 FFmpeg은 실행 중인 CPU의 기능을 자동으로 감지합니다.

과제에서 본 것처럼, 함수 포인터는 기본적으로 C로 정의되어 있으며 특정 명령어 집합 버전으로 대체됩니다. 이 과정은 한 번만 수행되며 이후에는 다시 감지할 필요가 없습니다. 이는 특정 명령어 집합을 하드코딩해 하드웨어 호환성을 잃는 상용프로그램들과 달리, FFmpeg이 오래된 시스템에서도 동작할 수 있게 해줍니다. 또한 런타임에서 최적화된 함수를 켜거나 끌 수 있다는 장점도 있습니다. 이것이 바로 오픈소스의 큰 이점 중 하나입니다.

FFmpeg은 전 세계의 수십억 대의 기기에서 사용되며, 그중에는 매우 오래된 장치들도 있습니다. 기술적으로 FFmpeg은 SSE만 지원하는 시스템에서도 동작하며, 이는 약 25년 된 하드웨어입니다. 다행이도 x86inc.asm은 특정 명령어 집합에서 사용 불가능한 명령어를 사용할 경우 경고를 제공합니다.

실제 환경의 예시로 2024년 11월 기준 [Steam 하드웨어 설문](https://store.steampowered.com/hwsurvey/Steam-Hardware-Software-Survey-Welcome-to-Steam)에서의 명령어 집합 지원 현황은 다음과 같습니다 (게이머 중심 데이터이므로 약간의 편향이 있습니다).

| 명령어 집합 | 지원 비율 |
| :---- | :---- |
| SSE2 | 100% |
| SSE3 | 100% |
| SSSE3 | 99.86% |
| SSE4.1 | 99.80% |
| AVX | 97.39% |
| AVX2 | 94.44% |
| AVX512 (Steam에서는 AVX512와 AVX512ICL을 구분하지 않음) | 14.09% |

FFmpeg처럼 전 세계 수십억 명이 사용하는 프로그램에서는 0.1%의 사용자라도 문제가 생기면 매우 큰 숫자가 됩니다. 이 때문에 FFmpeg은 다양한 CPU/OS/컴파일러 조합을 테스트하는 [FATE 테스트 시스템](https://fate.ffmpeg.org/?query=subarch:x86_64%2F%2F)을 보유하고 있습니다. 모든 커밋은 수백 대의 머신에서 자동으로 실행되어 문제가 없는지 검증됩니다.

인텔의 공식 명령어 집합 매뉴얼은 다음에서 확인할 수 있습니다: [https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)

PDF로 검색하기 번거롭다면 비공식 웹 버전도 있습니다: [https://www.felixcloutier.com/x86/](https://www.felixcloutier.com/x86/)

또한 SIMD 명령어를 시각적으로 표현한 자료도 있습니다: [https://www.officedaytime.com/simd512e/](https://www.officedaytime.com/simd512e/)

x86 어셈블리의 어려움 중 하나는 자신의 필요에 맞는 올바른 명령어를 찾는 것입니다. 때로는 명령어가 원래 의도된 용도와 다르게 사용되기도 합니다.

**포인터 오프셋 트릭 (Pointer offset trickery)**

1강에서 다뤘던 원래의 함수를 다시 살펴보되, 이번에는 C 함수에 width 인자를 추가해보겠습니다.

width 변수는 int 대신 ptrdiff_t 타입을 사용합니다. 이렇게 하는 64비트 인자의 상위 32비트가 반드시 0이 되도록 보장하기 위해서입니다. 만약 함수 시그니처에서 int width를 그대로 전달하고, 이후 포인터 연산에서 `widthq`로 사용한다면, 레지스터의 상위 32비트가 임의의 값으로 채워질 수 있습니다. 물론 `movsxd` 명령(또는 x86inc.asm의 `movsxdifnidn` 매크로)을 사용해 부호 확장(sign extend)으로 수정할 수도 있지만, 이 방법이 더 간단한 해결책입니다.

다음 함수는 포인터 오프셋 트릭을 포함하고 있습니다.

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

이제 각 단계를 순서대로 살펴보겠습니다.

```assembly
   add srcq, widthq
   add src2q, widthq
   neg widthq
```

width 값이 각 포인터에 더해져서, 이제 각각의 포인터는 처리할 버퍼의 끝을 가리키게 됩니다. 그 후 width를 음수로 바꿉니다.

```assembly
    movu  m0, [srcq+widthq]
    movu  m1, [src2q+widthq]
```

이제 widthq가 음수인 상태로 데이터를 로드합니다. 즉, 첫 번째 반복에서는 [srcq+widthq]가 원래의 srcq 주소(버퍼의 시작점)를 가리키게 됩니다.

```assembly
    add   widthq, mmsize
    jl .loop
```

mmsize를 음수인 widthq에 더해주어 0에 점점 가까워지게 만듭니다. 루프 조건은 jl(0보다 작으면 점프)입니다. 이 트릭의 핵심은 widthq를 포인터 오프셋이자 루프 카운터로 동시에 사용하여 cmp 명령어를 절약하는 것입니다. 또한 widthq를 여러 번 로드/스토어 연산에 재사용할 수 있고, 필요하다면 포인터 오프셋의 배수를 계산하는 데에도 활용할 수 있습니다. (이 부분은 과제에서 중요하게 사용될 것입니다.)

**메모리 정렬 (Alignment)**

지금까지의 모든 예제에서는 정렬 문제를 피하기 위해 movu를 사용했습니다. 대부분의 CPU는 데이터가 정렬되어 있을 때(즉, 메모리 주소가 SIMD 레지스터 크기로 나누어떨어질 때) 데이터를 더 빠르게 로드하고 저장할 수 있습니다. 따라서 가능한 경우 FFmpeg에서는 mova를 사용해 정렬된 로드와 스토어를 수행하려고 합니다.

FFmpeg에서는 av_malloc을 통해 힙(heap)에 정렬된 메모리를 할당할 수 있으며, C 전처리기 지시문인 DECLARE_ALIGNED를 사용하면 스택(stack)에 정렬된 메모리를 확보할 수 있습니다. 만약 mova를 정렬되지 않은 주소에 사용하면 세그멘테이션 폴트(segmentation fault)가 발생하여 프로그램이 크래시합니다.

또한 정렬 크기(alignment value)가 SIMD 레지스터 크기와 일치해야 합니다. 즉, xmm은 16바이트, ymm은 32바이트, zmm은 64바이트입니다.

다음은 RODATA 섹션의 시작을 64바이트에 맞춰 정렬하는 방법입니다.

```assembly
SECTION_RODATA 64
```

이 명령은 RODATA의 시작 부분만 정렬합니다. 다음 레이블(label)이 64바이트 경계에 있도록 하려면 패딩 바이트(padding bytes)를 추가해야 할 수도 있습니다.

**범위 확장 (Range expansion)**

지금까지 다루지 않았던 또 하나의 주제는 오버플로(overflow)입니다. 예를 들어, 어떤 바이트 값이 덧셈이나 곱셈 연산 후 255를 초과하면 오버플로가 발생합니다. 이때 중간 계산값이 바이트보다 큰 크기(예: 워드)가 필요할 수 있으며, 혹은 데이터를 그 더 큰 중간 크기 상태로 유지하고 싶을 수도 있습니다.

부호 없는 바이트(unsigned byte)의 경우, 이러한 상황에서 punpcklbw(packed unpack low bytes to words)와 punpckhbw(packed unpack high bytes to words) 명령어를 사용할 수 있습니다.

punpcklbw의 작동 방식을 살펴보겠습니다. Intel 매뉴얼에 따르면 SSE2 버전의 문법은 다음과 같습니다.

| PUNPCKLBW xmm1, xmm2/m128 |
| :---- |

이는 오른쪽 피연산자(소스)가 xmm 레지스터이거나 메모리 주소(m128, 즉, [base + scale*index + disp] 형태)일 수 있으며, 왼쪽 피연산자(대상)는 xmm 레지스터라는 의미입니다.

위에 소개한 officedaytime.com 웹사이트에는 이 동작을 시각적으로 보여주는 좋은 다이어그램이 있습니다.

![What is this](image1.png)

여기서 각 레지스터의 하위 절반에서 바이트들이 서로 교차(interleave)되어 배치되는 것을 볼 수 있습니다. 그렇다면 이것이 "범위 확장(range extension)"과 어떤 관련이 있을까요? 만약 src 레지스터가 전부 0이라면, dst의 바이트들은 0과 교차되어 합쳐집니다. 이는 바이트가 부호 없는 값이므로, 바이트를 *제로 확장(zero extension)* 하는 과정이 됩니다. punpckhbw 명령은 상위 바이트 영역에 대해서 동일한 작업을 수행합니다.

다음은 이 과정을 보여주는 코드 예시입니다.

```assembly
pxor      m2, m2 ; zero out m2

movu      m0, [srcq]
movu      m1, m0 ; make a copy of m0 in m1
punpcklbw m0, m2
punpckhbw m1, m2
```

이제 ```m0```와 ```m1```은 원래의 바이트 데이터를 워드 단위로 제로확장된 형태로 포함하고 있습니다. 다음 강의에서는 AVX의 3-피연산자 명령어를 통해 이 두 번째 movu 명령이 필요 없어지는 방법을 배우게 될 것입니다.

**부호 확장 (Sign extension)**

부호 있는 데이터(signed data)는 조금 더 복잡합니다. 부호 있는 정수를 확장하려면 [부호 확장(sign extension)](https://en.wikipedia.org/wiki/Sign_extension)이라는 과정을 사용해야 합니다. 이는 최상위 비트(MSB)를 부호 비트로 채우는 방식입니다. 예를 들어, int8_t인 -2는 0b11111110입니다. 이를 int16_t로 부호 확장하면, 최상위 비트 1이 반복되어 0b1111111111111110이 됩니다.

```pcmpgtb```(packed compare greater than byte) 명령어를 이용해 부호 확장을 수행할 수 있습니다. 이 명령은 (0 > byte) 비교를 수행하여, 바이트 값이 음수일 경우 대상(destination) 바이트의 모든 비트를 1로 설정하고, 양수일 경우 모든 비트를 0으로 설정합니다. 이후 punpckX 명령을 앞서 설명한 방식으로 함께 사용하면 부호 확장이 이루어집니다. 즉, 바이트가 음수이면 해당 바이트는 0b11111111, 그렇지 않으면 0x00000000이 되며, 이 값을 원래의 바이트와 교차(interleave)하면 워드 단위의 부호 확장이 수행됩니다.

```assembly
pxor      m2, m2 ; m2를 0으로 초기화

movu      m0, [srcq]
movu      m1, m0 ; m0를 m1에 복사

pcmpgtb   m2, m0
punpcklbw m0, m2
punpckhbw m1, m2
```

보시다시피, 부호 없는 경우에 비해 명령어 하나 더 추가되었습니다.

**패킹 (Packing)**

packuswb (pack unsigned word to byte)와 packsswb 명령은 워드(16비트) 데이터를 바이트(8비트)로 변환할 때 사용됩니다. 이 명령들은 두 개의 SIMD 레지스터에 들어 있는 워드 데이터를 하나의 SIMD 레지스터 안의 바이트들로 교차(interleave)시킵니다. 값이 바이트 범위를 초과할 경우, 해당값은 포화 연산(saturation)으로 처리되어 즉, 최대값에서 클램프(clamp)됩니다.

**셔플 (Shuffles)**

셔플(Shuffles), 또는 퍼뮤트(Permutes)라고도 불리는 명령어들은 비디오 처리에서 가장 중요한 명령어라고 할 수 있습니다. 그중에서도 SSSE3에서 제공되는 pshufb(packed shuffle bytes)는 가장 핵심적인 셔플 명령어입니다.

이 명령은 각 바이트에 대해, 해당 바이트의 값을 소스 바이트 인덱스로 사용하여 대상(destination) 레지스터의 바이트를 재배치합니다. 단, 바이트의 최상위 비트(MSB)가 설정되어있으면 해당 출력 바이트는 0으로 채워집니다. 이는 다음의 C 코드와 유사합니다. (단, SIMD에서는 모든 16번 반복이 동시에 병렬로 수행됩니다)

```c
for(int i = 0; i < 16; i++) {
    if(src[i] & 0x80)
        dst[i] = 0;
    else
        dst[i] = dst[src[i]]
}
```

다음은 간단한 어셈블리 예시입니다.

```assembly
SECTION_DATA 64

shuffle_mask: db 4, 3, 1, 2, -1, 2, 3, 7, 5, 4, 3, 8, 12, 13, 15, -1

section .text

movu m0, [srcq]
movu m1, [shuffle_mask]
pshufb m0, m1 ; shuffle m0 based on m1
```

여기서 -1은 읽기 편하게 하기 위해 셔플 인덱스로 사용된 값이며, 출력 바이트를 0으로 만드는 역할을 합니다. -1은 바이트 단위로 표현하면 0b11111111 (2의 보수)이므로, MSB(0x80)가 설정되어있어 해당 바이트가 0으로 처리됩니다.

[image1]: <data:image/png;base64,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>