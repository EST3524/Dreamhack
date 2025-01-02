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
pwndbg> ni
1 + 2 = 3
0x000000000040117b in main ()
[중략]

─────────────────────────────────[ DISASM / x86-64 / set emulate on ]─────────────────────────────────
   0x401176       <main+64>                       call   printf@plt                  <printf@plt>

 ► 0x40117b       <main+69>                       mov    eax, 0     EAX => 0
   0x401180       <main+74>                       leave
   0x401181       <main+75>                       ret                                <__libc_start_call_main+128>
    ↓
   0x7ffff7db4d90 <__libc_start_call_main+128>    mov    edi, eax     EDI => 0
   0x7ffff7db4d92 <__libc_start_call_main+130>    call   exit                        <exit>

   0x7ffff7db4d97 <__libc_start_call_main+135>    call   __nptl_deallocate_tsd       <__nptl_deallocate_tsd>

   0x7ffff7db4d9c <__libc_start_call_main+140>    lock dec dword ptr [rip + 0x1f0505]
   0x7ffff7db4da3 <__libc_start_call_main+147>    sete   al
   0x7ffff7db4da6 <__libc_start_call_main+150>    test   al, al
   0x7ffff7db4da8 <__libc_start_call_main+152>    jne    __libc_start_call_main+168  <__libc_start_call_main+168>
[생략]
```

printf가 출력하고자 하는 문자열은 stdout의 버퍼에서 잠시 대기한 뒤 출력된다. 여기서 버퍼는 데이터가 목적지로 이동하기 전에 잠시 저장되는 장소이다. stdout 버퍼는 특정 조건이 만족됐을 때만 데이터를 목적지로 이동시키는데, 조건은 다음과 같다.

1. 프로그램이 종료될 때
2. 버퍼가 가득 찼을 때
3. fflush와 같은 함수로 버퍼를 비우도록 명시했을 때
4. 개행문자가 버퍼에 들어왔을 때

### step into
printf 함수를 호출하는 지점까지 다시 프로그램을 실행시킨 뒤, **si**를 입력하면 다음과 같이 printf 함수 내부로 rip가 이동한 것을 확인할 수 있다. 프로그램을 분석하다가 어떤 함수의 내부까지 궁금할 때는 si를, 그렇지 않을 때는 ni를 사용한다.

```asm
pwndbg> si
0x0000000000401040 in printf@plt ()
[중략]
──────────────────────────────────────────────────────────────────────────────────────[ DISASM / x86-64 / set emulate on ]───────────────────────────────────────────────────────────────────────────────────────
 ► 0x401040       <printf@plt>                      endbr64
   0x401044       <printf@plt+4>                    bnd jmp qword ptr [rip + 0x2fcd]   <0x401030>
    ↓
   0x401030                                         endbr64
   0x401034                                         push   0
   0x401039                                         bnd jmp 0x401020                   <0x401020>
    ↓
   0x401020                                         push   qword ptr [rip + 0x2fe2]
   0x401026                                         bnd jmp qword ptr [rip + 0x2fe3]   <_dl_runtime_resolve_xsavec>
    ↓
   0x7ffff7fd8d30 <_dl_runtime_resolve_xsavec>      endbr64
   0x7ffff7fd8d34 <_dl_runtime_resolve_xsavec+4>    push   rbx
   0x7ffff7fd8d35 <_dl_runtime_resolve_xsavec+5>    mov    rbx, rsp                    RBX => 0x7fffffffe040 ◂— 0
   0x7ffff7fd8d38 <_dl_runtime_resolve_xsavec+8>    and    rsp, 0xffffffffffffffc0     RSP => 0x7fffffffe040 (0x7fffffffe040 & -0x40)
