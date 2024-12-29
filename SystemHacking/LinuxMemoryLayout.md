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
코드 세그먼트는 **실행 가능한 기계 코드**가 위치하는 영역으로, **Text Segment**라고도 불린다. 프로그램이 동작하려면 코드를 실행할 수 있어야 하므로 이 세그먼트에는 읽기 권한과 실행 권한이 부여된다. **(r-x)** 반면 쓰기 권한이 있으면 공격자가 악의적인 코드를 삽입하기가 쉬워지므로, 대부분의 현대 운영체제는 이 세그먼트에서 쓰기 권한을 제거한다.

```c
int main() { return 31337; }
```

위의 정수 31337을 반환하는 main함수가 컴파일 되면 554889e5b8697a00005dc3라는 기계 코드로 변환되는데, 이 기계 코드가 코드 세그먼트에 위치하게 된다.

