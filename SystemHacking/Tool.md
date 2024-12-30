# Tool
이 파일에는 **gdb, pwntools** 사용법에 대해서 간단하게 정리한다.

## Debuger
디버거는 버그를 없애기 위해 사용하는 도구로, 프로그램을 어셈블리 코드 단위로 실행하면서 실행 결과를 사용자에게 보여준다.

## gdb & pwndbg
gdb는 리눅스의 대표적인 디버거이다. gdb는 다양한 플러그인을 가지고 있는데, 그 중 pwndbg에 대해 정리한다.

## 실습 예제

```c
// Name: debugee.c
// Compile: gcc -o debugee debugee.c -no-pie

#include <stdio.h>
int main(void) {
  int sum = 0;
  int val1 = 1;
  int val2 = 2;

  sum = val1 + val2;

  printf("1 + 2 = %d\n", sum);

  return 0;
}
```

## entry
리눅스는 실행파일의 형식으로 **ELF**를 규정하고 있다. ELF는 크게 헤더와 여러 섹션들로 구성되어 있다. 헤더에는 실행에 필요한 여러 정보가 적혀 있고, 센션들에는 컴파일된 기계어 코드, 프로그램 문자열을 비롯한 여러 데이터가 포함되어 있다. ELF 헤더 중 **Entry Point(진입점)** 이라는 필드가 있는데, 운영체제는 ELF를 실행할 때, 진입점부터 프로그램을 실행한다.

```bash
EST3524@DESKTOP-411274Q:~$ readelf -h debugee
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x401050
  Start of program headers:          64 (bytes into file)
  Start of section headers:          13912 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         31
  Section header string table index: 30
```

**readelf**를 사용하면 파일의 구조와 같은 다양한 정보를 확인할 수 있다. **-h** 옵션은 헤더파일을 대상으로 지정한다. debugee의 헤더 중 **Entry point address : 0x401050** 으로 프로그램 진입점의 메모리 주소를 알 수 있다.  

gdb의 **entry** 명령어를 사용하면 진입점부터 프로그램을 분석할 수 있다. 

```asm
pwndbg> entry

Program stopped.
0x00007ffff7fe3290 in _start () from /lib64/ld-linux-x86-64.so.2
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
[중략]
─────────────────────────────────[ DISASM / x86-64 / set emulate on ]─────────────────────────────────
 ► 0x401050 <_start>       endbr64
   0x401054 <_start+4>     xor    ebp, ebp                    EBP => 0
   0x401056 <_start+6>     mov    r9, rdx                     R9 => 0x7ffff7fc9040 (_dl_fini) ◂— endbr64
   0x401059 <_start+9>     pop    rsi                         RSI => 1
   0x40105a <_start+10>    mov    rdx, rsp                    RDX => 0x7fffffffe188 —▸ 0x7fffffffe3f7 ◂— '/home/EST3524/debugee'
   0x40105d <_start+13>    and    rsp, 0xfffffffffffffff0     RSP => 0x7fffffffe180 (0x7fffffffe188 & -0x10)
   0x401061 <_start+17>    push   rax
   0x401062 <_start+18>    push   rsp
   0x401063 <_start+19>    xor    r8d, r8d                    R8D => 0
   0x401066 <_start+22>    xor    ecx, ecx                    ECX => 0
   0x401068 <_start+24>    mov    rdi, main                   RDI => 0x401136 (main) ◂— endbr64
[생략]
```

**DISASM** 영역의 화살표(►)가 가리키는 주소는 현재 rip의 값인데, readelf으로 확인했던 것과 동일한 0x401050을 가리키고 있다. 참고로 0x401050을 실행한 후가 아니라 실행하기 직전인 상태이다.

## context
프로그램은 실행되면서 레지스터를 비롯한 여러 메모리에 접근한다. pwndbg는 주요 메모리들의 상태를 프로그램이 실행되고 있는 context라고 부르며, 이를 가독성 있게 표현한다.  

context는 크게 4개의 영역으로 구분된다.  

1. REGISTERS : 레지스터의 상태를 보여준다.
2. DISASM : rip부터 여러 줄에 걸쳐 디스어셈블된 결과를 보여준다.
3. STACK L rsp부터 여러 줄에 걸쳐 스택의 값들을 보여준다.
4. BACKTRACE : 현재 rip에 도달할 때까지 어떤 함수들이 중첩되어 호출됐는지 보여준다.

