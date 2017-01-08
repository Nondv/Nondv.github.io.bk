---

title:  'mpd зависает при смене песни на Raspberry Pi'
categories: administration
tags: [raspberry, mpd, linux]
---

# Проблема

Накатил на Raspberry Pi 3B плеер-демон mpd, но при смене песен
периодически демон зависал и помогала только команда

```
sudo systemctl restart mpd
```

# Решение

Изначально я грешил на NFS (на котором хранилась музыка), но
оказалось, что проблема каким-то образом связана с alsa.

В общем, достаточно было в конфиге указать типом миксера "software":

```
audio_output {
  type        "alsa"
  name        "bcm2835 ALSA"
  device      "hw:0,0"
  mixer_type  "software"
}
```

Увидел этот прием случайно в [этой](https://habrahabr.ru/post/201034/)
статье.
