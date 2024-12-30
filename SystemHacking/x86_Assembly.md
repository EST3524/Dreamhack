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

