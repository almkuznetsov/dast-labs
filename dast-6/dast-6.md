## Часть 6. Минимизация корпуса, санитайзеры и многопоточный фаззинг
В этой части мы погружаемся в детали фаззинга, в том числе описанные в документации AFL++: [fuzzing in depth](https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/fuzzing_in_depth.md)
Мы минимизируем входной корпус, рассмотрим сборку с санитайзерами и запуск фаззинга в несколько потоков.

В ходе выполнения данной части предлагается:
- минимизировать входной корпус из **части 2** и уменьшить размер оставшихся его элементов;
- собрать обертку для AFL++ из **части 4** с ASAN, MSAN, UBSAN;
- запустить многопоточный фаззинг с помощью AFL++ для всех получившихся исполняемых файлов;
- собрать обертку для libfuzzer из **части 4** с ASAN и UBSAN;
- запустить libfuzzer в несколько потоков.

В ходе выполнения заданий 6 части у нас должны получиться следующие артефакты работы:
- минимизированный корпус для проектов из **части 2**;
- скриншоты запущенного в несколько потоков фаззинга проекта из **части 4** с помощью AFL++;
- общая директория output, где хранятся результаты работы всех экземпляров фаззера;
- скриншоты запущенного фаззинга проекта из **части 4** в несколько потоков с помощью libfuzzer.

### Минимизация корпуса
#### Удаление элементов корпуса, не открывающих новые пути
Чтобы [сделать входной корпус уникальным](https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/fuzzing_in_depth.md#b-making-the-input-corpus-unique) используется утилита `afl-cmin`
Проверим на уникальность корпус, подготовленный нами в ходе выполнения части 3:
```shell
afl-cmin -i input -o input-unique -- ./program [...program's cmdline...] @@
```

#### Минимизация элементов корпуса
Для [уменьшения размера элементов корпуса](https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/fuzzing_in_depth.md#c-minimizing-all-corpus-files) используется утилита `afl-tmin`, которую необходимо вызывать отдельно для каждого файла

Минимизируем размер одного из полученных на предыдущем шаге файлов (на примере cppcheck из части 2):
```shell
afl-tmin -i input-unique/1.cpp -o input/1.cpp.min -- ./bin/cppcheck @@
```

Самостоятельно проверьте на уникальность подготовленный вами в части 2 корпус из 3 элементов, а затем уменьшите размер оставшихся файлов.

### Санитайзеры
[Санитайзеры](https://github.com/google/sanitizers) - инструменты комилятора, помогающие работать с программой (в том числе находить ошибки) с помощью допонительной инструментации. Мы уже использовали санитайзер сбора покрытия в части 3.

В рамках выполнения этого задания нам потребуются следующие санитайзеры:
- [AddressSanitizer (ASAN)](https://github.com/google/sanitizers/wiki/AddressSanitizer) (Для AFL++ - `AFL_USE_ASAN=1`, для libfuzzer - `-fsanitize=address`)
- [LeakSanitizer (LSAN)](https://github.com/google/sanitizers/wiki/AddressSanitizerLeakSanitizer) (Включается вместе с ASAN)
- [MemorySanitizer (MSAN)](https://github.com/google/sanitizers/wiki/MemorySanitizer) (Для AFL++ - `AFL_USE_MSAN=1`, для libfuzzer - `-fsanitize=memory`)
- [UndefinedBehaviorSanitizer (UBSAN)](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html) Для AFL++ - `AFL_USE_UBSAN=1`, для libfuzzer - `-fsanitize=undefined`

AFL++ имеет собственные переменные окружения для облегчения использования санитайзеров (см. [selecting sanitizers](https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/fuzzing_in_depth.md#c-selecting-sanitizers)). Для фаззинга с их помощью соберем отдельные исполняемые файлы под каждый вид санитайзеров (на примере обертки под AFL++ для pugixml из части 4):
```shell
afl-clang-lto++ wrapper.cpp src/pugixml.cpp -o wrapper-clear
AFL_USE_ASAN=1 afl-clang-lto++ wrapper.cpp src/pugixml.cpp -o wrapper-asan
AFL_USE_MSAN=1 afl-clang-lto++ wrapper.cpp src/pugixml.cpp -o wrapper-msan
AFL_USE_UBSAN=1 afl-clang-lto++ wrapper.cpp src/pugixml.cpp -o wrapper-ubsan
```

Для фаззинга с помощью libfuzzer соберем исполняемый файл с ASAN и UBSAN (на примере обертки под libfuzzer для pugixml из части 4):
```shell
clang++ -fsanitize=fuzzer,address,undefined wrapper.cpp src/pugixml.cpp -o wrapper-libfuzzer
```

### Многопоточный фаззинг
#### Многопоточный фаззинг в AFL++
Для масштабирования фаззинг-тестирования приходится запускать процессы фаззинга используя [все доступные ядра](https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/fuzzing_in_depth.md#c-using-multiple-cores)

Первый процесс будет основным - master-процессом, для его запуска добавляем к вызову `afl-fuzz` ключ `-M master`. Остальные процессы как правило называют slave-процессами и для них запуска используется ключ `-S <name>`. Главным для того, чтобы фаззеры работали параллельно и могли обмениваться корпусом является общая директория output, указываемая с флагом `-o`.

Запустите многопоточный фаззинг используя собранные с санитайзерами исполняемые файлы, полученные на предыдущем шаге. В результате должны быть запущены:
- master
- slave-clear (второй экземпляр, запущенный на "чистой" обертке)
- slave-asan (экземпляр, запущенный на обертке с ASAN)
- slave-msan (экземпляр, запущенный на обертке с MSAN)
- slave-ubsan (экземпляр, запущенный на обертке с UBSAN)

Если количество ядер не позволяет запустить сразу все указанные процессы, следует запустить мастер-процесс, а затем по очереди каждый из slave-процессов. Проверить свободные ядра можно, например, с помощью утилиты `afl-gotcpu`.

#### Многопоточный фаззинг в libfuzzer
Для запуска многопоточного фаззинга с помощью libfuzzer воспользуемся экспирементальным ключом `-fork=N`, где N-количество потоков, добавив этот ключ к вызову исполняемого файла.

### Итог 5 части
В ходе выполнения задач 5 части мы научились минимизировать входной корпус, рассмотрели сборку с санитайзерами и запуск фаззинга в несколько потоков.
