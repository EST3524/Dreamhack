# x86 Assembly
이 파일에는 x86-64 어셈블리어 명령어들을 간단하게 정리한다.

# Overview
- Assembly
- x86-64 Assembly
- Data Transfer
- Arithmetic
- Logical
- Comparison
- Branch
- Stack
- Procedure
- System call

## Assembly
어셈블리어는 0과 1로만 구성되어 있는 기계어를 인간이 쉽게 이해할 수 있도록 만들어진 언어이다. 어셈블러를 통하여 어셈블리어를 기계어로 번역할 수 있고, 역어셈블러를 통하여 반대의 과정도 수행할 수 있다. 어셈블리어는 CPU가 사용하는 ISA가 무엇인지에 따라 달라진다.

## x86-64 Assembly
x64 어셈블리 언어는 **Operation Code, Opcode(명령어)** 와 **Operand(피연산자)** 로 구성된다.     
-> mov eax, 3 (eax에 3을 대입)  

x64에는 매우 많은 명령어가 존재하지만, 그 중 중요한 21개의 명령어는 다음과 같다.

| 명령 코드 | 명령어 |
| :---: | :---: |
| Data Transfer | mov, lea |
| Arithmetic | inc, dec, add, sub |
| Logical | and, or, xor, not |
| Comparison | cmp, test |
| Branch | jmp, je, jg |
| Stack | push, pop |
| Procedure | call, ret, leave |
| System call | syscall |

피연산자에는 다음과 같이 3가지 종류가 올 수 있다.
- Immediate Value
- Register
- Memory

메모리 피연산자는 **[]** 으로 둘러싸인 것으로 표현되며, 앞에 크기 지정자 **TYPE PTR**이 추가될 수 있다. 여기서 타입에는 **BYTE, WORD, DWORD, QWORD**가 올 수 있으며, 각각 **1바이트, 2바이트, 4바이트, 8바이트**의 크기를 지정한다.  

-> QWORD PTR [0x8048000] (0x8048000의 데이터를 8바이트만큼 참조)  
-> WORD PTR [rax] (rax가 가리키는 주소에서 데이터를 2바이트 만큼 참조)  

### Data Transfer
데이터 이동 명령어는 **어떤 값을 레지스터나 메모리에 옮기도록 지시**한다.

| mov dst, src | src에 들어있는 값을 dst에 대입 |
| :---: | :---: |
| mov rdi, rsi | rsi의 값을 rdi에 대입 |
| mov QWORD PTR [rdi], rsi | rsi의 값을 rdi가 가리키는 주소에 대입 |
| mov QWORD PTR [rdi+8*rcx], rsi | rsi의 값을 rdi+8*rcx가 가리키는 주소에 대입 | 

| lea dst, src | src의 유효 주소를 dst에 저장 |
| :---: | :---: |
| lea rsi, [rbx+8*rcx] | rbx+8*rcx를 rsi에 대입 |

레지스터는 자체적인 메모리 주소를 갖고 있지 않다. 여기서 의문이 생긴다. **mov QWORD PTR [rdi], rsi**과 **mov QWORD PTR [rdi+8*rcx], rsi**에서 rdi는 레지스터 자체이고 rdi+8\*rcx는 주소이다. 이렇게 생각한다면 [rdi]는 rdi라는 레지스터에 저장되어 있는 값(주소)를 읽어서 사용하겠다는 것이고, [rdi+8*rcx]는 rdi+8\*rcx에 저장되어 있는 값을 읽어서 사용하겠다는 것이 아닌 주소 자체를 사용하겠다는 것처럼 보인다. 하지만, 당연하게도 [rdi]에서 rdi는 사실 rdi에 저장되어 있는 값과 완벽하게 같다. rdi 자체가 그냥 주소인 것이다. 이 때문에 rdi+8\*rcx가 주소 계산식으로 동작할 수 있다. 그러므로, [] 안에 있는 것은 모두 주소라는 것을 알고 있어야 한다.  

