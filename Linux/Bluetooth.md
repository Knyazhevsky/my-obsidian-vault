1. Открыть файл:
```bash
sudo nano /etc/bluetooth/main.conf
```

2. Изменить параметры:
```
[General]
FastConnectable=true
ReconnectAttempts=7
ReconnectIntervals=1,1,2,3,5
IdleTimeout=0
ControllerMode=bredr
```

3. Перезапусти службу Bluetooth:
```bash
sudo systemctl restart bluetooth
```