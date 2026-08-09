# Steam - Включение/Отключение автоматических и ручных обновлений

### Включение обновлений клиента

Удаляем конфигурационный файл `steam.cfg`:

```bash
rm ~/.steam/steam/steam.cfg
```

### Отключение обновлений клиента

Создаём конфигурационный файл `steam.cfg` со следующим содержимым:

```bash
cat > ~/.steam/steam/steam.cfg << 'EOF'
BootStrapperInhibitAll=enable
BootStrapperForceSelfUpdate=disable
EOF
```