mov 명령문에서 dst에 주소를 사용하려면 크기 지정자가 붙어야 하는 것과 달리, lea 명령문에서 크기 지정자가 붙지 않는 이유는 mov는 src에서 어느정도 크기 만큼 대입해야하는지 결정해야하는 것과 달리 lea는 유효 주소를 계산해서 저장하면 되기 때문에 크기 지정자가 붙을 필요 없이 자동으로 저장되기 때문이다. 명령어 자체가 유효 주소를 계산하도록 되어있기 때문에 크기를 직접 지정하는 의미가 없다는 뜻이다. 또, 크기 지정자는 dst이 아닌 src의 크기를 나타내는 것이다.

### Arithmetic
산술 연산 명령어는 **덧셈, 뺄셈, 곱셈, 나눗셈 연산을 지시**한다.

| add dst, src | dst에 src의 값을 더함 |
| :---: | :---: |
| add eax, 3 | eax += 3 |
| add ax, WORD PTR [rdi] | ax += *(WORD *)rdi |

| sub dst, src | dst에서 src의 값을 뺌 |
| :---: | :---: |
| sub eax, 3 | eax -= 3 |
| sub ax, WORD PTR [rdi] | ax -= *(WORD *)rdi |

| inc op | op의 값을 1 증가 시킴 |
| :---: | :---: |
| inc eax | eax += 1 |

| dec op | op의 값을 1 감소 시킴 |
| :---: | :---: |
| dec eax | eax -= 1 |

### Logical
논리 연산 명령어는 **and, or, xor, neg 등의 비트 연산을 지시**한다. 연산은 비트 단위로 이루어진다.  

**and dst, src : dst와 src의 비트가 모두 1이면 1, 아니면 0**

```asm
[Register]
eax = 0xffff0000
ebx = 0xcafebabe

[Code]
and eax, ebx

[Result]
eax = 0xcafe0000
```

**or dst, src : dst와 src의 비트 중 하나라도 1이면 1, 아니면 0**

```asm
[Register]
eax = 0xffff0000
ebx = 0xcafebabe

[Code]
or eax, ebx

[Result]
eax = 0xffffbabe
```

**xor dst, src : dst와 src의 비트가 서로 다르면 1, 같으면 0**

```asm
[Register]
eax = 0xffffffff
ebx = 0xcafebabe

[Code]
xor eax, ebx

[Result]
eax = 0x35014541
```

**not op : op의 비트 전부 반전**

```asm
[Register]
eax = 0xffffffff

[Code]
not eax

[Result]
eax = 0x00000000
```

### Comparison
비교 명령어는 **두 피연산자의 값을 비교하고 플래그를 설정**한다.  

**cmp op1, op2 : op1과 op2를 비교**  
cmp는 두 피연산자를 빼서 대소를 비교한다. 연산의 결과는 op1에 대입하지 않는다.

```asm
[Code]
1: mov rax, 0xA
2: mov rbx, 0xA
3: cmp rax, rbx ; ZF=1
```

**test op1, op2 : op1과 op2를 비교**  
test는 두 피연산자에 AND 비트연산을 취한다. 연산의 결과는 op1에 대입하지 않는다.

```asm
[Code]
1: xor rax, rax
2: test rax, rax ; ZF=1
```

### Branch
분기 명령어는 **rip**를 이동시켜 **실행 흐름을 바꾼다.**  

**jmp addr : addr로 rip를 이동시킨다.**

```asm
[Code]
1: xor rax, rax
2: jmp 1 ; jump to 1
```

**je addr : 직전에 비교한 두 피연산자가 같으면 점프**

```asm
[Code]
1: mov rax, 0xcafebabe
2: mov rbx, 0xcafebabe
3: cmp rax, rbx ; rax == rbx
4: je 1 ; jump to 1
```

**jg addr : 직전에 비교한 두 연산자 중 전자가 더 크면 점프**

```asm
[Code]
1: mov rax, 0x31337
2: mov rbx, 0x13337
3: cmp rax, rbx ; rax > rbx
4: jg 1  ; jump to 1
```

### Stack
x64 아키텍처에서 다음의 명령어로 스택을 조작할 수 있다.  

**push val** : val을 스택 최상단에 쌓음 

**연산**  
-> rsp -= 8  
-> [rsp] = val  

**예제**

```asm
[Register]
rsp = 0x7fffffffc400

[Stack]
0x7fffffffc400 | 0x0  <- rsp
0x7fffffffc408 | 0x0

[Code]
push 0x31337
```

**결과**