```asm
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]────────────────────────
 RAX  0x1c
 RBX  0
 RCX  0x7fffffffe198 —▸ 0x7fffffffe40d ◂— 'SHELL=/bin/bash'
 RDX  0x7ffff7fc9040 (_dl_fini) ◂— endbr64
 RDI  0x7ffff7ffe2e0 ◂— 0
 RSI  0x7ffff7ffe888 ◂— 0
 R8   0x1fff
 R9   0xf
 R10  0x7ffff7fc3860 ◂— 0xd0012000000c1
 R11  0x202
 R12  0x401050 (_start) ◂— endbr64
 R13  0x7fffffffe180 ◂— 1
 R14  0
 R15  0
 RBP  0
 RSP  0x7fffffffe180 ◂— 1
 RIP  0x401050 (_start) ◂— endbr64
─────────────────────────────────[ DISASM / x86-64 / set emulate on ]─────────────────────────────────
 ► 0x401050 <_start>       endbr64
   0x401054 <_start+4>     xor    ebp, ebp                    EBP => 0
   0x401056 <_start+6>     mov    r9, rdx                     R9 => 0x7ffff7fc9040 (_dl_fini) ◂— endbr64
   0x401059 <_start+9>     pop    rsi                         RSI => 1
   0x40105a <_start+10>    mov    rdx, rsp                    RDX => 0x7fffffffe188 —▸ 0x7fffffffe3f7 ◂— '/home/EST3524/debugee'
   0x40105d <_start+13>    and    rsp, 0xfffffffffffffff0     RSP => 0x7fffffffe180 (0x7fffffffe188 & -0x10)
   0x401061 <_start+17>    push   rax
   0x401062 <_start+18>    push   rsp
   0x401063 <_start+19>    xor    r8d, r8d                    R8D => 0
   0x401066 <_start+22>    xor    ecx, ecx                    ECX => 0
   0x401068 <_start+24>    mov    rdi, main                   RDI => 0x401136 (main) ◂— endbr64
──────────────────────────────────────────────[ STACK ]───────────────────────────────────────────────
00:0000│ r13 rsp 0x7fffffffe180 ◂— 1
01:0008│         0x7fffffffe188 —▸ 0x7fffffffe3f7 ◂— '/home/EST3524/debugee'
02:0010│         0x7fffffffe190 ◂— 0
03:0018│ rcx     0x7fffffffe198 —▸ 0x7fffffffe40d ◂— 'SHELL=/bin/bash'
04:0020│         0x7fffffffe1a0 —▸ 0x7fffffffe41d ◂— 'WSL2_GUI_APPS_ENABLED=1'
05:0028│         0x7fffffffe1a8 —▸ 0x7fffffffe435 ◂— 'WSL_DISTRO_NAME=Ubuntu'
06:0030│         0x7fffffffe1b0 —▸ 0x7fffffffe44c ◂— 'NAME=DESKTOP-411274Q'
07:0038│         0x7fffffffe1b8 —▸ 0x7fffffffe461 ◂— 'PWD=/home/EST3524'
────────────────────────────────────────────[ BACKTRACE ]─────────────────────────────────────────────
 ► 0         0x401050 _start
──────────────────────────────────────────────────────────────────────────────────────────────────────
```

## break & continue
**break**를 사용하여 특정 주소에 **breakpoint**를 설정할 수 있고, **continue**를 사용하여 중단된 프로그램을 계속 실행시킬 수 있다.

```asm
pwndbg> b *main
Breakpoint 1 at 0x401136
pwndbg> c
Continuing.

Breakpoint 1, 0x0000000000401136 in main ()
[중략]
──────────────────────────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────────────────────────
 ► 0x401136 <main>       endbr64
   0x40113a <main+4>     push   rbp
   0x40113b <main+5>     mov    rbp, rsp                       RBP => 0x7fffffffe070 ◂— 1
   0x40113e <main+8>     sub    rsp, 0x10                      RSP => 0x7fffffffe060 (0x7fffffffe070 - 0x10)
   0x401142 <main+12>    mov    dword ptr [rbp - 0xc], 0       [0x7fffffffe064] <= 0
   0x401149 <main+19>    mov    dword ptr [rbp - 8], 1         [0x7fffffffe068] <= 1
   0x401150 <main+26>    mov    dword ptr [rbp - 4], 2         [0x7fffffffe06c] <= 2
   0x401157 <main+33>    mov    edx, dword ptr [rbp - 8]       EDX, [0x7fffffffe068] => 1
   0x40115a <main+36>    mov    eax, dword ptr [rbp - 4]       EAX, [0x7fffffffe06c] => 2
   0x40115d <main+39>    add    eax, edx                       EAX => 3 (2 + 1)
   0x40115f <main+41>    mov    dword ptr [rbp - 0xc], eax     [0x7fffffffe064] <= 3
[생략]
```

## run
**run**은 프로그램을 실행시키는 명령어이다.

```asm
pwndbg> r
Starting program: /home/EST3524/debugee
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, 0x0000000000401136 in main ()
```

gdb의 명령어 축약
- b : break
- c : continue
- r : run
- si : step into
- ni : next instruction
- i : info
- k : kill
- pd : pdisas

