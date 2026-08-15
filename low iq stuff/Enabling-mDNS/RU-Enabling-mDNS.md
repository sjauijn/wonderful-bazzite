# Bazzite — Включение mDNS для доступа к локальным сервисам `(.local)`

```bash
resolvectl status
```

mDNS (Multicast DNS) позволяет разрешать имена хостов в локальной сети без центрального DNS-сервера. Это даёт возможность открывать локальные сервисы по именам вида `myserver.local:8080` или `nas.local`, а не только по IP-адресу. В Bazzite поддержка mDNS по умолчанию отключена, поэтому без ручной настройки такие имена не будут работать ни в браузере, ни в других приложениях.

### Шаг 1. Проверить текущее состояние DNS/mDNS

```bash
resolvectl status
```

### Шаг 2. Создать каталог для drop-in конфига systemd-resolved

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
```

### Шаг 3. Включить mDNS в systemd-resolved

```bash
sudo tee /etc/systemd/resolved.conf.d/10-mdns.conf << EOF
[Resolve]
MulticastDNS=yes
EOF
```

### Шаг 4. Создать каталог для конфига NetworkManager

```bash
sudo mkdir -p /etc/NetworkManager/conf.d
```

### Шаг 5. Включить mDNS на уровне сетевого соединения

```bash
sudo tee /etc/NetworkManager/conf.d/mdns.conf << EOF
[connection]
connection.mdns=yes
EOF
```

### Шаг 6. Перезапустить службы

```bash
sudo systemctl restart systemd-resolved NetworkManager
```

### Шаг 7. Проверить, что mDNS включён

```bash
resolvectl status
```

### Шаг 8. Протестировать работу *.local

```bash
getent hosts <имя_хоста>.local
```

или открыть в браузере:

```bash
http://<имя_хоста>.local:порт
```
