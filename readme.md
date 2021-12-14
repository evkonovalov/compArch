### Анализ изначальной функции

```c
#include <cstddef>
#include <cstdint>
#include <malloc.h>

void randomfill(uint64_t *out, size_t n, uint64_t x, uint64_t a, uint64_t c) {
    for(; n; n--){
        x = x * a + c;
        *out++ = x;
    }
}

int main(){
    int n = 100;
    uint64_t *out = (uint64_t*)malloc(sizeof(uint64_t) * n);
    randomfill(out,n,4,2,3);
}
```
В данной функции randomfill только один цикл. У него нет инварианта, а так же каждая итерация зависит от результата предидущей (переменная x).
При компиляции `g++ -O3 -fopt-info-vec-all main.cpp` выдает следующий ответ:
>6: 11: missed: couldn't vectorize loop

>7: 15: missed: not vectorized: unsupported use in stmt.

>5: 6: note: vectorized 0 loops in function.

>12: 5: note: vectorized 0 loops in function.

То есть как раз из-за того, что в каждой итерации используется x из предидущей g++ не может векторизовать цикл. Это и есть`unsupported use in stmt`.

### Новая функция, которую можно векторизовать

По сути x считается в каждой итерации считается рекурентно. Формула для n-ого члена:
$$x_n = x_0 a^n + c(\sum_i^n{a^i})$$
Для того, что бы посчитать n-ый член последовательности нам необходимо предварительно подсчитать степени a. Считать все было бы неоптимально, поэтому я решил подсчитать сначала фиксированное количество степеней. Назовем количество заранее подсчитанных степеней - блоком. Из-за особенностей архитектуры процессора он должен быть кратный 4. Зная эти степени для первой части формулы и частные суммы из второй части, умноженные на $$c$$, мы могли бы посчитать первые блок значений икс независимо. Такой цикл можно векторизовать. 
Теперь, когда мы посчитали первые блок значений, мы можем заменить первоначальный $$x_0$$ на самый большой из уже посчитанных и повторить процедуру. К сожалению этот цикл из-за зависимости данных векторизовать не удасться.
Таким образом мы получаем следующую программу:
```c
#include <cstddef>
#include <cstdint>
#include <malloc.h>

void randomfill(uint64_t *out, size_t n, uint64_t x, uint64_t a, uint64_t c) {
    size_t block_size = 64;
    uint64_t powers[block_size];
    uint64_t c_a_sums[block_size];
    powers[0] = a;
    c_a_sums[0] = c;
    //this cannot be vectorized
    for (size_t i = 1; i < block_size; i++) {
        powers[i] = powers[i - 1] * a;
        c_a_sums[i] = c_a_sums[i - 1] + c * powers[i - 1];
    }
    size_t blocks = n / block_size;
    size_t count_limit = block_size * blocks;
    for (size_t i = 0; i < count_limit; i += block_size) {
        //vectorized cycle
        for (size_t j = 0; j < block_size; j++) {
            out[i + j] = x * powers[j] + c_a_sums[j];
        }
        x = out[i + block_size - 1];
    }
    //this cannot be vectorized
    for (size_t i = count_limit; i < n; i++) {
        x = x * a + c;
        out[i] = x;
    }
}


int main(){
    int n = 100;
    uint64_t *out = (uint64_t*)malloc(sizeof(uint64_t) * n);
    randomfill(out,n,4,2,3);
}
```
При компиляции командой `g++ -O3 -fopt-info-vec-all main.cpp`  получаем следующий вывод:
>26: 42: missed: couldn't vectorize loop

>27: 15: missed: not vectorized: unsupported use in stmt.

>18: 26: missed: couldn't vectorize loop

>21: 24: missed: not vectorized: complicated access pattern.

>20: 30: optimized: loop vectorized using 16 byte vectors

>12: 26: missed: couldn't vectorize loop

>13: 33: missed: not vectorized, possible dependence between data-refs *powers.1_37[_11] and *powers.1_37[i_72]

>5: 6: note: vectorized 1 loops in function.

Таким образом мы получили функцию, которая лучше поддается векторизации, а следовательно работает эффективнее. Она задействует дополнительную память, но в пренебрежимо малых размерах. 
