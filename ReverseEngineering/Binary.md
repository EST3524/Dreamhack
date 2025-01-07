# Binary
이 파일에는 바이너리에 대한 내용을 정리한다.

# Overview
- Program & Compile
  - Program
  - Compiler
  	- Preprocess
  	- Compile
  	- Assemble
  	- Link

# Program & Compile

## Program
프로그램은 연산 장치가 수행해야 하는 동작을 정의한 일종의 문서이다. 프로그램은 바이너리라고도 불리는데, 이는 프로그램이 저장 장치에 이진(Binary) 형태로 저장되기 때문이다.

## Compiler
CPU가 수행해야 할 명령들을 프로그래밍 언어로 작성한 것을 **Source Code**라고 하는데, 이를 컴퓨터가 이해할 수 있는 기계어의 형식으로 번역하는 것을 **Compile**이라고 한다. 컴파일을 해주는 소프트웨어는 **Compiler**라고 불리는데, 대표적으로 **GCC, Clang, MSVC** 등이 있다.  

C언어로 작성된 코드는 일반적으로 **Preprocess, Compile, Assemble, Link**의 과정을 거쳐 바이너리로 번역된다.  

**예제 코드**

```c
// Name: add.c

#include "add.h"

#define HI 3

int add(int a, int b) { return a + b + HI; }  // return a+b
```

```c
// Name: add.h

int add(int a, int b);
```

**Compile**의 정확한 의미는 어떤 언어로 작성된 **Source Code**를 다른 언어의 **Object Code**로 번역하는 것이다. 이런 맥락에서 소스 코드를 어셈블리어로 또는, 소스 코드를 기계어로 번역하는 것 모두 컴파일이라고 볼 수 있다.

## Preprocess
**Preprocessing**은 컴파일러가 소스코드를 어셈블리어로 컴파일하기 전에, 필요한 형식으로 가공하는 과정이다. 언어마다 조금씩 다르지만, 컴파일 언어의 대부분은 다음의 전처리 과정을 거친다.  

1. 주석 제거
주석은 개발자가 자신과 개발자들의 코드 이해를 돕기위해 작성하는 메모로, 프로그램의 동작과 상관이 없으므로 전처리 단계에서 모두 제거된다.  

2. 매크로 치환
#define으로 정의한 매크로는 자주쓰이는 코드나 상숫값을 단어로 정의한 것으로, 전처리 과정에서 매크로의 이름은 값으로 치환된다.

3. 파일 병합
일반적인 프로그램은 여러 개의 소스와 헤더 파일로 이루어져 있다. 컴파일러는 이를 따로 컴파일해 합치기도 하지만, 어떤 경우는 전처리 단계에서 파일을 합치고 컴파일하기도 한다.

```bash
$ gcc -E add.c > add.i
```

gcc에서는 -E 옵션을 사용하여 소스 코드의 전처리 결과를 확인할 수 있다.

```c
# 1 "add.c"
# 1 "<built-in>"
# 1 "<command-line>"
# 31 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 32 "<command-line>" 2
# 1 "add.c"
# 1 "add.h" 1
int add(int a, int b);
# 2 "add.c" 2



int add(int a, int b) { return a + b + 3; }
```

## Compile
**Compile**은 C로 작성된 소스 코드를 어셈블리어로 번역하는 것이다. 이 과정에서 **Compiler**는 소스 코드의 문법을 검사하는데, 코드에 문법적으로 오류가 있다면 컴파일을 멈추고 에러를 출력한다.  

또한, 컴파일러는 코드를 번역할 때, 몇몇 조건을 만족하면 최적화 기술을 적용하여 효율적인 어셈블리 코드를 생성해준다. gcc에서는 **-O -O0 -O1 -O2 -O3 -Os -Ofast -Og** 등의 옵션을 사용하여 최적화를 적용할 수 있다.  

```c
// Name: opt.c
// Compile: gcc -o opt opt.c -O2

#include <stdio.h>

int main() {
  int x = 0;
  for (int i = 0; i < 100; i++) x += i; // x에 0부터 99까지의 값 더하기
  printf("%d", x);
}
```

```asm
0x0000000000000560 <+0>:     lea    rsi,[rip+0x1bd]        ; 0x724
0x0000000000000567 <+7>:     sub    rsp,0x8
0x000000000000056b <+11>:    mov    edx,0x1356  ; hex((0+99)*50) = '0x1356' = sum(0,1,...,99) 
0x0000000000000570 <+16>:    mov    edi,0x1
0x0000000000000575 <+21>:    xor    eax,eax
0x0000000000000577 <+23>:    call   0x540 <__printf_chk@plt>
0x000000000000057c <+28>:    xor    eax,eax
0x000000000000057e <+30>:    add    rsp,0x8
0x0000000000000582 <+34>:    ret
```

