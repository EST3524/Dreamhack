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
x64 어셈블리 언어는 **Operation Code, Opcode(명령어)** 와 **Operand(피연산자)** 로 구성된다. x64에는 매우 많은 명령어가 존재하지만, 그 중 중요한 21개의 명령어는 다음과 같다.

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

메모리 피연산자는 **[]** 으로 둘러싸인 것으로 표현되며, 앞에 크기 지정자 **TYPE PTR**이 추가될 수 있다. 여기서 타입에는 **BYTE, WORD, DWORD, QWORD**가 올 수 있으며, 각각 1바이트, 2바이트, 4바이트, 8바이트의 크기를 지정한다.
