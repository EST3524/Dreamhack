# Static Link vs. Dynamic Link
이 파일에는 Static Link, Dynamic Link에 대해서 정리한다.

# Overview

- Library

- Link
	- Static Link
	- Dynamic Link

- PLT & GOT

## Library
라이브러리는 컴퓨터 시스템에서, 프로그램들이 함수나 변수를 공유해서 사용할 수 있게 한다. C의 표준 라이브러리인 **libc**는 ubuntu에 기본적으로 탑재된 라이브러리이며, **/lib/x86_64-linux-gnu/libc.so.6**에 있다.  

라이브러리는 크게 **Static Library**, **Dynamic Library**로 구분된다.

## Link
링크는 컴파일의 마지막 단계로, 프로그램에서 어떤 라이브러리의 함수를 사용한다면, 호출된 함수와 실제 라이브러리의 함수가 링크 과정에서 연결된다.

```c
// Name: hello-world.c
// Compile: gcc -o hello-world hello-world.c

#include <stdio.h>

int main() {
  puts("Hello, world!");
  return 0;
}
```

```c
// Path: /usr/include/stdio.h

...
/* Write a string, followed by a newline, to stdout.

   This function is a possible cancellation point and therefore not
   marked with __THROW.  */
extern int puts (const char *__s);
...
```

리눅스에서 C 소스 코드는 전처리, 컴파일, 어셈블 과정을 거쳐 ELF형식을 갖춘 **Object file**로 번역된다. 다음 명령어로 hello-world.c를 어셈블할 수 있다.

```bash
$ gcc -c hello-world.c -o hello-world.o
```

오브젝트 파일은 실행 가능한 형식을 갖추고 있지만, 라이브러리 함수들의 정의가 어디 있는지 알지 못하므로 실행은 불가능하다. 다음 명령어를 실행해보면, puts의 선언이 stdio.h에 있어서 **Symbol**로는 기록되어 있지만, 심볼에 대한 자세한 내용은 하나도 기록되어 있지 않다. 심볼과 관련된 정보들을 찾아서 최종 실행 파일에 기록하는 것이 링크 과정에서 하는 일 중 하나이다.

```bash
$ readelf -s hello-world.o | grep puts
    11: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND puts
```

다음은 예제를 완전히 컴파일한 것인데, libc에서 puts의 정의를 찾아 연결한 것을 확인할 수 있다.

```bash
$ gcc -o hello-world hello-world.c
$ readelf -s hello-world | grep puts
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5 (2)
    46: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@@GLIBC_2.2.5
$ ldd hello-world
        linux-vdso.so.1 (0x00007ffec3995000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fee37831000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fee37e24000)
```

여기서 libc를 같이 컴파일하지 않았음에도 libc에서 해당 심볼을 탐색한 것은, libc가 있는 /lib/x86_64-linux-gnu/가 표준 라이브러리 경로에 포함되어 있기 때문이다. gcc는 소스 코드를 컴파일할 때 표준 라이브러리의 라이브러리 파일들을 모두 탐색한다. 다음 명령어들로 표준 라이브러리의 경로를 확인할 수 있다.

```bash
$ ld --verbose | grep SEARCH_DIR | tr -s ' ;' '\n'
SEARCH_DIR("=/usr/local/lib/x86_64-linux-gnu")
SEARCH_DIR("=/lib/x86_64-linux-gnu")
SEARCH_DIR("=/usr/lib/x86_64-linux-gnu")
SEARCH_DIR("=/usr/lib/x86_64-linux-gnu64")
SEARCH_DIR("=/usr/local/lib64")
SEARCH_DIR("=/lib64")
SEARCH_DIR("=/usr/lib64")
SEARCH_DIR("=/usr/local/lib")
SEARCH_DIR("=/lib")
SEARCH_DIR("=/usr/lib")
SEARCH_DIR("=/usr/x86_64-linux-gnu/lib64")
SEARCH_DIR("=/usr/x86_64-linux-gnu/lib")
```

링크를 거치고 나면 프로그램에서 puts를 호출할 때, puts의 정의가 있는 libc에서 puts의 코드를 찾고, 해당 코드를 실행하게 된다.

