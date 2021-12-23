### Задание 2. Функция hexdump
```c
#include <stdio.h>
#include <stdlib.h>
#include <stdint-gcc.h>
#include <string.h>

// 0x5c - '/'
// 0x78 - 'x'
// 0x30 - '0'
// 0x41 - 'A'

void hexdump(char* restrict out, char* restrict in, int n){
    for (int i = 0; i < n; i++){
        unsigned char first = in[i] / 16;
        unsigned char second = in[i] % 16;
        uint32_t word = 0x0000785C;
        word |= ((first < 10 ? 0x30 + first : 0x41 + first  - 10) << 16);\
        word |= ((second < 10 ? 0x30 + second : 0x41 + second - 10) << 24);\
        memcpy(out + sizeof(uint32_t) * i, &word, sizeof(uint32_t));\
    }
}
```
word = 0x0000785C кладет \x в обратном порядке, так как на современных системах используется правило little endian.
Так же кладется первая и вторая цифра 16-ричного представления символа.

При компиляции `gсс -O3 -fopt-info-vec-all main.c` выдает следующий ответ:
>12: 5: optimized: loop vectorized using 16 byte vectors
>
>11: 6: note: vectorized 1 loops in function.

Таким образом функция векторизируется. 
