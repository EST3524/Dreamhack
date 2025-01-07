# Calling Convention
이 파일에는 함수 호출 규약에 대해 정리한다.

# Overview
- Calling Convention
  - SYSV
    - Argument Passing
    - Return Address Saving
    - Stack Frame Saving
    - Stack Frame Allocation
    - Return Value Passing
    - Return

  - cdecl

## Calling Convention
Calling Convention은 함수의 호출 및 반환에 대한 약속이다. 함수를 호출할 때는 Caller의 **Stack Frame** 및 **Return Address**를 저장해야 한다. 또한, Caller는 Callee가 요구하는 인자를 전달해줘야 하며, Callee의 실행이 종료될 때는 반환 값을 전달받아야 한다.  

프로그래머가 고수준 언어로 코드를 작성했을 때, 함수 호출 규약을 따로 명시하지 않는다면 컴파일러가 지원되는 여러 가지 호출 규약 중 CPU의 아키텍처에 적합한 것을 선택한다. 하지만, 컴파일러의 도움 없이 어셈블리 코드를 작성하려 하거나 또는 어셈블리로 작성된 코드를 읽고자 한다면 함수 호출 규약을 알아야 한다.  

CPU의 아키텍처가 같아도 컴파일러가 다르면 적용하는 호출 규약이 다를 수 있다.

- x86
  - cdecl +
  - stdcall
  - fastcall
  - thiscall

- x86-64
  - SYSTEM V +
  - MS x64

## SYSV
리눅스는 SYSTEM V(SYSV) ABI를 기반으로 만들어졌다. SYSV ABI는 ELF 포맷, 링킹 방법, 함수 호출 규약 등의 내용을 담고 있다. file 명령어를 이용하여 바이너리의 정보를 살펴보면, 아래와 같이 SYSV 문자열이 포함된 것을 확인할 수 있다.  

SYSV에서 정의한 함수 호출 규약은 다음의 특징을 갖는다.  

1. 6개의 인자를 **RDI, RSI, RDX, RCX, R8, R9**에 순서대로 저장하여 전달한다. 더 많은 인자를 사용해야 할 때는 추가로 스택을 이용한다.

2. Caller에서 인자 전달에 사용된 스택을 정리한다.

3. 함수의 반환 값은 **RAX**로 전달한다.  

**예제 코드**

```c
// Name: sysv.c
// Compile: gcc -fno-asynchronous-unwind-tables  -masm=intel \
//         -fno-omit-frame-pointer -S sysv.c -fno-pic -O0

#define ull unsigned long long

ull callee(ull a1, int a2, int a3, int a4, int a5, int a6, int a7) {
  ull ret = a1 + a2 + a3 + a4 + a5 + a6 + a7;
  return ret;
}

void caller() { callee(123456789123456789, 2, 3, 4, 5, 6, 7); }

int main() { caller(); }
```

## 1. Argument Passing
gdb로 sysv를 로드한 후 중단점을 설정하여 caller 함수까지 실행한다. context의 DISASM을 보면, 인자를 뒤에서부터 설정하는데, caller+10부터 caller+37까지 6개의 인자를 각각의 레지스터에 설정하고 있으며, caller+8에서는 7번째 인자인 7을 스택으로 전달하고 있다.

```asm
$ gdb -q sysv
pwndbg: loaded 139 pwndbg commands and 49 shell commands. Type pwndbg [--shell | --all] [filter] for a list.
pwndbg: created $rebase, $ida GDB functions (can be used with print/break)
Reading symbols from sysv...
...
pwndbg> b *caller
Breakpoint 1 at 0x1185
pwndbg> r
Starting program: /home/dreamhack/sysv

Breakpoint 1, 0x0000555555555185 in caller ()
...
──────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────
 ► 0x555555555185 <caller>       endbr64
   0x555555555189 <caller+4>     push   rbp
   0x55555555518a <caller+5>     mov    rbp, rsp
   0x55555555518d <caller+8>     push   7
   0x55555555518f <caller+10>    mov    r9d, 6
   0x555555555195 <caller+16>    mov    r8d, 5
   0x55555555519b <caller+22>    mov    ecx, 4
   0x5555555551a0 <caller+27>    mov    edx, 3
   0x5555555551a5 <caller+32>    mov    esi, 2
   0x5555555551aa <caller+37>    movabs rax, 0x1b69b4bacd05f15
   0x5555555551b4 <caller+47>    mov    rdi, rax
   0x5555555551b7 <caller+50>    call   0x555555555129 <callee>
   0x5555555551bc <caller+55>    add    rsp,0x8
...
```