[생략]
```

### finish
step into로 함수 내부에 들어가서 필요한 부분을 모두 분석했는데, 함수의 규모가 커서 ni로는 원래 실행 흐름으로 돌아가기 어려울 때 **finish**라는 명령어를 사용하여 함수의 끝까지 한 번에 실행할 수 있다.

```asm
pwndbg> finish
Run till exit from #0  0x0000000000401040 in printf@plt ()
1 + 2 = 3
0x000000000040117b in main ()
[중략]
──────────────────────────────────────────────────────────────────────────────────────[ DISASM / x86-64 / set emulate on ]───────────────────────────────────────────────────────────────────────────────────────
   0x401176       <main+64>                       call   printf@plt                  <printf@plt>

 ► 0x40117b       <main+69>                       mov    eax, 0     EAX => 0
   0x401180       <main+74>                       leave
   0x401181       <main+75>                       ret                                <__libc_start_call_main+128>
    ↓
   0x7ffff7db4d90 <__libc_start_call_main+128>    mov    edi, eax     EDI => 0
   0x7ffff7db4d92 <__libc_start_call_main+130>    call   exit                        <exit>

   0x7ffff7db4d97 <__libc_start_call_main+135>    call   __nptl_deallocate_tsd       <__nptl_deallocate_tsd>

   0x7ffff7db4d9c <__libc_start_call_main+140>    lock dec dword ptr [rip + 0x1f0505]
   0x7ffff7db4da3 <__libc_start_call_main+147>    sete   al
   0x7ffff7db4da6 <__libc_start_call_main+150>    test   al, al
   0x7ffff7db4da8 <__libc_start_call_main+152>    jne    __libc_start_call_main+168  <__libc_start_call_main+168>
[생략]
```

## examine
프로그램을 분석하다 보면 가상 메모리에 존재하는 임의 주소의 값을 관찰해야할 때가 있다. 이를 위해 gdb에서는 기본적으로 **x**라는 명령어를 제공한다. x를 이용하면 **특정 주소**에서 **원하는 길이 만큼**의 데이터를 **원하는 형식으로 인코딩**하여 볼 수 있다.

- Format letters 
  - o(octal)
  - x(hex)
  - d(decimal)
  - u(unsigned decimal)
  - t(binary)
  - f(float)
  - a(address)
  - i(instruction)
  - c(char)
  - s(string)
  - z(hex, zero padded on the left)
  
- Size letters 
  - b(byte)
  - h(halfword)
  - w(word)
  - g(giant, 8 bytes)

**rsp부터 8바이트씩 hex형식으로 출력하는 것을 10번 반복**

```asm
pwndbg> x/10gx $rsp
0x7fffffffe078: 0x00007ffff7db4d90      0x0000000000000000
0x7fffffffe088: 0x0000000000401136      0x00000001ffffe170
0x7fffffffe098: 0x00007fffffffe188      0x0000000000000000
0x7fffffffe0a8: 0xb0117a1251dc0e82      0x00007fffffffe188
0x7fffffffe0b8: 0x0000000000401136      0x0000000000403e18
```

**rip부터 어셈블리 명령어를 출력하는 것을 5번 반복**

```asm
pwndbg> x/5i $rip
=> 0x401136 <main>:     endbr64
   0x40113a <main+4>:   push   rbp
   0x40113b <main+5>:   mov    rbp,rsp
   0x40113e <main+8>:   sub    rsp,0x10
   0x401142 <main+12>:  mov    DWORD PTR [rbp-0xc],0x0
