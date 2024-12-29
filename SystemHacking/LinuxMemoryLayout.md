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