## Static Link
정적 링크는 정적 라이브러리를 링크하는 것으로, 링크 과정에서 바이너리에 정적 라이브러리의 필요한 모든 함수를 포함시킨다. 따라서 해당 함수를 호출할 때, 라이브러리를 참조하는 것이 아니라 자신의 함수를 호출하는 것처럼 호출할 수 있다.  

정적 링크 시 컴파일 옵션에 따라 include한 헤더의 함수가 모두 포함될 수도 있고, 그렇지 않을 수도 있다.

```bash
$ gcc -o static hello-world.c -static
```

```asm
 main:
  push   rbp
  mov    rbp,rsp
  lea    rax,[rip+0x96880] # 0x498004
  mov    rdi,rax
  call   0x40c140 <puts>
  mov    eax,0x0
  pop    rbp
  ret
```

정적 컴파일 시, puts가 있는 0x40c140을 직접 호출하는 것을 볼 수 있다.

## Dynamic Link
동적 링크는 동적 라이브러리를 링크하는 것으로, 동적 링크된 바이너리를 실행하면 동적 라이브러리가 프로세스의 메모리에 매핑된다. 실행 중에 라이브러리의 함수를 호출하면 매핑된 라이브러리에서 호출할 함수의 주소를 찾고, 그 함수를 실행한다.

```bash
$ gcc -o dynamic hello-world.c -no-pie
```

```asm
main: 
 push   rbp
 mov    rbp,rsp
 lea    rdi,[rip+0xebf] # 0x402004
 mov    rdi,rax
 call   0x401040 <puts@plt>
 mov    eax,0x0
 pop    rbp
 ret
```

동적 컴파일 시, puts를 직접 컴파일 하는 것이 아닌 puts의 plt주소인 puts@plt : 0x401040을 호출한다. plt는 동적 링크된 바이너리가 함수의 주소를 라이브러리에서 찾을 때 사용하는 테이블이다.

##	PLT & GOT
PLT와 GOT는 라이브러리에서 동적 링크된 심볼의 주소를 찾을 때 사용하는 테이블이다. 바이너리가 실행되면, ASLR에 의해 라이브러리가 임의의 주소에 매핑된다. 이 상태에서 system()과 같은 라이브러리 함수를 처음 호출하면, system@plt라는 system()의 PLT엔트리를 호출하는데, PLT는 GOT에 적혀있는 주소를 참조하여 그 주소로 jmp한다. 이 때, GOT에는 system()의 주소를 찾기 전이므로, 실제 system()의 주소가 아니라 동적 링커의 주소가 적혀있다. 동적 링커는 함수의 이름을 바탕으로 라이브러리에서 심볼들을 탐색하고, 해당 함수의 정의를 발견하면 그 주소로 실행 흐름을 옮기고 GOT에 실제 system()의 주소를 적는다. 이 과정을 runtime resolve라고 한다. 따라서 두 번째 호출부터는 system@plt가 실행되었을 때, PLT가 GOT에 적혀있는 system()의 주소로 바로 jmp할 수 있다.  

다음은 실제 바이너리의 동작이다.  

**예제 코드**

```c
// Name: got.c
// Compile: gcc -o got got.c -no-pie

#include <stdio.h>

int main() {
  puts("Resolving address of 'puts'.");
  puts("Get address from GOT");
}
```

got.c를 컬파일하고 실행한 직후에, GOT에는 puts의 GOT 엔트리인 0x404018에는 아직 puts의 주소를 찾기 전이므로, 함수 주소 대신 .plt 섹션 어딘가의 주소인 0x401030이 적혀있다.

```asm
$ gdb ./got
pwndbg> entry
pwndbg> got
GOT protection: Partial RELRO | GOT functions: 1
[0x404018] puts@GLIBC_2.2.5 -> 0x401030 ◂— endbr64

pwndbg> plt
Section .plt 0x401020-0x401040:
No symbols found in section .plt
pwndbg>
```

main()에서 puts@plt를 호출하는 지점에 중단점을 설정하고, si를 사용하여 함수의 내부로 들어가면, 먼저 PLT에서 puts의 GOT 엔트리에 쓰인 값인 0x401030으로 실행흐름을 옮긴다. 여기서 Control flow를 따라가면 _dl_runtime_resolve_fxsave가 호출된다는 것을 확인할 수 있다.

