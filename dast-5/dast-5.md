## Часть 5. Обертки, persistent mode и libfuzzer
Помимо фаззинга приложения через "толстый вход" как единого целого (whole application fuzzing), часто используется фаззинг конкретных функций, для чего разрабатываются фаззинг-обертки.
Для увеличения скорости фаззинг-тестирования также используются различные дополнительные инструменты, в том числе [persistent mode AFL++](https://github.com/AFLplusplus/AFLplusplus/blob/stable/instrumentation/README.persistent_mode.md) ([типовой пример использования всех функций persistent mode](https://github.com/AFLplusplus/AFLplusplus/blob/stable/utils/persistent_mode/persistent_demo_new.c))

В ходе выполнения данной части предлагается:
- разработать фаззинг-обертку для выделенной функции, запустить и зафиксировать скорость фаззинга;
- добавить shared memory fuzzing, запустить и зафиксировать скорость фаззинга;
- добавить deferred init, запустить и зафиксировать скорость фаззинга;
- добавить persistent mode, запустить и зафиксировать скорость фаззинга;
- переписать обертку под использование libfuzzer, запустить и зафиксировать скорость фаззинга;
- пересобрать libfuzzer-обертку с использованием AFL++, запустить и зафиксировать скорость фаззинга.

Важно отметить, что скорость фаззинга не является абсолютным показателем и, очевидно, зависит от множества вещей, в том числе производительности конкретной системы. В данном задании мы используем скорость не как абсолютный показатель качества, а как относительный показатель, сравнивая значения которого между собой можно получить некую информацию об эффективности применяемых методов

Варианты:
1. [nlohmann-json](https://github.com/nlohmann/json), функция parse;
2. [tinyxml2](https://github.com/leethomason/tinyxml2), функция Parse.

В ходе выполнения заданий 3 части у нас должны получиться следующие артефакты работы:
- скриншоты запущенного фаззинга на каждой итерации;
- код оберток на каждой итерации;
- общая информация по скорости фаззинга на каждой итерации.

### Обертки
Для фаззинга конкретной функции разрабатывается файл wrapper.cpp, который принимает на вход (через cin) данные от фаззера. Чтобы определить их размер можем воспользоваться `vector`, что не очень оптимально, но подходит для демонстрации.

Здесь и далее в качестве примера используется фаззинг функции `load_buffer()` из проекта [pugixml](https://github.com/zeux/pugixml). Запишем полученные из `cin` данные в `buffer`, который затем передадим на вход функции `load_buffer()`:
```cpp
#include "src/pugixml.hpp"
#include <iostream>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <vector>


int main() {
    std::vector<char> buffer(std::istreambuf_iterator<char>(std::cin), {});
	pugi::xml_document doc;
	doc.load_buffer(buffer.data(), buffer.size());
	return 0;
}
```
Для сборки этой цели необходимо использовать компилятор из состава AFL++:
```shell
afl-clang-lto++ wrapper.cpp src/pugixml.cpp -o wrapper
```

### Shared memory fuzzing
Ускорим фаззинг путем загрузки входных данных из разделяемой памяти, без необходимости чтения cin. Для этого воспользуемся shared memory возможностями AFL++.

Добавим инициализацию разделяемой памяти (`__AFL_FUZZ_INIT()` перед `main()`) и присвоим уже использованным переменным новые значения (`__AFL_FUZZ_TESTCASE_BUF` и `__AFL_FUZZ_TESTCASE_LEN`):
```cpp
[...]

__AFL_FUZZ_INIT();

int main() {
    unsigned char *data = __AFL_FUZZ_TESTCASE_BUF;
    ssize_t size = __AFL_FUZZ_TESTCASE_LEN;
    [...]
    doc.load_buffer(data, size);
    [...]
}
```

### Deferred init
Для ускорения фаззинга можно также отложить момент форка нового процесса. В нашем случае это будет не так полезно, поскольку никакой сложной инициализации перед вызовом функции `load_buffer()` не требуется, но тем не менее добавим указание на момент, до которого можно отложить форк (`__AFL_INIT()`):
```cpp
[...]
int main() {

    __AFL_INIT();

    unsigned char *data = __AFL_FUZZ_TESTCASE_BUF;
    [...]
}
```

### Persistent mode
Наконец, можно разрешить повторять вызов функции большее количество раз без необходимости каждый раз создавать форк процесса. Для этого добавим цикл, вызывающий функцию без форка процесса (`while (__AFL_LOOP(UINT8_MAX))`):
```cpp
[...]

int main() {

    __AFL_INIT();

    unsigned char *data = __AFL_FUZZ_TESTCASE_BUF;

    while (__AFL_LOOP(UINT8_MAX)) {
        ssize_t size = __AFL_FUZZ_TESTCASE_LEN;
        [...]
    }

    return 0;
}
```

### Libfuzzer
Для написания [libfuzzer](https://llvm.org/docs/LibFuzzer.html)-обертки функция `main` заменяется на `extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {...}`:
```cpp
#include "src/pugixml.hpp"
#include <iostream>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size){
    pugi::xml_document doc;
    doc.load_buffer(data, size);
	return 0;
}
```

Поскольку libfuzzer является частью проекта LLVM, для сборки такой обертки используется обычный компилятор `clang++` с ключом `-fsanitize=fuzzer`, а фаззинг запускается путем простого исполнения получившегося бинарного файла

### AFL++ & Libfuzzer
Формат оберток libfuzzer по сути является стандартом для фаззинг-оберток, и AFL++ также [умеет с ним работать](https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/fuzzing_in_depth.md#g-libfuzzer-fuzzer-harnesses-with-llvmfuzzertestoneinput).

Убедитесь в этом, пересобрав libfuzzer-обертку компилятором afl-clang-lto++ с использованием того же ключа `-fsanitize=fuzzer`, а затем запустив фаззинг, как и в предыдущих примерах, с помощью `afl-fuzz`.

### Итог 5 части
В ходе выполнения заданий 5 части мы научились разрабатывать фаззинг-обертки для тестирования конкретных функций программы, а также изучили способы ускорения фаззинг-тестирования.

Также мы познакомились с новым фаззером - Libfuzzer, разработали обертку для его запуска и проверили её совместимость с уже известным нам AFL++.
