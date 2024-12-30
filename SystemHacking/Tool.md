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
[중략]
```

**DISASM** 영역의 화살표(►)가 가리키는 주소는 현재 rip의 값인데, readelf으로 확인했던 것과 동일한 0x401050을 가리키고 있다. 참고로 화살표가 가리키고 있는 주소는 CPU가 다음에 실행할 코드의 주소를 표시하고 있는 것이다.

## context