```asm
pwndbg> b *main+18
pwndbg> c
...
──────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────
   0x40113e <main+8>     lea    rax, [rip + 0xebf]
   0x401145 <main+15>    mov    rdi, rax
 ► 0x401148 <main+18>    call   puts@plt                      <puts@plt>
        s: 0x402004 ◂— "Resolving address of 'puts'."
...
pwndbg> si
...
──────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────
 ► 0x401040       <puts@plt>                        endbr64
   0x401044       <puts@plt+4>                      bnd jmp qword ptr [rip + 0x2fcd]     <0x401030>
    ↓
   0x401030                                         endbr64
   0x401034                                         push   0
   0x401039                                         bnd jmp 0x401020                     <0x401020>
    ↓
   0x401020                                         push   qword ptr [rip + 0x2fe2]      <_GLOBAL_OFFSET_TABLE_+8>
   0x401026                                         bnd jmp qword ptr [rip + 0x2fe3]     <_dl_runtime_resolve_fxsave>
    ↓
   0x7ffff7fd8be0 <_dl_runtime_resolve_fxsave>      endbr64
   0x7ffff7fd8be4 <_dl_runtime_resolve_fxsave+4>    push   rbx
   0x7ffff7fd8be5 <_dl_runtime_resolve_fxsave+5>    mov    rbx, rsp
   0x7ffff7fd8be8 <_dl_runtime_resolve_fxsave+8>    and    rsp, 0xfffffffffffffff0
...
```

_dl_runtime_resolve_fxsave에서는 puts의 주소가 구해지고, GOT 엔트리에 이 주소를 쓴다. ni 명령어를 반복적으로 수행해서 _dl_runtime_resolve_fxsave 안으로 진입한 후, finish 명령어로 함수를 빠져나오면, puts의 GOT 엔트리에 libc 영역 내 실제 puts 주소인 0x7ffff7e02ed0이 쓰여있는 것을 확인할 수 있다.

```asm
pwndbg> ni
...
pwndbg> ni
_dl_runtime_resolve_fxsave () at ../sysdeps/x86_64/dl-trampoline.h:67
67  ../sysdeps/x86_64/dl-trampoline.h: No such file or directory.
...
──────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────
   0x401030                                          endbr64
   0x401034                                          push   0
   0x401039                                          bnd jmp 0x401020                     <0x401020>
    ↓
   0x401020                                          push   qword ptr [rip + 0x2fe2]      <_GLOBAL_OFFSET_TABLE_+8>
   0x401026                                          bnd jmp qword ptr [rip + 0x2fe3]     <_dl_runtime_resolve_fxsave>
    ↓
 ► 0x7ffff7fd8be0 <_dl_runtime_resolve_fxsave>       endbr64
   0x7ffff7fd8be4 <_dl_runtime_resolve_fxsave+4>     push   rbx
   0x7ffff7fd8be5 <_dl_runtime_resolve_fxsave+5>     mov    rbx, rsp
   0x7ffff7fd8be8 <_dl_runtime_resolve_fxsave+8>     and    rsp, 0xfffffffffffffff0
   0x7ffff7fd8bec <_dl_runtime_resolve_fxsave+12>    sub    rsp, 0x240
   0x7ffff7fd8bf3 <_dl_runtime_resolve_fxsave+19>    mov    qword ptr [rsp], rax
...
pwndbg> finish
Run till exit from #0  _dl_runtime_resolve_fxsave () at ../sysdeps/x86_64/dl-trampoline.h:67
Resolving address of 'puts'.
...
──────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────
   0x401148 <main+18>    call   puts@plt                      <puts@plt>

 ► 0x40114d <main+23>    lea    rax, [rip + 0xecd]
   0x401154 <main+30>    mov    rdi, rax
   0x401157 <main+33>    call   puts@plt                      <puts@plt>

   0x40115c <main+38>    mov    eax, 0
   0x401161 <main+43>    pop    rbp
   0x401162 <main+44>    ret

   0x401163              add    bl, dh
...
pwndbg> got
GOT protection: Partial RELRO | GOT functions: 1
[0x404018] puts@GLIBC_2.2.5 -> 0x7ffff7e02ed0 (puts) ◂— endbr64
pwndbg> vmmap 0x7ffff7e02ed0
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
             Start                End Perm     Size Offset File
    0x7ffff7daa000     0x7ffff7f3f000 r-xp   195000  28000 /usr/lib/x86_64-linux-gnu/libc.so.6 +0x58ed0
```
puts@plt를 두 번째로 호출할 때는 puts의 GOT 엔트리에 실제 puts의 주소인 0x7ffff7e02ed0가 쓰여있어서 바로 puts가 실행된다.