```asm
[Register]
rsp = 0x7fffffffc3f8

[Stack]
0x7fffffffc3f8 | 0x31337 <- rsp 
0x7fffffffc400 | 0x0
0x7fffffffc408 | 0x0
```

**pop reg** : 스택 최상단의 값을 꺼내서 reg에 대입  

**연산**  
-> rsp += 8  
-> reg = [rsp-8]  

**예제**

```asm
[Register]
rax = 0
rsp = 0x7fffffffc3f8

[Stack]
0x7fffffffc3f8 | 0x31337 <- rsp 
0x7fffffffc400 | 0x0
0x7fffffffc408 | 0x0

[Code]
pop rax
```

**결과**

```asm
[Register]
rax = 0x31337
rsp = 0x7fffffffc400

[Stack]
0x7fffffffc400 | 0x0 <- rsp 
0x7fffffffc408 | 0x0
```

### Procedure
컴퓨터 과학에서 프로시저는 특정 기능을 수행하는 코드 조각을 말한다. 프로시저를 사용하면 반복되는 연산을 프로시저 호출로 대체할 수 있어서 전체 코드의 크기를 줄일 수 있으며, 기능별로 코드 조각에 이름을 붙일 수 있게 되어 코드의 가독성을 크게 높일 수 있다.  

프로시저를 부르는 행위를 Call(호출)이라고 하며, 프로시저에서 돌아오는 것을 Return(반환)이라고 한다. 프로시저를 호출할 때는 프로시저를 실행하고 나서 원래의 실행 흐름으로 돌아와야 하므로, call 다음의 return Address를 스택에 저장하고 프로시저로 rip를 이동시킨다.  

x64 어셈블리 언어에는 프로시저의 호출과 반환을 위한 call, leave, ret 명령어가 있다.  

**call addr** : addr에 위치한 프로시저 호출  

**연산**  
-> push return_address  
-> jmp addr  

**예제**

```asm
[Register]
rip = 0x400000
rsp = 0x7fffffffc400 

[Stack]
0x7fffffffc3f8 | 0x0
0x7fffffffc400 | 0x0 <- rsp

[Code]
0x400000 | call 0x401000  <- rip
0x400005 | mov esi, eax
...
0x401000 | push rbp
```

**결과**

```asm
[Register]
rip = 0x401000
rsp = 0x7fffffffc3f8

[Stack]
0x7fffffffc3f8 | 0x400005  <- rsp
0x7fffffffc400 | 0x0

[Code]
0x400000 | call 0x401000
0x400005 | mov esi, eax
...
0x401000 | push rbp  <- rip
```

**leave** : 스택 프레임 정리  

**연산**  
-> mov rsp, rbp  
-> pop rbp

**예제**  

```asm
[Register]
rsp = 0x7fffffffc400
rbp = 0x7fffffffc480

[Stack]
0x7fffffffc400 | 0x0 <- rsp
...
0x7fffffffc480 | 0x7fffffffc500 <- rbp
0x7fffffffc488 | 0x31337 

[Code]
leave
```

**결과**  

```asm
[Register]
rsp = 0x7fffffffc488
rbp = 0x7fffffffc500

[Stack]
0x7fffffffc400 | 0x0
...
0x7fffffffc480 | 0x7fffffffc500
0x7fffffffc488 | 0x31337 <- rsp
...
0x7fffffffc500 | 0x7fffffffc550 <- rbp
```

**ret** : return address로 반환  

**연산**  
-> pop rip  

**예제**

```asm
[Register]
rip = 0x401008
rsp = 0x7fffffffc3f8

[Stack]
0x7fffffffc3f8 | 0x400005    <- rsp
0x7fffffffc400 | 0

[Code]
0x400000 | call 0x401000
0x400005 | mov esi, eax
...
0x401000 | mov rbp, rsp  
...
0x401007 | leave
0x401008 | ret  <- rip
```

**결과**

```asm
[Register]
rip = 0x400005
rsp = 0x7fffffffc400

[Stack]
0x7fffffffc3f8 | 0x400005
0x7fffffffc400 | 0x0    <- rsp

[Code]
0x400000 | call 0x401000
0x400005 | mov esi, eax   <- rip
...
0x401000 | mov rbp, rsp  
...
0x401007 | leave
0x401008 | ret
```

스택