```

**0x400000부터 문자열을 1번 출력**

```asm
pwndbg> x/s 0x400000
0x400000:       "\177ELF\002\001\001"
```

## telescope
**telescope**는 pwndbg가 제공하는 강력한 메모리 덤프 기능이다. 특정 주소의 메모리 값들을 보여주는 것에서 그치지 않고, 메모리가 참조하고 있는 주소를 재귀적으로 탐색하여 값을 보여준다.

```asm
pwndbg> tele
00:0000│ rsp 0x7fffffffe078 —▸ 0x7ffff7db4d90 (__libc_start_call_main+128) ◂— mov edi, eax
01:0008│     0x7fffffffe080 ◂— 0
02:0010│     0x7fffffffe088 —▸ 0x401136 (main) ◂— endbr64
03:0018│     0x7fffffffe090 ◂— 0x1ffffe170
04:0020│     0x7fffffffe098 —▸ 0x7fffffffe188 —▸ 0x7fffffffe3f7 ◂— '/home/EST3524/debugee'
05:0028│     0x7fffffffe0a0 ◂— 0
06:0030│     0x7fffffffe0a8 ◂— 0xd2dca7b59173129a
07:0038│     0x7fffffffe0b0 —▸ 0x7fffffffe188 —▸ 0x7fffffffe3f7 ◂— '/home/EST3524/debugee'
```

## vmmap
**vmmap은 가상 메모리의 레이아웃을 보여준다. 어떤 파일이 매핑 된 영역일 경우, 해당 파일의 경로까지 보여준다.

```asm
pwndbg> vmmap
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
             Start                End Perm     Size Offset File
          0x400000           0x401000 r--p     1000      0 /home/EST3524/debugee
          0x401000           0x402000 r-xp     1000   1000 /home/EST3524/debugee
          0x402000           0x403000 r--p     1000   2000 /home/EST3524/debugee
          0x403000           0x404000 r--p     1000   2000 /home/EST3524/debugee
          0x404000           0x405000 rw-p     1000   3000 /home/EST3524/debugee
    0x7ffff7d88000     0x7ffff7d8b000 rw-p     3000      0 [anon_7ffff7d88]
    0x7ffff7d8b000     0x7ffff7db3000 r--p    28000      0 /usr/lib/x86_64-linux-gnu/libc.so.6
    0x7ffff7db3000     0x7ffff7f48000 r-xp   195000  28000 /usr/lib/x86_64-linux-gnu/libc.so.6
    0x7ffff7f48000     0x7ffff7fa0000 r--p    58000 1bd000 /usr/lib/x86_64-linux-gnu/libc.so.6
    0x7ffff7fa0000     0x7ffff7fa1000 ---p     1000 215000 /usr/lib/x86_64-linux-gnu/libc.so.6
    0x7ffff7fa1000     0x7ffff7fa5000 r--p     4000 215000 /usr/lib/x86_64-linux-gnu/libc.so.6
    0x7ffff7fa5000     0x7ffff7fa7000 rw-p     2000 219000 /usr/lib/x86_64-linux-gnu/libc.so.6
    0x7ffff7fa7000     0x7ffff7fb4000 rw-p     d000      0 [anon_7ffff7fa7]
    0x7ffff7fbb000     0x7ffff7fbd000 rw-p     2000      0 [anon_7ffff7fbb]
    0x7ffff7fbd000     0x7ffff7fc1000 r--p     4000      0 [vvar]
    0x7ffff7fc1000     0x7ffff7fc3000 r-xp     2000      0 [vdso]
    0x7ffff7fc3000     0x7ffff7fc5000 r--p     2000      0 /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffff7fc5000     0x7ffff7fef000 r-xp    2a000   2000 /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffff7fef000     0x7ffff7ffa000 r--p     b000  2c000 /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffff7ffb000     0x7ffff7ffd000 r--p     2000  37000 /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffff7ffd000     0x7ffff7fff000 rw-p     2000  39000 /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
    0x7ffffffde000     0x7ffffffff000 rw-p    21000      0 [stack]
