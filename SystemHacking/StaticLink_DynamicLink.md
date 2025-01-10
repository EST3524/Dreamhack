# Static Link vs. Dynamic Link
이 파일에는 Static Link, Dynamic Link에 대해서 정리한다.

# Overview

- Library

- Link
	- Static Link
	- Dynamic Link

- PLT & GOT
	- PLT
	- GOT

## Library
라이브러리는 컴퓨터 시스템에서, 프로그램들이 함수나 변수를 공유해서 사용할 수 있게 한다. C의 표준 라이브러리인 **libc**는 ubuntu에 기본적으로 탑재된 라이브러리이며, **/lib/x86_64-linux-gnu/libc.so.6**에 있다.  

라이브러리는 크게 **Static Library**, **Dynamic Library**로 구분된다.

## Link
링크는 컴파일의 마지막 단계로, 프로그램에서 어떤 라이브러리의 함수를 사용한다면, 호출된 함수와 실제 라이브러리의 함수가 링크 과정에서 연결된다.

```c
// Name: hello-world.c
// Compile: gcc -o hello-world hello-world.c

#include <stdio.h>

int main() {
  puts("Hello, world!");
  return 0;
}
```

```c
// Path: /usr/include/stdio.h

...
/* Write a string, followed by a newline, to stdout.

   This function is a possible cancellation point and therefore not
   marked with __THROW.  */
extern int puts (const char *__s);
...
```

리눅스에서 C 소스 코드는 전처리, 컴파일, 어셈블 과정을 거쳐 ELF형식을 갖춘 **Object file**로 번역된다. 다음 명령어로 hello-world.c를 어셈블할 수 있다.

```bash
$ gcc -c hello-world.c -o hello-world.o
```

오브젝트 파일은 실행 가능한 형식을 갖추고 있지만, 라이브러리 함수들의 정의가 어디 있는지 알지 못하므로 실행은 불가능하다. 다음 명령어를 실행해보면, puts의 선언이 stdio.h에 있어서 **Symbol**로는 기록되어 있지만, 심볼에 대한 자세한 내용은 하나도 기록되어 있지 않다. 심볼과 관련된 정보들을 찾아서 최종 실행 파일에 기록하는 것이 링크 과정에서 하는 일 중 하나이다.

```bash
$ readelf -s hello-world.o | grep puts
    11: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND puts
```

다음은 예제를 완전히 컴파일한 것인데, libc에서 puts의 정의를 찾아 연결한 것을 확인할 수 있다.

```bash
$ gcc -o hello-world hello-world.c
$ readelf -s hello-world | grep puts
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5 (2)
    46: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@@GLIBC_2.2.5
$ ldd hello-world
        linux-vdso.so.1 (0x00007ffec3995000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fee37831000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fee37e24000)
```

여기서 libc를 같이 컴파일하지 않았음에도 libc에서 해당 심볼을 탐색한 것은, libc가 있는 /lib/x86_64-linux-gnu/가 표준 라이브러리 경로에 포함되어 있기 때문이다. gcc는 소스 코드를 컴파일할 때 표준 라이브러리의 라이브러리 파일들을 모두 탐색한다. 다음 명령어들로 표준 라이브러리의 경로를 확인할 수 있다.

```bash
$ ld --verbose | grep SEARCH_DIR | tr -s ' ;' '\n'
SEARCH_DIR("=/usr/local/lib/x86_64-linux-gnu")
SEARCH_DIR("=/lib/x86_64-linux-gnu")
SEARCH_DIR("=/usr/lib/x86_64-linux-gnu")
SEARCH_DIR("=/usr/lib/x86_64-linux-gnu64")
SEARCH_DIR("=/usr/local/lib64")
SEARCH_DIR("=/lib64")
SEARCH_DIR("=/usr/lib64")
SEARCH_DIR("=/usr/local/lib")
SEARCH_DIR("=/lib")
SEARCH_DIR("=/usr/lib")
SEARCH_DIR("=/usr/x86_64-linux-gnu/lib64")
SEARCH_DIR("=/usr/x86_64-linux-gnu/lib")
```

링크를 거치고 나면 프로그램에서 puts를 호출할 때, puts의 정의가 있는 libc에서 puts의 코드를 찾고, 해당 코드를 실행하게 된다.

## Static Link
정적 링크는 정적 라이브러리를 링크하는 것으로, 링크 과정에서 바이너리에 정적 라이브러리의 필요한 모든 함수를 포함시킨다. 따라서 해당 함수를 호출할 때, 라이브러리를 참조하는 것이 아니라 자신의 함수를 호출하는 것처럼 호출할 수 있다.  

정적 링크 시 컴파일 옵션에 따라 include한 헤더의 함수가 모두 포함될 수도 있고, 그렇지 않을 수도 있다.

## Dynamic Link
동적 링크는 동적 라이브러리를 링크하는 것으로, 동적 링크된 바이너리를 실행하면 동적 라이브러리가 프로세스의 메모리에 매핑된다. 실행 중에 라이브러리의 함수를 호출하면 매핑된 라이브러리에서 호출할 함수의 주소를 찾고, 그 함수를 실행한다.































