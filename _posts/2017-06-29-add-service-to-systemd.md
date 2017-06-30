---

title:  'Как добавить свой сервис в systemd'
categories: development
tags: [ruby, notes]
---

## Intro

На моей Raspberry Pi играет MPD и так же там поднято 3 самописных
сервера: HTTP API на Go для него, приложение на React.js, использующее
это API и Telegram бот.

При перезагрузке все это, разумеется, падает и нужно запускать все
заново. Тут-то и приходит на помощь systemd.

<!--more-->

## Добавляем свой скрипт в автозагрузку systemd

Приведу пример своего скрипта, который просто выполняет одну команду:
`amixer set PCM 0db`

Создаем сервис в `/etc/systemd/system/set-volume.service` со следующим
содержимым:

```
[Unit]
Description=Set alsa volume gain to 0db

[Service]
Type=oneshot
ExecStart=/home/pi/set-volume.sh

[Install]
WantedBy=multi-user.target
```

`oneshot` говорит о том, что systemd должен просто выполнить ExecStart
и продолжить выполнение остальных сервисов. Красноречивое название.

Далее смотрим статус сервиса:

```
$ sudo systemctl status set-volume
set-volume.service - Set alsa volume gain to 0db
   Loaded: loaded (/etc/systemd/system/set-volume.service; disabled)
   Active: inactive (dead)
```

Обратите внимание на `disabled`, systemd видит наш сервис, но никак
использует его. Выполняем: `sudo systemctl enable set-volume`.

Вот и все.

Можем вручную запустить скрипт через systemd: `sudo systemctl start
set-volume`. После перезапуска сервис запустится автоматически.

### NOTE

Скорее всего, можно обойтись и без отдельного файла для скрипта, а
передавать аргументы в amixer через сам файл юнита.

## Добавляем "неумирающий" сервис

Т.к. мой API для MPD является багованным недоделанным говном, имеет
смысл сделать его сервисом, который будет запущен даже после падений.

Вот содержимое файла `/etc/systemd/system/mpd-api.service`:

```
[Unit]
Description=MPD API

[Service]
ExecStart=/home/pi/mpd_api/current/mpd_api
WorkingDirectory=/home/pi/mpd_api/current
Restart=always

[Install]
WantedBy=multi-user.target
```

Тип сервиса нам не нужен (по-умолчанию используется `simple`), интерес
представляет только `Restart=always`.

Активируется он также, как и предыдущий скрипт. Но теперь systemd
будет следить за тем, чтобы сервис всегда работал, т.е. каждый раз,
когда процесс умирает, сервис будет запущен заново (так что осторожнее
со скриптами, а то они будут запускаться по 100500 раз).


## Outro

Я вообще ничего не понимаю в systemd, просто мне потребовалось
добавить пару сервисов в автозагрузку. Если помимо этого ничего не
нужно, то инфы выше более чем достаточно.
