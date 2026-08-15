# Bazzite - Установка пользовательского Центра Сертификации

#### [🇺🇸English](https://github.com/sjauijn/wonderful-bazzite/blob/main/low%20iq%20stuff/Installing-Root-CA/Installing-Root-CA.md) [🇷🇺Русский](https://github.com/sjauijn/wonderful-bazzite/blob/main/low%20iq%20stuff/Installing-Root-CA/RU-Installing-root-CA.md)

## Шаг 1. Скопируйте сертификат CA в хранилище доверия

```bash
sudo cp rootCA.crt /etc/pki/ca-trust/source/anchors/
```

## Шаг 2. Обновите конфигурацию доверенных CA-сертификатов

```bash
sudo update-ca-trust extract
```

## Шаг 3. Перезагрузитесь, чтобы применить изменения

```bash
sudo systemctl reboot
```

### Удаление

Если вы хотите откатить все изменения, примененные данным туториалом, выполните следующие действия:

Удалите сертификат из хранилища доверия

```bash
sudo rm /etc/pki/ca-trust/source/anchors/rootCA.crt
```

Пересоберите хранилище доверия без этого CA

```bash
sudo update-ca-trust extract
```

Выполните перезагрузку для применения изменений

```bash
sudo systemctl reboot
```