disass 명령어로 caller()의 디스어셈블된 코드를 보고 caller()를 호출하는 부분을 파악한 후 해당 부분에 중단점을 설정한다.

```asm
pwndbg> disass caller
...
   0x00005555555551b7 <+50>:  call   0x555555555129 <callee>
...
pwndbg> b *caller+50
Breakpoint 2 at 0x5555555551b7
```

c 명령어를 사용해서 프로그램을 실행하면 callee()를 호출하기 직전에 멈춘다.

```asm 
pwndbg> c
Continuing.

Breakpoint 2, 0x00005555555551b7 in caller ()
...
─────────────[ REGISTERS / show-flags off / show-compact-regs off ]─────────────
*RAX  0x1b69b4bacd05f15
 RBX  0x0
*RCX  0x4
*RDX  0x3
*RDI  0x1b69b4bacd05f15
*RSI  0x2
*R8   0x5
*R9   0x6
 R10  0x7ffff7fc3908 ◂— 0xd00120000000e
 R11  0x7ffff7fde680 (_dl_audit_preinit) ◂— endbr64
...

pwndbg> x/4gx $rsp
0x7fffffffe2f8: 0x0000000000000007  0x00007fffffffe310
0x7fffffffe308: 0x00005555555551d5  0x0000000000000001
```

레지스터와 스택을 확인해보면, 소스 코드에서 **callee(123456789123456789, 2, 3, 4, 5, 6, 7)** 로 함수를 호출했는데, 인자들이 순서대로 **rdi, rsi, rdx, rcx, r8, r9 그리고 [rsp]** 에 설정되어 있는 것을 확인할 수 있다.

## 2. Return Address Saving
si 명령어로 한 단계 더 실행시킨다. call이 실행되고 스택을 확인해보면 0x555555554682가 반환주소로 저장되어 있다. gdb로 확인해보면 0x555555554682는 callee호출 다음 명령어의 주소이다. callee에서 반환됐을 때, 이 주소를 꺼내어 원래의 실행 흐름으로 돌아갈 수 있다.

```asm
pwndbg> si
0x00005555555545fa in callee ()
...
pwndbg> x/4gx $rsp
0x7fffffffdf70:	0x0000555555554682	0x0000000000000007
0x7fffffffdf80:	0x00007fffffffdf90	0x0000555555554697
pwndbg> x/10i 0x0000555555554682 - 5
   0x55555555467d <caller+43>:	call   0x5555555545fa <callee>
   0x555555554682 <caller+48>:	add    rsp,0x8
```

## 3. Stack Frame Saving
x/5i $rip 명령어로 callee함수의 Prologue를 살펴보면, 가장 먼저 push rbp를 통해 caller의 rbp를 저장하고 있다. rbp가 스택프레임의 가장 낮은 주소를 가리키는 포인터이므로, 이를 **SFP(Stack Frame Pointer)** 라고도 부른다. callee에서 반환될 때, SFP를 꺼내어 caller의 스택 프레임으로 돌아갈 수 있다.  

si로 push rbp를 실행하고, 스택을 확인해보면 rbp값인 0x00007fffffffe300가 저장된 것을 확인할 수 있다.

