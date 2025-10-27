## Часть 3. Минимизация корпуса и сбор покрытия

В ходе выполнения данной части предлагается:
- минимизировать входной корпус из **части 2** и уменьшить размер оставшихся его элементов;
- собрать покрытие кода, достигаемое выбранной нами стратегией фаззинга

В ходе выполнения заданий 3 части у нас должны получиться следующие артефакты работы:
- скриншоты выполнения пунктов задания;
- минимизированный корпус для проектов из **части 2**;
- сгенерированный HTML-отчет с анализом покрытия проекта из части 2.

### Минимизация корпуса
#### Удаление элементов корпуса, не открывающих новые пути
Чтобы [сделать входной корпус уникальным](https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/fuzzing_in_depth.md#b-making-the-input-corpus-unique) используется утилита `afl-cmin`
Проверим на уникальность корпус, подготовленный нами в ходе выполнения части 2:
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

### Сбор покрытия
Для сбора покрытия воспользуемся средствами сбора покрытия, входящими в проект llvm: https://clang.llvm.org/docs/SourceBasedCodeCoverage.html
В качестве примера используется проект cppcheck

Требуется пересобрать исследуемый в части 2 проект с инструментацией для сбора покрытия:
```
CFLAGS="-fprofile-instr-generate -fcoverage-mapping" CXXFLAGS="-fprofile-instr-generate -fcoverage-mapping" CC=clang CXX=clang++ cmake .
CFLAGS="-fprofile-instr-generate -fcoverage-mapping" CXXFLAGS="-fprofile-instr-generate -fcoverage-mapping" CC=clang CXX=clang++ make
```

Определить место хранения файлов, использующихся для сбора покрытия:
```
mkdir cov
cd cov
export LLVM_PROFILE_FILE="$(pwd)/cppcheck.profraw"
```

Затем запустить исследумый проект подавая на вход все наработанные в ходе 5-минутного фаззинг-тестирования файлы:
```
./bin/cppcheck $(find "output/default/queue" -type f) 2>&1 > /dev/null
```

Проиндексировать сгенерированный профиль:
```
llvm-profdata merge -sparse cov/cppcheck.profraw -o cov/cppcheck.profdata
```

Экспортировать в формат lcov:
```
llvm-cov export -format=lcov -instr-profile=cov/cppcheck.profdata ./bin/cppcheck > cov/cppcheck.lcov
```

Сгенерировать HTML-отчет
```
genhtml cov/cppcheck.lcov -o cov/html
```

### Итог 3 части
В ходе выполнения задач 3 части мы научились минимизировать входной корпус и собирать покрытие кода.
