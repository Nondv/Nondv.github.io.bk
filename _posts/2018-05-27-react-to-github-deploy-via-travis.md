---

title:  'Деплой react.js приложения на github.io с помощью Travis CI'
categories: development
tags: [js, travis, node, react, javascript, guide]
---

## Intro

Тут должно было быть многабукоф про приложение, которое я начал делать, но
я вовремя решил, что это лишнее.

Задача: есть webpack приложение, созданное с помощью `create-react-app`, нужно
его разместить на github.io (github-pages).

Поскольку github.io подходит только для статических и Jekyll сайтов, то webpack
вроде как в пролете, но мы ведь можем компилировать приложение в одном месте, а
потом его уже пушить на гитхаб. Как выяснилось, Travis CI это уже умеет.

<!--more-->

## Решение

Я создал два репозитория:

* Код приложения
* Репозиторий для скомпилированных статических файлов с включенным github.io

В основном репозитории и настраиваем Travis.
Вот так выглядит *мой* (не шаблонный) `.travis.yml`:

```yaml
language: node_js
node_js:
- '9.11'

cache:
  directories:
    - "node_modules"

script:
  - yarn test
  - yarn build
  - echo storm-picker.xyz > build/CNAME

deploy:
  provider: pages
  github-token: $GITHUB_TOKEN  # Set in the settings page of your repository, as a secure variable
  committer-from-gh: true
  skip-cleanup: true
  keep-history: true
  local-dir: build
  repo: Nondv/storm-picker-compiled
  target-branch: master
  on:
    branch: master
```

### Замечания к конфигу

В `script` есть странная команда `echo storm-picker.xyz > build/CNAME`.
Она нужна в случае использования собственного домена, т.к. github использует
этот файл для того, чтобы хранить эту настройку (уж не знаю, почему они так
сделали), а при деплое все содержимое репозитория заменится на свежий билд (в
который CNAME не входит).

В настройках деплоя заслуживают внимания:

* `keep_history` - без этой опции travis будет просто делать хард ресет
  репозитория, оставляя там лишь один коммит.
* `local-dir` - в ней мы указываем, какую именно папку нужно использовать как
  код для деплоя. Указываем в ней скомпилированный код.
* `repo` - теоретически можно хранить все в одном репозитории. Github позволяет
  выкатывать на github.io только файлы в директории `docs/`, соответственно,
  если в тревисе билдить все в `/docs`, то получится так, что разработчик делает
  изменения в коде, а тревис их билдит и коммитит. Важно указывать владельца репозитория.
* `target_branch` - в какую ветку пушить. В моем репозитории используется
  `master` вместо дефолтной `gh-pages`.

## Возможные проблемы

Я столкнулся только с одной проблемой. Т.к. изначально под сайт выделяется
`yourname.github.io/projectname`, то возникают проблемы с путями. Во время билда
выдается подсказка, что можно указать `"homepage":
"yourname.github.io/projectname"` в `package.json`. Однако если вы используете
статические файлы (`public/`), то ссылки вида `/img.png` перестанут
работать, т.к. они будут указывать на `yourname.github.io/img.png` вместо
`yourname.github.io/projectname/img.png`. Я не думал, как с этим справиться,
т.к. у меня отдельный домен.


## Заключение

Я обожаю github.io <3