```asm
pwndbg> x/9i $rip
=> 0x555555555129 <callee>:	endbr64
   0x55555555512d <callee+4>:	push   rbp
   0x55555555512e <callee+5>:	mov    rbp,rsp
   0x555555555131 <callee+8>:	mov    QWORD PTR [rbp-0x18],rdi
   0x555555555135 <callee+12>:	mov    DWORD PTR [rbp-0x1c],esi
   0x555555555138 <callee+15>:	mov    DWORD PTR [rbp-0x20],edx
   0x55555555513b <callee+18>:	mov    DWORD PTR [rbp-0x24],ecx
   0x55555555513e <callee+21>:	mov    DWORD PTR [rbp-0x28],r8d
   0x555555555142 <callee+25>:	mov    DWORD PTR [rbp-0x2c],r9d
pwndbg> si
pwndbg> si
0x000055555555512e in callee ()
──────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────
   0x555555555129 <callee>       endbr64
   0x55555555512d <callee+4>     push   rbp
 ► 0x55555555512e <callee+5>     mov    rbp, rsp
   0x555555555131 <callee+8>     mov    qword ptr [rbp - 0x18], rdi
...
pwndbg> x/4gx $rsp
0x7fffffffe2e8: 0x00007fffffffe300  0x00005555555551bc
0x7fffffffe2f8: 0x0000000000000007  0x00007fffffffe310
pwndbg> print $rbp
$1 = (void *) 0x7fffffffe300
```

## 4. Stack Frame Allocation
mov rbp, rsp로 rbp와 rsp가 같은 주소를 가리키게 한 뒤, rsp의 값을 빼게 되면, rbp와 rsp 사이 공간을 새로운 스택 프레임으로 할당하는 것이지만, callee 함수는 지역 변수를 사용하지 않으므로, 새로운 스택 프레임을 만들지 않는다.  

si로 실행하고, 레지스터를 보면 이 둘이 같은 주소를 가리키는 것을 확인할 수 있다.

```asm
pwndbg> x/5i $rip
=> 0x55555555512e <callee+5>: mov    rbp,rsp
   0x555555555131 <callee+8>: mov    QWORD PTR [rbp-0x18],rdi
   0x555555555135 <callee+12>:  mov    DWORD PTR [rbp-0x1c],esi
   0x555555555138 <callee+15>:  mov    DWORD PTR [rbp-0x20],edx
   0x55555555513b <callee+18>:  mov    DWORD PTR [rbp-0x24],ecx

pwndbg> print $rbp
$2 = (void *) 0x7fffffffe300
pwndbg> print $rsp
$3 = (void *) 0x7fffffffe2e8

pwndbg> si

pwndbg> print $rbp
$4 = (void *) 0x7fffffffe2e8
pwndbg> print $rsp
$5 = (void *) 0x7fffffffe2e8
```

코드를 보면, ret라는 변수를 선언하긴 했지만, 반환 값을 저장하는 용도 외로는 사용되지 않는다. gcc는 이런 변수에 대해 스택을 할당하지 않고, **rax**를 직접 사용한다.

## 5. Return Value Passing
덧셈 연산을 모두 마치고, 함수의 Epilogue에 도달하면, 반환값을 **rax**에 옮긴다. 반환 직전에 **rax**를 출력하면 전달한 7개 인자의 합인 123456789123456816을 확인할 수 있다.

```asm
pwndbg> b *callee+79
Breakpoint 3 at 0x555555555178
pwndbg> c
...
──────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────
 ► 0x555555555178 <callee+79>    add    rax, rdx
   0x55555555517b <callee+82>    mov    qword ptr [rbp - 8], rax
   0x55555555517f <callee+86>    mov    rax, qword ptr [rbp - 8]
   0x555555555183 <callee+90>    pop    rbp
   0x555555555184 <callee+91>    ret

pwndbg> b *callee+91
Breakpoint 4 at 0x555555555184
pwndbg> c
...
──────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────
   0x555555555178 <callee+79>    add    rax, rdx
   0x55555555517b <callee+82>    mov    qword ptr [rbp - 8], rax
   0x55555555517f <callee+86>    mov    rax, qword ptr [rbp - 8]
   0x555555555183 <callee+90>    pop    rbp
 ► 0x555555555184 <callee+91>    ret                                  <0x5555555551bc; caller+55>
    ↓
...

pwndbg> print $rax
$1 = 123456789123456816
```

