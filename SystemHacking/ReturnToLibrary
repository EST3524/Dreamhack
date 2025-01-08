# Return to Library

## Program
익스플로잇 대상 프로그램은 다음과 같다.

```c
// Name: rtl.c
// Compile: gcc -o rtl rtl.c -fno-PIE -no-pie

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

const char* binsh = "/bin/sh";

int main() {
  char buf[0x30];

  setvbuf(stdin, 0, _IONBF, 0);
  setvbuf(stdout, 0, _IONBF, 0);

  system("echo 'system@plt'");

  printf("[1] Leak Canary\n");
  printf("Buf: ");
  read(0, buf, 0x100);
  printf("Buf: %s\n", buf);

  printf("[2] Overwrite return address\n");
  printf("Buf: ");
  read(0, buf, 0x100);

  return 0;
}
```

[Return to Library](https://dreamhack.io/wargame/challenges/353)

## Analyze
프로그램은 Return to Library 공격을 실습하기 위해 제작되었다.

```c
const char* binsh = "/bin/sh";
```

위의 코드에서는 