## disassemble
**disassemble**은 gdb가 기본적으로 제공하는 디스어셈블 명령어이다. 함수 이름을 인자로 전달하면 해당 함수가 반환될 때까지 전부 디스어셈블하여 보여준다.

```asm
pwndbg> disassemble main
Dump of assembler code for function main:
=> 0x0000000000401136 <+0>:     endbr64
   0x000000000040113a <+4>:     push   rbp
   0x000000000040113b <+5>:     mov    rbp,rsp
   0x000000000040113e <+8>:     sub    rsp,0x10
   0x0000000000401142 <+12>:    mov    DWORD PTR [rbp-0xc],0x0
   0x0000000000401149 <+19>:    mov    DWORD PTR [rbp-0x8],0x1
   0x0000000000401150 <+26>:    mov    DWORD PTR [rbp-0x4],0x2
   0x0000000000401157 <+33>:    mov    edx,DWORD PTR [rbp-0x8]
   0x000000000040115a <+36>:    mov    eax,DWORD PTR [rbp-0x4]
   0x000000000040115d <+39>:    add    eax,edx
   0x000000000040115f <+41>:    mov    DWORD PTR [rbp-0xc],eax
   0x0000000000401162 <+44>:    mov    eax,DWORD PTR [rbp-0xc]
   0x0000000000401165 <+47>:    mov    esi,eax
   0x0000000000401167 <+49>:    lea    rax,[rip+0xe96]        # 0x402004
   0x000000000040116e <+56>:    mov    rdi,rax
   0x0000000000401171 <+59>:    mov    eax,0x0
   0x0000000000401176 <+64>:    call   0x401040 <printf@plt>
   0x000000000040117b <+69>:    mov    eax,0x0
   0x0000000000401180 <+74>:    leave
   0x0000000000401181 <+75>:    ret
End of assembler dump.
```

**u, nearpc, pdisas**는 pwndbg에서 제공하는 디스어셈블 명령어이다. 디스어셈블된 코드를 가독성 좋게 출력해 준다.

```asm
pwndbg> u
 ► 0x401136 <main>       endbr64
   0x40113a <main+4>     push   rbp
   0x40113b <main+5>     mov    rbp, rsp
   0x40113e <main+8>     sub    rsp, 0x10
   0x401142 <main+12>    mov    dword ptr [rbp - 0xc], 0
   0x401149 <main+19>    mov    dword ptr [rbp - 8], 1
   0x401150 <main+26>    mov    dword ptr [rbp - 4], 2
   0x401157 <main+33>    mov    edx, dword ptr [rbp - 8]
   0x40115a <main+36>    mov    eax, dword ptr [rbp - 4]
   0x40115d <main+39>    add    eax, edx
   0x40115f <main+41>    mov    dword ptr [rbp - 0xc], eax
pwndbg> nearpc
 ► 0x401136 <main>       endbr64
   0x40113a <main+4>     push   rbp
   0x40113b <main+5>     mov    rbp, rsp
   0x40113e <main+8>     sub    rsp, 0x10
   0x401142 <main+12>    mov    dword ptr [rbp - 0xc], 0
   0x401149 <main+19>    mov    dword ptr [rbp - 8], 1
   0x401150 <main+26>    mov    dword ptr [rbp - 4], 2
   0x401157 <main+33>    mov    edx, dword ptr [rbp - 8]
   0x40115a <main+36>    mov    eax, dword ptr [rbp - 4]
   0x40115d <main+39>    add    eax, edx
   0x40115f <main+41>    mov    dword ptr [rbp - 0xc], eax
pwndbg> pdisas
 ► 0x401136 <main>       endbr64
   0x40113a <main+4>     push   rbp
   0x40113b <main+5>     mov    rbp, rsp
   0x40113e <main+8>     sub    rsp, 0x10
   0x401142 <main+12>    mov    dword ptr [rbp - 0xc], 0
   0x401149 <main+19>    mov    dword ptr [rbp - 8], 1
   0x401150 <main+26>    mov    dword ptr [rbp - 4], 2
   0x401157 <main+33>    mov    edx, dword ptr [rbp - 8]
   0x40115a <main+36>    mov    eax, dword ptr [rbp - 4]
   0x40115d <main+39>    add    eax, edx
   0x40115f <main+41>    mov    dword ptr [rbp - 0xc], eax
```

## navigate
관찰하고자 하는 함수의 중단점에 도달했다면, 그 지점부터는 명령어를 한 줄씩 자세히 분석해야 한다. 이 때 **ni, si**을 사용한다.  

ni와 si는 모두 어셈블리 명령어를 한 줄 실행하지만, call 등을 통해 서브루틴을 호출하는 경우 ni는 서브루틴의 내부로 들어가지 않지만, si는 서브루틴의 내부로 들어간다는 차이점이 있다.

### next instruction
ni를 입력하면 다음과 같이 printf함수 바로 다음으로 rip가 이동한 것을 확인할 수 있다.

```asm