## 6. Return
반환은 저장해뒀던 스택 프레임과 반환 주소를 꺼내면서 이루어진다. 여기서는 callee 함수가 스택 프레임을 만들지 않았기 때문에, pop rbp로 스택 프레임을 꺼낼 수 있지만, 일반적으로 leave로 스택 프레임을 꺼낸다.  

스택 프레임을 꺼낸 뒤에는, ret로 caller에 복귀한다. 앞에서 저장해뒀던 sfp로 rbp가, 반환 주소로 rip가 설정된 것을 확인할 수 있다.

```asm
pwndbg> d
pwndbg> b *callee+90
Breakpoint 1 at 0x1183
pwndbg> r
...
──────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────
 ► 0x555555555183 <callee+90>                     pop    rbp
   0x555555555184 <callee+91>                     ret
    ↓
...

pwndbg> si
pwndbg> si
...
──────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────
   0x555555555183 <callee+90>                     pop    rbp
   0x555555555184 <callee+91>                     ret
    ↓
 ► 0x5555555551bc <caller+55>                     add    rsp, 8
   0x5555555551c0 <caller+59>                     nop
   0x5555555551c1 <caller+60>                     leave
   0x5555555551c2 <caller+61>                     ret
    ↓
...
pwndbg> print $rbp
$1 = (void *) 0x7fffffffe300
pwndbg> print $rip
$2 = (void (*)()) 0x5555555551bc <caller+55>
```

## cdecl
x86 아키텍처는 레지스터의 수가 적으므로, 스택을 통해 인자를 전달한다. 또한, 인자를 전달하기 위해 사용한 스택을 caller가 정리한다. 스택을 통해 인자를 전달할 때는, 마지막 인자부터 첫 번째 인자까지 거꾸로 스택에 push한다.  

**예제 코드**

```c
// Name: cdecl.c
// Compile: gcc -fno-asynchronous-unwind-tables -nostdlib -masm=intel \
//          -fomit-frame-pointer -S cdecl.c -w -m32 -fno-pic -O0

void __attribute__((cdecl)) callee(int a1, int a2){ // cdecl로 호출
}

void caller(){
   callee(1, 2);
}
```

```asm
; Name: cdecl.s

.file "cdecl.c"
.intel_syntax noprefix
.text
.globl callee
.type callee, @function
callee:
nop
ret ; 스택을 정리하지 않고 리턴합니다.
.size callee, .-callee
.globl caller
.type caller, @function
caller:
push 2 ; 2를 스택에 저장하여 callee의 인자로 전달합니다.
push 1 ; 1를 스택에 저장하여 callee의 인자로 전달합니다.
call callee
add esp, 8 ; 스택을 정리합니다. (push를 2번하였기 때문에 8byte만큼 esp가 증가되어 있습니다.)
nop
ret
.size caller, .-caller
.ident "GCC: (Ubuntu 11.3.0-1ubuntu1~22.04.1) 11.3.0"
.section .note.GNU-stack,"",@progbits
```

**x86 함수 호출 규약**

| 함수호출규약 | 사용 컴파일러 | 인자 전달 방식 | 스택 정리 | 적용 |
| :---: | :---: | :---: | :---: | :---: |
| stdcall | MSVC | Stack | Callee | WINAPI |
| cdecl | GCC, MSVC | Stack | Caller | 일반 함수 |
| fastcall | MSVC | ECX, EDX | Callee | 최적화된 함수 |
| thiscall | MSVC | ECX(인스턴스), Stack(인자) | Callee | 클래스의 함수 |

**x86-64 함수 호출 규약**

| 함수호출규약 | 사용 컴파일러 | 인자 전달 방식 | 스택 정리 | 적용 |
| :---: | :---: | :---: | :---: | :---: |
| MS ABI | MSVC | RCX, RDX, R8, R9 | Caller | 일반 함수, Windows Syscall |
| System ABI | GCC | RDI, RSI, RDX, RCX, R8, R9, XMM0-7 | Caller | 일반 함수 |

참고 : [Calling Convention](https://dreamhack.io/lecture/courses/54)

































