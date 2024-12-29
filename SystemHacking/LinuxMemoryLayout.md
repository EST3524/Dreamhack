# Linux Memory Layout
이 파일에는 리눅스의 메모리 구조에 대해 간단하게 정리한다.

# Overview
- Segment
- Code Segment
- Data Segment
- BSS Segment
- Stack Segment
- Heap Segment

## Segment
리눅스에서는 프로세스의 메모리를 크게 5가지의 세그먼트로 구분한다. 여기서 세그먼트란 적재되는 데이터의 용도별로 메모리의 구획을 나눈것인데, 크게 **Code, Data, BSS, Stack, Heap**으로 구분한다.

![linux_process_memory_segment](Img/linux_process_memory_segment.png)

메모리를 용도별로 나누면, 각 용도에 맞게 적절한 권한을 부여할 수 있다. 권한은 **rwx(읽기, 쓰기, 실행)** 이 존재하며, CPU는 메모리에 대해 권한이 부여된 행위만 할 수 있다.

### Code Segment
코드 세그먼트는 **실행 가능한 기계 코드**가 위치하는 영역으로, **Text Segment**라고도 불린다.  

프로그램이 동작하려면 코드를 실행할 수 있어야 하므로 이 세그먼트에는 읽기 권한과 실행 권한이 부여된다. **(r-x)** 반면 쓰기 권한이 있으면 공격자가 악의적인 코드를 삽입하기가 쉬워지므로, 대부분의 현대 운영체제는 이 세그먼트에서 쓰기 권한을 제거한다.

```c
int main() { return 31337; }
```

위의 정수 31337을 반환하는 main함수가 컴파일 되면 554889e5b8697a00005dc3라는 기계 코드로 변환되는데, 이 기계 코드가 코드 세그먼트에 위치하게 된다.

### Data Segment
데이터 세그먼트에는 **컴파일 시점에 값이 정해진 전역 변수 및 전역 상수**들이 위치한다. CPU가 이 세그먼트의 데이터를 읽을 수 있어야 하므로, 읽기 권한이 부여된다.  

데이터 세그먼트는 쓰기가 가능한 세그먼트와 쓰기가 불가능한 세그먼트로 다시 분류된다.  

- Data Segment
  - data (rw-)
  - rodata (r--)

쓰기가 가능한 data 세그먼트에는 전역 변수와 같이 프로그램이 실행되면서 **값이 변할 수 있는 데이터**들이 위치한다.  

반면 쓰기가 불가능한 rodata 세그먼트에는 프로그램이 실행되면서 **값이 변하면 안되는 데이터**들이 위치한다. 전역으로 선언된 상수가 여기에 포함된다.

```c
int data_num = 31337;                       // data
char data_rwstr[] = "writable_data";        // data
const char data_rostr[] = "readonly_data";  // rodata
char *str_ptr = "readonly";  // str_ptr은 data, 문자열은 rodata

int main() { ... }
```

위의 예시에서, 전역 변수 **data_num, data_rwstr**은 각각 정수 값 31337, 문자형 배열 "writable_data"의 첫 번째 문자 'w'의 주소를 값으로 갖는다. 이는 프로그램 실행 중 값이 변할 수 있으므로 data 세그먼트에 위치한다.  
그러나, 전역 변수 **data_rostr**은 키워드 const를 사용하여 data_rostr 과 "readonly_data" 모두 값이 변할 수 없도록 하였다. 따라서 rodata 세그먼트에 위치한다.   
또, 전역 변수 **str_ptr**은 문자열 "readonly"의 첫 번째 문자 'r'의 주소를 값으로 갖는다. 문자 배열과는 다르게 문자열은 프로그램이 실행되면서 값을 변경할 수 없다. 따라서, 주소를 저장하는 str_ptr은 data 세그먼트에, 변하지 않는 값인 "readonly" 문자열은 rodata 세그먼트에 위치하게 된다.
