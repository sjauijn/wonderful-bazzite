# Steam - Включение/Отключение автоматических и ручных обновлений

#### [[English]](https://github.com/sjauijn/wonderful-bazzite/blob/main/low%20iq%20stuff/Disable-Steam-Update/Disable-Steam-Update.md) [[Русский]](https://github.com/sjauijn/wonderful-bazzite/blob/main/low%20iq%20stuff/Disable-Steam-Update/RU-Disable-Steam-Update.md)

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
