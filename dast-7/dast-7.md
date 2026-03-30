
## Часть 7. Нативный фаззинг в Golang и gofuzz
Для фаззинга проектов на Golang как правило используется либо встроенный в golang механизм фаззинга (далее testing.F), либо standalone-фаззер gofuzz.

В ходе выполнения данной части предлагается:
- запустить имеющуюся обертку для фаззинга с помощью gofuzz;
- написать аналогичную обертку для фаззинга той же функции с помощью testing.F.

В ходе выполнения заданий 7 части у нас должны получиться следующие артефакты работы:
- скриншоты процесса выполнения задания;
- две обертки для фаззинга с помощью gofuzz и testing.F.

### Gofuzz
В качестве целевой функции будем рассматривать функцию ParseImageName проекта Kubernetes - https://github.com/kubernetes/kubernetes/blob/master/pkg/util/parsers/parsers.go#L32. Обертка для её фаззинга с помощью gofuzz есть в проекте cncf - https://github.com/cncf/cncf-fuzzing/blob/main/projects/kubernetes/parser_fuzzer.go#L216

Установить gofuzz для запуска этой фаззинг-цели можно с помощью
```shell
go install github.com/dvyukov/go-fuzz/go-fuzz@latest github.com/dvyukov/go-fuzz/go-fuzz-build@latest
export PATH="$PATH:$(go env GOPATH)/bin"
```

После этого можно перенести код обертки, например, в kubernetes (имеется ввиду уже оригинальный https://github.com/kubernetes/kubernetes) test/fuzz/gofuzz/fuzz.go, откуда собрать её с помощью
```shell
GOWORK=off GOFLAGS='-mod=mod' go-fuzz-build -o parseimagename.zip k8s.io/kubernetes/test/fuzz/gofuzz
```
И затем запустить с помощью `gofuzz`.

### Testing.F
В golang также имеется встроенный механизм фаззинга - testing.F. Изучите туториал https://go.dev/doc/tutorial/fuzz и добавьте в файл parsers_test.go обертку для фаззинга той же функции с помощью testing.F. Запустите её минимум на 2 минуты.

Дополнительно:
- В чем разница, между подходом к фаззингу в gofuzz и testing.F?

### Итог 7 части
В ходе выполнения задач 7 части мы познакомились с двумя механизмами фаззинга Golang - gofuzz и testing.F.
