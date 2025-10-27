## Часть 3. Сбор покрытия
Для оценки результатов фаззинг-тестирования требуется:
- собрать покрытие кода, достигаемое выбранной нами стратегией фаззинга

В ходе выполнения заданий 3 части у нас должны получиться следующие артефакты работы:
- скриншоты выполнения пунктов задания;
- сгенерированный HTML-отчет с анализом покрытия проекта из части 2.

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
