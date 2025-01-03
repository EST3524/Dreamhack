# Calling Convention
이 파일에는 함수 호출 규약에 대해 정리한다.

# Overview
- Calling Convention
  - cdecl
  - SYSV

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

## SYSTEM V
리눅스는 SYSTEM V(SYSV) ABI를 기반으로 만들어졌다. SYSV ABI는 ELF 포맷, 링킹 방법, 함수 호출 규약 등의 내용을 담고 있다. 