```asm
pwndbg> b *main+33
pwndbg> c
...
──────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────
   0x401148 <main+18>    call   puts@plt                      <puts@plt>

   0x40114d <main+23>    lea    rax, [rip + 0xecd]
   0x401154 <main+30>    mov    rdi, rax
 ► 0x401157 <main+33>    call   puts@plt                      <puts@plt>
        s: 0x402021 ◂— 'Get address from GOT'
...
pwndbg> si
...
──────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────
 ► 0x401040       <puts@plt>      endbr64
   0x401044       <puts@plt+4>    bnd jmp qword ptr [rip + 0x2fcd]     <puts>
    ↓
   0x7ffff7e02ed0 <puts>          endbr64
   0x7ffff7e02ed4 <puts+4>        push   r14
   0x7ffff7e02ed6 <puts+6>        push   r13
   0x7ffff7e02ed8 <puts+8>        push   r12
   0x7ffff7e02eda <puts+10>       mov    r12, rdi
   0x7ffff7e02edd <puts+13>       push   rbp
   0x7ffff7e02ede <puts+14>       push   rbx
   0x7ffff7e02edf <puts+15>       sub    rsp, 0x10
   0x7ffff7e02ee3 <puts+19>       call   *ABS*+0xa8720@plt                <*ABS*+0xa8720@plt>
...
```

PLT와 GOT는 동적 링크된 바이너리에서 라이브러리 함수의 주소를 찾고 기록할 때 사용되는 중요한 테이블이다. 그런데 PLT에서 GOT를 참조하여 실행 흐름을 옮길 때, GOT의 값을 검증하지 않는다는 보안상의 취약점이 있다.  

만약 앞의 예시에서 puts의 GOT 엔트리에 저장된 값을 공격자가 임의로 변경할 수 있으면, puts가 호출될 때 공격자가 원하는 코드가 실행될 수 있다.  

GOT 엔트리에 저장된 값을 임의로 변조할 수 있는 수단이 있음을 가정하고 이 공격 기법이 가능한지 gdb를 이용하여 간단하게 실험을 해볼 수 있다. got 바이너리에서 main() 내 두 번째 puts()호출 직전에 puts의 GOT 엔트리를 "AAAAAAAA"로 변경한 후 실행시키면, 실제로 "AAAAAAAA"로 실행 흐름이 옮겨지는 것을 확인할 수 있다.

```asm
$ gdb -q ./got
pwndbg> b *main+33
pwndbg> r
...
──────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────
 ► 0x401157 <main+33>    call   puts@plt                      <puts@plt>
        s: 0x402021 ◂— 'Get address from GOT'

   0x40115c <main+38>    mov    eax, 0
pwndbg> got
GOT protection: Partial RELRO | GOT functions: 1
[0x404018] puts@GLIBC_2.2.5 -> 0x7ffff7e02ed0 (puts) ◂— endbr64

pwndbg> set *(unsigned long long *)0x404018 = 0x4141414141414141

pwndbg> got
GOT protection: Partial RELRO | GOT functions: 1
[0x404018] puts@GLIBC_2.2.5 -> 0x4141414141414141 ('AAAAAAAA')
pwndbg> c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x0000000000401044 in puts@plt ()
...
──────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────
 ► 0x401044 <puts@plt+4>    bnd jmp qword ptr [rip + 0x2fcd]     <0x4141414141414141>
```

이와 같이 GOT 엔트리에 임의의 값을 Overwrite하여 실행 흐름을 변조하는 공격 기법을 GOT Overwrite라고 한다. 일반적으로 임의 주소에 임의의 값을 Overwrite하는 수단을 가지고 있을 때 수행하는 공격 기법이다.

참고 : [Static Link vs. Dynamic Link](https://dreamhack.io/lecture/courses/66)
