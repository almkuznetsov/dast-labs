# Формат сдачи DAST

Результат сдаётся одним PR в выданный репозиторий. Все проверяемые материалы
должны находиться в ветке PR, а не только в его комментариях или внешних сервисах.

## Структура результата

```text
submission/
├── submission.json
└── artifacts/
    ├── input/
    ├── out/
    ├── logs/
    │   ├── build.txt
    └── screenshots/
        ├── build.png
        └── fuzzing.png
```

В `input/` сохраните начальный корпус, с которым запускался фаззер, а в `out/`
скопируйте весь каталог результатов AFL++, сохранив имена файлов и структуру
каталогов экземпляров фаззера. В него входят `queue`, найденные `crashes` и
`hangs`, `fuzzer_stats`, `plot_data` и другие созданные AFL++ файлы.

## submission.json

```json
{
  "schema_version": 1,
  "assignment": "dast-1",
  "variant": 2,
  "environment": {
    "os": {
      "id": "значение ID из /etc/os-release",
      "version_id": "значение VERSION_ID из /etc/os-release"
    },
    "aflplusplus_version": "точная версия AFL++",
    "compiler": "afl-clang-lto",
    "compiler_version": "точная версия компилятора и используемого LLVM"
  },
  "project": {
    "name": "gzip",
    "version": "версия собранного проекта",
    "source": {
      "repository": "URL исходного Git-репозитория",
      "commit": "полный SHA исходного коммита"
    }
  },
  "build": {
    "cwd": "/work/gzip",
    "command": "точные команды конфигурирования и сборки с переменными окружения",
    "artifact": "submission/artifacts/logs/build.txt"
  },
  "fuzzing": {
    "cwd": "/work/gzip",
    "command": "точная команда afl-fuzz с опциями исследуемой программы",
    "input": "submission/artifacts/input",
    "output": "submission/artifacts/out",
    "stats": "submission/artifacts/out/default/fuzzer_stats",
    "log": "submission/artifacts/logs/fuzz.txt",
    "duration_seconds": 300
  }
}
```

Строки-подсказки замените фактическими значениями. Все поля примера обязательны,
за исключением альтернативного способа указать источник проекта, описанного
ниже. `variant` — число `1` для `tar` или `2` для `gzip`; `project.name` должно
соответствовать выбранному варианту.

В `environment.os` укажите `ID` и `VERSION_ID` из вывода `cat /etc/os-release` в среде, где запускался анализатор: на ВМ, физической машине или внутри контейнера. Оба значения записываются строками без внешних кавычек из `os-release`; прикладывать весь вывод не требуется. При использовании контейнера можно дополнительно указать необязательное поле `environment.container` с именем и версией образа, например `"container": "registry.altlinux.org/alt/alt:p11"`

## Предметные скриншоты

В каждой части указаны этапы, на которых нужно сделать скриншоты. Они сохраняются под указанными именами в `submission/artifacts/screenshots/`.

Все скриншоты сдаются как исходные PNG размером не менее 1000×600, без редактирования после съёмки.