위의 opt.c를 최적화하여 컴파일하면, 컴파일러는 반복문을 어셈블리어로 옮기는 것이 아니라 반복문의 결과로 x가 가질 값을 직접 계산하여 이를 대입하는 코드를 생성한다. 이를 통해 사용자가 작성한 소스 코드와 연산 결과는 같으면서도 최적화를 적용하지 않았을 때보다 더 짧고 실행시간도 단축되는 어셈블리 코드가 만들어지게 된다.

```bash
$ gcc -S add.i -o add.S
$ cat add.S
```

```asm
        .file   "add.c"
        .intel_syntax noprefix
        .text
        .globl  add
        .type   add, @function
add:
.LFB0:
        .cfi_startproc
        push    rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        mov     rbp, rsp
        .cfi_def_cfa_register 6
        mov     DWORD PTR -4[rbp], edi
        mov     DWORD PTR -8[rbp], esi
        mov     edx, DWORD PTR -4[rbp]
        mov     eax, DWORD PTR -8[rbp]
        add     eax, edx
        add     eax, 3
        pop     rbp
        .cfi_def_cfa 7, 8
        ret
        .cfi_endproc
.LFE0:
        .size   add, .-add
        .ident  "GCC: (Ubuntu 7.5.0-3ubuntu1~18.04) 7.5.0"
        .section        .note.GNU-stack,"",@progbits
```

위와 같이 -S 옵션을 이용하면 소스 코드를 어셈블리 코드로 컴파일할 수 있다.

## Assemble
**Assemble**은 컴파일로 생성된 어셈블리어 코드를 ELF형식의 **Object file**로 변환하는 과정이다. 여기서 **ELF**는 리눅스의 실행파일 형식이다. 윈도우에서 어셈블한다면 목적 파일은 **PE**형식을 갖게 된다.  

목적 파일로 변환되고 나면 어셈블리 코드가 기계어로 번역되므로 더이상 사람이 해석하기 어려워진다.

```
$ gcc -c add.S -o add.o
$ file add.o
add.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
$ hexdump -C add.o
00000000  7f 45 4c 46 02 01 01 00  00 00 00 00 00 00 00 00  |.ELF............|
00000010  01 00 3e 00 01 00 00 00  00 00 00 00 00 00 00 00  |..>.............|
00000020  00 00 00 00 00 00 00 00  10 02 00 00 00 00 00 00  |................|
00000030  00 00 00 00 40 00 00 00  00 00 40 00 0b 00 0a 00  |....@.....@.....|
00000040  55 48 89 e5 89 7d fc 89  75 f8 8b 55 fc 8b 45 f8  |UH...}..u..U..E.|
00000050  01 d0 5d c3 00 47 43 43  3a 20 28 55 62 75 6e 74  |..]..GCC: (Ubunt|
00000060  75 20 37 2e 35 2e 30 2d  33 75 62 75 6e 74 75 31  |u 7.5.0-3ubuntu1|
00000070  7e 31 38 2e 30 34 29 20  37 2e 35 2e 30 00 00 00  |~18.04) 7.5.0...|
00000080  14 00 00 00 00 00 00 00  01 7a 52 00 01 78 10 01  |.........zR..x..|
00000090  1b 0c 07 08 90 01 00 00  1c 00 00 00 1c 00 00 00  |................|
000000a0  00 00 00 00 14 00 00 00  00 41 0e 10 86 02 43 0d  |.........A....C.|
000000b0  06 4f 0c 07 08 00 00 00  00 00 00 00 00 00 00 00  |.O..............|
...
```

위의 예시는 gcc의 -c 옵션을 통해 add.S를 목적 파일로 변환하고, 결과로 나온 파일을 16진수로 출력한 것이다.

## Link
**Link**는 여러 목적 파일들을 연결하여 실행 가능한 바이너리로 만드는 과정이다.

```c
// Name: hello-world.c
// Compile: gcc -o hello-world hello-world.c

#include <stdio.h>

int main() { printf("Hello, world!"); }
```

위의 코드에서 printf함수를 호출하지만, printf함수의 정의는 hello-world.c에 없으며, libc라는 공유 라이브러리에 존재한다. libc는 gcc의 기본 라이브러리 경로에 있는데, 링커는 바이너리가 printf를 호출하면 libc의 함수가 실행될 수 있도록 연결해준다.링크를 거치고 나면 실행할 수 있는 프로그램이 완성된다.  

다음은 add.o를 링크하는 명령어이다. 링크 과정에서 링커는 main함수를 찾는데 add의 소스 코드에는 main함수의 정의가 없으므로 에러가 발생할 수 있다. 이를 방지하기 위해 --unresolved-symbols를 컴파일 옵션에 추가했다.

```bash
$ gcc add.o -o add -Xlinker --unresolved-symbols=ignore-in-object-files
$ file add
add: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/l, ...
```

- **Preproces -> Compile -> Aseemble -> Link**					
- **전처리된 코드 -> 어셈블리 코드 -> 목적 파일 -> 실행 가능한 바이너리**

참고 : [Binary](https://dreamhack.io/lecture/courses/67)