```

어떤 파일을 메모리에 적재하는 것을 파일 매핑이라고 한다. 위 메모리 레이아웃에서 **/home/EST3524/debugee, /usr/lib/x86_64-linux-gnu/libc.so.6, /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2**가 매핑된 파일들이다.  

리눅스에서는 ELF를 실행할 때, 먼저 ELF의 코드와 여러 데이터를 가상 메모리에 매핑하고, 해당 ELF에 링크된 **Shared Object, so(공유 오브젝트)** 를 추가로 메모리에 매핑한다. 공유 오브젝트는 자주 사용되는 함수들을 미리 컴파일해둔 것이다. C언어의 printf, scanf 등이 리눅스에서는 libc에 구현되어 있다. 공유 오브젝트에 이미 구현된 함수를 호출할 때는 매핑된 메모리에 존재하는 함수를 대신 호출한다.

## gdb / python
gdb를 사용하여 프로그램을 디버깅할 때, 키보드로 직접 타이핑하기 어렵거나 불가능한 값을 python을 사용하여 넘겨줄 수 있다. 

## 실습 예제

```c
// Name: debugee2.c
// Compile: gcc -o debugee2 debugee2.c -no-pie
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(int argc, char *argv[])
{
	char name[20];
	if( argc < 2 ) {
		printf("Give me the argv[1]!\n");
		exit(0);
	}
	memset(name, 0, sizeof(name));

	printf("argv[1] %s\n", argv[1]);

	read(0, name, sizeof(name)-1);
	printf("Name: %s\n", name);
	return 0;
}
```

int main(int argc, char *argv[])에서 argv는 문자열을 여러개 입력받을 수 있고, 입력 받은 문자열 포인터를 저장하는 배열이다. argv[0]은 프로그램의 실행 파일 이름 또는 경로로 자동 설정되며, 입력한 main()의 인자는 argv[1]부터 전달된다. argc는 전달된 인자의 개수인데, argv[0]을 포함하여 자동으로 계산하고 저장된다. memset()함수는 메모리 블록을 원하는 값으로 덮어 씌운다.

## gdb / python argv
run 명령어의 인자로 **$()** 와 함께 파이썬 코드를 입력하면 값을 전달할 수 있다. 다음은 파이썬에서 print 함수를 통해 출력한 값을 run 명령어의 인자로 전달하는 명령이다.

```bash
pwndbg> r $(python3 -c "print('\xff' * 100)")
Starting program: /home/EST3524/debugee2 $(python3 -c "print('\xff' * 100)")
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
argv[1] ÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿ
```

-c 옵션을 사용하면 "print('\xff' * 100)"를 파일 명이 아닌 문자열로 작성한 명령어로 해석한다.

## gdb / python input
이전과 같이 $()와 함께 파이썬 코드를 입력하면 값을 입력하여 함수의 인자로 전달할 수는 있지만, scanf()처럼 표준 입력이 필요한 함수에 입력 값으로써 전달할 수는 없다. 이 때 **<<<**를 사용하면 표준 입력 값으로써 출력을 전달할 수 있다.

```bash
pwndbg> r $(python3 -c "print('\xff' * 100)") <<< $(python3 -c "print('dreamhack')")
Starting program: /home/EST3524/debugee2 $(python3 -c "print('\xff' * 100)") <<< $(python3 -c "print('dreamhack')")
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
argv[1] ÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿÿ
Name: dreamhack

[Inferior 1 (process 4830) exited normally]
```

## pwntools
pwntools는 간단하게 익스플로잇을 제작할 수 있도록 도와주는 python 모듈이다.

## process & remote
**process** 함수는 익스플로잇을 로컬 바이너리를 대상으로 할 때 사용하는 함수이고, **remote** 함수는 원격 서버를 대상으로 할 때 사용하는 함수이다.

```pytnon
from pwn import *
p = process('./test')  # 로컬 바이너리 'test'를 대상으로 익스플로잇 수행
p = remote('example.com', 31337)  # 'example.com'의 31337 포트에서 실행 중인 프로세스를 대상으로 익스플로잇 수행
```

## send
**send**는 데이터를 프로세스에 전송하기 위해 사용한다.

```python
from pwn import *
p = process('./test')

p.send(b'A')  # ./test에 b'A'를 입력
p.sendline(b'A') # ./test에 b'A' + b'\n'을 입력
p.sendafter(b'hello', b'A')  # ./test가 b'hello'를 출력하면, b'A'를 입력
p.sendlineafter(b'hello', b'A')  # ./test가 b'hello'를 출력하면, b'A' + b'\n'을 입력
```




참고 : [gdb](https://dreamhack.io/lecture/courses/55)

