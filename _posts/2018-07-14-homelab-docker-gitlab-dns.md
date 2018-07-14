---
title:  'Настройка домашней среды для разработки (docker + gitlab + DNS)'
categories: development
tags: [gitlab, environment]
---

Пост доступен на [Хабрахабре](https://habr.com/post/417179/).

## Intro

Не смог придумать подходящее название для поста, поэтому кратко опишу, о чем
будет идти речь.

У большинства из нас есть какие-нибудь мелкие личные поделки, которые не выходят
за рамки наших домов. Кто-то хостит их на рабочем компьютере, кто-то - на
Heroku, кто-то - на VPS, а у кого-то есть домашний сервер. На реддите даже есть
сообщество [r/homelab](https://www.reddit.com/r/homelab/), в котором люди
обсуждают разные железки и софт для т.н. *домашней лаборатории*.

Я не настолько увлечен этим вопросом, но у меня дома стоит Intel NUC, который
проигрывает музыку с NAS с помощью [MPD](https://www.musicpd.org). Помимо MPD на
нем крутятся мои мелкие поделки, которые помогают мне с ним работать: ныне
мертвый бот для телеграма, HTTP API на синатре и корявенький фронтенд для него.

В посте я без особых подробностей (многих из которых сам не понимаю) опишу
процесс установки DNS-сервера для работы с доменными именами для сервисов, схему
одновременной работы нескольких сервисов с помощью  Docker и установку Gitlab с
CI. Ничего нового вы не узнаете, но вдруг кому-нибудь пригодится этот "гайд". К
тому же я бы хотел услышать предложения по поводу того, как можно было бы
сделать это проще/элегантнее/правильнее.

<!--more-->

Изначально код моих сервисов лежал на битбакете/гитхабе и после создания
докер-образов мне нужно было зайти под SSH и запустить пару скриптов, которые
создавали/обновляли контейнеры с сервисами. Поймал себя на мысли, что вижу
мелкий раздражающий баг в приложении, который я не исправляю только потому, что
мне лень производить всю эту процедуру. Очевидно, что пора было автоматизировать
все. Тут-то и пришла мысль установки Gitlab + CI.

## Локальные домены с помощью DNS

Все контейнеры создавались с флагом `--network=host` для простоты - достаточно
было использовать разные порты в приложениях. Однако с ростом количества
сервисов запоминать, какое приложение какой порт использует. Да и вводить каждый
раз IP-адрес с портом в браузере не очень красиво, поэтому прежде чем
устанавливать гитлаб я решил разобраться с хостингом нескольких приложений на
одном сервере.

Идея простая: настраиваем DNS, скармливаем его роутеру, устанавливаем Nginx и
с помощью его конфигурации перенаправляем запросы на разные порты в зависимости
от домена. Это же позволит не заморачиваться с портами при разработке,
т.к. контейнеры начнут использовать `--publish` вместо `--network=host`.

При установке использовался
[этот гайд](https://www.itzgeek.com/how-tos/linux/debian/configure-dns-server-on-debian-9-ubuntu-16-04.html).
В нем настройка производится для Ubuntu 16.04, у меня же Debian.

Дальнейшие действия производятся от пользователя `root`.

Первым делом устанавливаем `bind9` и утилиты:

```bash
apt-get install -y bind9 bind9utils bind9-doc dnsutils
```

Далее нам нужно настроить доменную зону. Для этого добавяем в файл
`/etc/bind/named.conf.local` следующее:

```
zone "nondv.home" IN { // Желаемое домееное имя
     type master;
     file "/etc/bind/fwd.nondv.home.db"; // Forward lookup file
     allow-update { none; }; // Since this is the primary DNS, it should be none.
};
```

Также в гайде добавляется конфигурация для reverse lookup, но, честно говоря, я
не особо понимаю, для чего это нужно, поэтому не стал этого делать.

Теперь создаем файл `/etc/bind/fwd.nondv.home.db`:

```
$TTL    604800
@       IN      SOA     ns1.mydomain.home. root.mydomain.home. (
                             20         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

;Name Server Information
        IN       NS     ns1.nondv.home.

;IP address of Name Server
ns1     IN       A      192.168.0.3

;A - Record HostName To Ip Address
nuc     IN       A      192.168.0.3
gitlab  IN       A      192.168.0.3
mpd     IN       A      192.168.0.3
@       IN       A      192.168.0.3
```

Далее перезапускаем bind9 и устанавливаем автозапуск:

```bash
systemctl restart bind9
systemctl enable bind9
```

Обратите внимание, что я использовал `.home` вместо `.local`. Это было сделано
потому, что домен `nondv.local` не резолвился без поддоменов. Ну, точнее `dig`
распознавал его нормально, но браузеры и `curl` - нет. Как мне объяснил коллега,
это скорее всего связано с разным софтом вроде Bonjour (мой рабочий ноутбук с
яблоком на крышке). В общем, с доменом `.home` подобных проблем быть не должно.

Собственно, это все. Я после этого добавил DNS как первичный в роутер и
переподключился к нему (чтобы автоматически обновился файл
`/etc/resolve.conf`).

## Nginx

Как я уже сказал, чтобы иметь возможность обращаться по HTTP на 80 порт ко всем
сервисам одновременно, нам нужно настроить Nginx так, чтобы он проксировал
запросы на разные порты в зависимости от домена.

Документация по образу nginx доступна на сайте
[Docker Hub](https://hub.docker.com/_/nginx/).

Подготовим основной файл конфигурации `/srv/nginx/nginx.conf`:

```
user  nginx;
worker_processes  1;
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;
events {
    worker_connections  1024;
}
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    keepalive_timeout  65;
    server {
      listen 80;
      server_name nondv.home;
      rewrite ^/$ http://mpd.nondv.home redirect; # основной домен, на который будут пеправляться ненастроенные запросы
    }
    include /etc/nginx/conf.d/*.conf;
}
```

Далее настроим домены. Я покажу только один:

```
# /srv/nginx/conf.d/gitlab.conf
server {
  listen       80;
  server_name  gitlab.nondv.home;

  location / {
    proxy_pass http://127.0.0.1:3080;
  }
}

```

Запуск контейнера производится командой:

```bash
docker run --detach \
           --network host \
           --name nginx \
           --restart always \
           --volume /srv/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
           --volume /srv/nginx/conf.d:/etc/nginx/conf.d:ro \
           nginx:alpine
```


Вот и все, теперь HTTP запросы на 80 порт будут отлавливаться с помощью nginx и
перенаправляться на нужный порт.

## Gitlab

Здесь все просто по [официальному руководству](https://docs.gitlab.com/omnibus/docker/):

```
docker run --detach \
           --hostname gitlab.nondv.home \
           --publish 3080:80 --publish 3022:22 \
           --name gitlab \
           --restart always \
           --volume /srv/gitlab/config:/etc/gitlab:Z \
           --volume /srv/gitlab/logs:/var/log/gitlab:Z \
           --volume /srv/gitlab/data:/var/opt/gitlab:Z \
           gitlab/gitlab-ce:latest
```

Ждем, когда все настроится (поглядываем в `docker logs -f gitlab`) и после
входим в контейнер (`docker exec -it gitlab bash`) для доп. настройки:

```
nano /etc/gitlab/gitlab.rb # or vim

# /etc/gitlab/gitlab.rb
external_url 'http://gitlab.nondv.home'
gitlab_rails['gitlab_shell_ssh_port'] = 3022
# /etc/gitlab/gitlab.rb

gitlab-ctl reconfigure
```

Для надежности можно перезапустить контейнер (`docker container restart
gitlab`).

#### CI

Gitlab CI уже интегрирован, но ему нужен Gitlab Runner
([документация](https://docs.gitlab.com/runner/install/docker.html#general-gitlab-runner-docker-image-usage)).

Для этого я написал небольшой скрипт:

```bash
NAME="gitlab-runner$1"
echo $NAME
docker run -d --name $NAME --restart always \
           --network=host \
           -v /srv/gitlab-runner/config:/etc/gitlab-runner \
           -v /var/run/docker.sock:/var/run/docker.sock \
           gitlab/gitlab-runner:alpine
```

После создания раннера, нам нужно его зарегистирировать. Для этого заходим в
гитлаб (через браузер), идем в Admin area -> Overview -> Runners. Там описана
установка раннеров. Вкратце, вы просто выполняете:

```bash
docker exec -it gitlab-runner register
```

и отвечаете на вопросы.


## Ваши собственные HTTP-сервисы

Они запускаются по аналогии с гитлабом. Публикуете их на каком-нибудь порте и
добавляете конфиг в nginx.


## Заключение

Теперь вы можете хостить ваши проекты на домашнем сервере и использовать мощь
Gitlab CI для автоматизации сборки и публикации ваших проектов. Удобно ведь
делать `git push` и не беспокоиться о запуске, верно?

Еще я бы рекомендовал настроить почту для гитлаба. Лично я использовал
почтовый ящик на Яндексе.
[Документация](https://docs.gitlab.com/omnibus/settings/smtp.html).
