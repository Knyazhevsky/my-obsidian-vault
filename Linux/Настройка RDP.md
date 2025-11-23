## Установка софта

Устанавливаем freerdp3
```bash
sudo apt install freerdp3-x11
```

## Скрипт для настройки для работы:
```bash
#!/bin/bash

REMOTE_SERVER='10.28.117.14'
USER_NAME='knyazhevskiy_59260'
USER_PASSWORD='my_password'
DOMAIN='hq'

xfreerdp3 /v:$REMOTE_SERVER /u:$USER_NAME /p:$USER_PASSWORD /d:$DOMAIN \
/multimon \
/sound:sys:alsa /microphone:sys:alsa \
/cert:ignore \
/f \
+clipboard
```