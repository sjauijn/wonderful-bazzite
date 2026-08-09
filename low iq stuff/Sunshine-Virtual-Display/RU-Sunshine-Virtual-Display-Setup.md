# Sunshine - Настройка виртуального дисплея

### Шаг 1. Настройте физический монитор как основной вывод

Настройте физический монитор как основной вывод для `Gamescope`, как описано в официальной документации `Bazzite`.

`⚠️Настройка не требуется, если подключен один монитор, и больше не используется других виртуальных видео-выводов.`

### Шаг 2. Найдите путь физического монитора для DRM-переключения

Чтобы включать/выключать физический монитор через команды, сначала нужно найти правильный путь для физического монитора. Выполните команду:

```bash
kscreen-doctor -o | grep Output:
```

В выводе найдите строку такого вида, это и есть ваш монитор:

```bash
Output: 2 DP-2 4a12b364-330b-4083-a5b3-ce74e7155c15
```

Теперь определим все имеющиеся порты вывода видеокарты:

```bash
sudo find /sys/kernel/debug/dri -type f -name force -print
```

Примерный вывод:

```bash
/sys/kernel/debug/dri/0000:28:00.0/Writeback-1/force
/sys/kernel/debug/dri/0000:28:00.0/HDMI-A-1/force
/sys/kernel/debug/dri/0000:28:00.0/DP-2/force
/sys/kernel/debug/dri/0000:28:00.0/DP-1/force
```

В данном случае физическим монитором занят порт `DP-2`, и доступно два порта на выбор для виртуального монитора - `DP-1` и `HDMI-A-1`.

Мы выберем порт `DP-1` для виртуального монитора.

Создайте скрипт `virtual-display-and-switch.sh`:

```bash
cat > ~/.local/bin/virtual-display-and-switch.sh << 'ENDSCRIPT'
#!/usr/bin/env bash
set -e
USER_NAME=$(whoami)
BIN_DIR="$HOME/.local/bin"
FW_DIR="/usr/local/lib/firmware"
EDID_NAME="edid.bin"
ROOT_HELPER="/usr/local/sbin/drm-force.sh"
echo "== Установщик скриптов для виртуального дисплея в Bazzite OS =="
echo
echo "Доступные файлы force:"
echo
sudo find /sys/kernel/debug/dri -type f -name force -print 2>/dev/null || true
echo
echo "Все подключённые мониторы:"
echo
kscreen-doctor -o
echo
read_force_path() {
    local PROMPT="$1"
    local PATH_VALUE
    local CONNECTOR
    while true; do
        read -r -p "$PROMPT" PATH_VALUE
        if [[ -z "$PATH_VALUE" ]]; then
            echo "Путь не может быть пустым."
            continue
        fi
        if [[ "$PATH_VALUE" != /sys/kernel/debug/dri/*/force ]]; then
            echo
            echo "Некорректный путь."
            echo "Ожидаемый формат:"
            echo "/sys/kernel/debug/dri/0000:28:00.0/DP-1/force"
            echo
            continue
        fi
        if ! sudo test -f "$PATH_VALUE"; then
            echo
            echo "Файл не найден:"
            echo "$PATH_VALUE"
            echo "Проверьте путь и попробуйте ещё раз."
            echo
            continue
        fi
        CONNECTOR=$(basename "$(dirname "$PATH_VALUE")")
        case "$CONNECTOR" in
            DP-*|HDMI-A-*)
                ;;
            *)
                echo
                echo "Указан неподдерживаемый тип выхода: $CONNECTOR"
                echo "Выберите выход DP-* или HDMI-A-*."
                echo
                continue
                ;;
        esac
        REPLY="$PATH_VALUE"
        return 0
    done
}
echo "Сначала укажите force-файл физического монитора."
echo "Он будет использоваться для его отключения во время стриминга."
echo
read_force_path \
    "Введите полный путь к force физического монитора: "
FORCE_PATH="$REPLY"
echo
echo "Теперь укажите force-файл свободного порта."
echo "Он будет использоваться для виртуального монитора."
echo
read_force_path \
    "Введите полный путь к force свободного порта для виртуального монитора: "
VIRTUAL_PATH="$REPLY"
if [[ "$FORCE_PATH" == "$VIRTUAL_PATH" ]]; then
    echo
    echo "Ошибка: физический и виртуальный порты совпадают."
    echo "Нужно выбрать два разных force-файла."
    exit 1
fi
VIRTUAL_CONNECTOR=$(basename "$(dirname "$VIRTUAL_PATH")")
echo
echo "Будет использоваться:"
echo
echo "Физический монитор:"
echo "  $FORCE_PATH"
echo
echo "Виртуальный монитор:"
echo "  $VIRTUAL_PATH"
echo "  Коннектор: $VIRTUAL_CONNECTOR"
echo
mkdir -p "$BIN_DIR"
sudo mkdir -p "$FW_DIR"
sudo mkdir -p /usr/local/sbin
FORCE_PATH_ESCAPED=$(printf '%q' "$FORCE_PATH")
echo "Installing root DRM force helper..."
sudo tee "$ROOT_HELPER" >/dev/null <<EOF
#!/bin/bash
set -e
ACTION="\$1"
FORCE_PATH=$FORCE_PATH_ESCAPED
if [[ "\$ACTION" != "on" && "\$ACTION" != "off" ]]; then
    echo "Usage: drm-force.sh on|off"
    exit 1
fi
if [[ ! -f "\$FORCE_PATH" ]]; then
    echo "DRM force-файл не найден:"
    echo "\$FORCE_PATH"
    exit 1
fi
echo "\$ACTION" > "\$FORCE_PATH"
udevadm trigger --subsystem-match=drm
EOF
sudo chown root:root "$ROOT_HELPER"
sudo chmod 700 "$ROOT_HELPER"
echo "Installing user switch scripts..."
cat > "$BIN_DIR/switch-to-streaming.sh" <<'EOF'
#!/bin/bash
set -e
sudo /usr/local/sbin/drm-force.sh off
EOF
cat > "$BIN_DIR/switch-to-local.sh" <<'EOF'
#!/bin/bash
set -e
sudo /usr/local/sbin/drm-force.sh on
EOF
chmod +x "$BIN_DIR/switch-to-streaming.sh"
chmod +x "$BIN_DIR/switch-to-local.sh"
echo "Configuring sudoers..."
sudo tee /etc/sudoers.d/sunshine-drm-switch >/dev/null <<EOF
$USER_NAME ALL=(root) NOPASSWD: $ROOT_HELPER
EOF
sudo chmod 440 /etc/sudoers.d/sunshine-drm-switch
echo
echo "✓ DRM auto-switch установлен"
echo
echo "Используемый force-файл физического монитора:"
echo "  $FORCE_PATH"
echo
echo "Используемый порт виртуального монитора:"
echo "  $VIRTUAL_PATH"
echo
echo "Скрипты находятся в:"
echo "  $BIN_DIR/switch-to-streaming.sh"
echo "  $BIN_DIR/switch-to-local.sh"
echo
echo "Теперь с помощью программы CRU создайте файл edid.bin и перенесите его по пути:"
echo "  $FW_DIR/$EDID_NAME"
echo
echo "После копирования edid.bin добавьте kernel args:"
echo
echo "  sudo rpm-ostree kargs --append-if-missing=\"firmware_class.path=$FW_DIR drm.edid_firmware=${VIRTUAL_CONNECTOR}:$EDID_NAME video=${VIRTUAL_CONNECTOR}:e\""
echo
echo "После добавления kernel args потребуется перезагрузка."
ENDSCRIPT
```

```bash
chmod +x ~/.local/bin/virtual-display-and-switch.sh
```

Этот установочный скрипт создаёт скрипты `switch-to-streaming.sh` и `switch-to-local.sh`, используемые для отключения/включения физического монитора во время стриминга, а также скрипт `drm-force.sh`, который использует `sudoers`, чтобы позволить двум скриптам переключения запускаться от root.

Для получения файла `edid.bin` с кастомными разрешениями нужно использовать программу для Windows под названием `Custom Resolution Utility` `(CRU)`. В `Bazzite OS` её можно запустить с помошью `Proton`.

Находясь в программе, нажмите `Add...` в разделе **Detailed resolutions**.

<p>
  <img src="https://raw.githubusercontent.com/sjauijn/wonderful-bazzite/refs/heads/main/low%20iq%20stuff/Sunshine-Virtual-Display/1.jpg" alt="1">
</p>

В открывшемся меню в поле `Active` укажите соотношение сторон, а в поле `Refresh rate` задайте частоту обновления, которая будет использоваться виртуальным монитором, затем нажмите `OK`.

<p>
  <img src="https://raw.githubusercontent.com/sjauijn/wonderful-bazzite/refs/heads/main/low%20iq%20stuff/Sunshine-Virtual-Display/2.jpg" alt="2">
</p>

Создайте нужное количество кастомных разрешений экрана , которые будут использоваться виртуальным монитором и нажмите `Export`. Файл необходимо сохранить с названием `edid.bin`.

<p>
  <img src="https://raw.githubusercontent.com/sjauijn/wonderful-bazzite/refs/heads/main/low%20iq%20stuff/Sunshine-Virtual-Display/3.jpg" alt="3">
</p>

Полученный файл `edid.bin` перенести в директорию `/usr/local/lib/firmware/`.

Теперь запустите скрипт virtual-display-and-switch.sh и следуйте его инструкциям :

```bash
~/.local/bin/virtual-display-and-switch.sh
```

После выполнения скрипта **не забудьте** выполнить команду kernel args из вывода скрипта, а после, перезагрузить ПК.

### Шаг 3. Настройка modes.cfg для работы виртуального монитора

В `Sunshine` создайте приложение с названием `Virtual Display`, замените `NAME_USER` на ваше `имя пользователя` и перенесите команды в раздел `Команды подготовки`:

**Выполнить команду**:

```bash
/home/NAME_USER/.local/bin/switch-to-streaming.sh
```

**Команда закрытия**:

```bash
/home/NAME_USER/.local/bin/switch-to-local.sh
```

Установите приложение `Moonlight` на устройство, на которое вы планируете выводить изображение. В приложении перейдите в настройки и в разделе `Разрешение и FPS` укажите соотношение сторон и частоту обновления дисплея вашего `Moonlight` устройства.

Соедините ваш хост-ПК и `Moonlight` устройство с помощью Pin кода.

Теперь перейдите в режим `Gamescope`. Находясь в нём, подключите своё `Moonlight` устройство через приложение `Virtual Display` к ПК. Физический монитор должен погаснуть, и появиться вывод на `Moonlight` устройстве, при этом автоматически разрешение экрана устройства не применилось. Это нормально.

Зайдите в настройки экрана, отключите пункт `Автоматически задать разрешение экрана`. Отключитесь от хоста ПК выходом из сессии.

Таким образом виртуальный монитор определился в `modes.cfg`.

Далее возвращайтесь в `Desktop Mode`. Нужно узнать имя виртуального дисплея в файле `modes.cfg`:

```bash
nano ~/.config/gamescope/modes.cfg
```

Если виртуальный дисплей под пустым именем, например:

```bash
@@@ :2400x1080@120
```

то нужно удалить эту строку, сохранить файл, перезагрузить ПК и снова в `Gamescope` запустить `Sunshine` сессию через приложение `Virtual Display`, зайти в настройки экрана, отключить пункт `Автоматически задать разрешение экрана` и после в `Desktop Mode` снова проверить файл `modes.cfg`.

Вот так может выглядеть нужная строка:

```bash
Microsoft :2400x1080@90
```

В данном случае название монитора - `"Microsoft "` (с пробелом, это важно).

Создайте скрипт `auto-resolution.sh` :

```bash
cat > ~/.local/bin/auto-resolution.sh << 'ENDSCRIPT'
#!/usr/bin/env bash
MODES_CFG="$HOME/.config/gamescope/modes.cfg"
MODES_LIST="$HOME/.local/bin/virtual-modes.txt"
DISPLAY_NAME="Microsoft "
VIRTUAL_CONNECTOR="DP-1"
REQ_W=${SUNSHINE_CLIENT_WIDTH:-0}
REQ_H=${SUNSHINE_CLIENT_HEIGHT:-0}
REQ_R=${SUNSHINE_CLIENT_FPS:-60}
if [ "$REQ_W" -eq 0 ] || [ "$REQ_H" -eq 0 ]; then
    exit 0
fi
best_mode=""
best_score=999999999
while read -r mode; do
    [[ "$mode" =~ ^[0-9]+x[0-9]+@[0-9]+$ ]] || continue
    w=${mode%x*}
    h=${mode#*x}; h=${h%@*}
    r=${mode#*@}
    dw=$((w-REQ_W))
    dh=$((h-REQ_H))
    dr=$((r-REQ_R))
    score=$(( dw*dw + dh*dh + dr*dr*10 )) || true
    if (( score < best_score )); then
        best_score=$score
        best_mode="$mode"
    fi
done < "$MODES_LIST"
[ -z "$best_mode" ] && exit 0
W=${best_mode%x*}
H=${best_mode#*x}; H=${H%@*}
R=${best_mode#*@}
echo "[auto-res] Selected ${DISPLAY_NAME}mode: ${W}x${H}@${R}"
if [ ! -f "${MODES_CFG}.bak" ]; then
    cp "$MODES_CFG" "${MODES_CFG}.bak" || true
fi
ESCAPED_DISPLAY_NAME=$(printf '%s\n' "$DISPLAY_NAME" | sed 's/[][\.^$*+?{}|()]/\\&/g')
sed -i \
    -E "s|^${ESCAPED_DISPLAY_NAME}:.*|${DISPLAY_NAME}:${W}x${H}@${R}|" \
    "$MODES_CFG" || true
if command -v flatpak-spawn >/dev/null 2>&1; then
    flatpak-spawn --host kscreen-doctor "output.${VIRTUAL_CONNECTOR}.mode.${W}x${H}@${R}" || true
else
    kscreen-doctor "output.${VIRTUAL_CONNECTOR}.mode.${W}x${H}@${R}" || true
fi
exit 0
ENDSCRIPT
```

```bash
chmod +x ~/.local/bin/auto-resolution.sh
```

Полученное название виртуального дисплея из `modes.cfg` добавить в скрипт `auto-resolution.sh` в строку:

```bash
DISPLAY_NAME="Microsoft "
```

(учитывая пробел в названии, если он есть) и сохранить скрипт.

Также в следующей строке измените тип порта виртуального монитора:

```bash
VIRTUAL_CONNECTOR="DP-1"
```

Найти его можно с помощью команды:

```bash
kscreen-doctor -o | grep Output:
```

Вывод будет примерно такой. Здесь `DP-2` физический монитор, а `DP-1` виртуальный:

```bash
Output: 1 DP-1 2609a03c-80b6-4061-b81e-4075ce94764e
Output: 2 DP-2 4a12b364-330b-4083-a5b3-ce74e7155c15
```

### Шаг 4. Создать файл virtual-modes.txt

Здесь перечисляются доступные к выбору разрешения экрана для виртуального дисплея. Эти разрешения также должды присутствовать в созданном `edid.bin`.

```bash
cat > ~/.local/bin/virtual-modes.txt << 'ENDSCRIPT'
2560x1440@60
2400x1080@120
2400x1080@90
1920x1080@60
ENDSCRIPT
```

### Шаг 5. Добавить скрипты в список Sunshine do/undo

Замените `NAME_USER` на ваше `имя пользователя` и перенесите команды в `Sunshine` в приложение `Virtual Display`:

**Выполнить команду**:

```bash
/home/NAME_USER/.local/bin/auto-resolution.sh
```

```bash
/home/NAME_USER/.local/bin/switch-to-streaming.sh
```

**Команда закрытия**:

```bash
/home/NAME_USER/.local/bin/switch-to-local.sh
```

### Финал.

Теперь в `Desktop Mode` и `Gamescope` при подключении к `Sunshine` через приложение `Virtual Display` к ПК физический монитор должен погаснуть, и появиться вывод на `Moonlight` устройстве с автоматически применённом разрешением экрана `Moonlight` устройства.

При выходе из `Sunshine` сессии рабочим выводом вновь становится физический монитор.

### Удаление.

Если вы хотите откатить все изменения, примененные данным туториалом, создайте и выполните следующий скрипт:

```bash
cat > ~/.local/bin/delete-virtual-display.sh << 'ENDSCRIPT'
#!/usr/bin/env bash
set -e
echo "== Удаление виртуального дисплея Sunshine =="
echo
# --- Восстановление modes.cfg из бекапа ---
MODES_CFG="$HOME/.config/gamescope/modes.cfg"
if [ -f "${MODES_CFG}.bak" ]; then
    cp "${MODES_CFG}.bak" "$MODES_CFG"
    rm -f "${MODES_CFG}.bak"
    echo "✓ Восстановлен $MODES_CFG из резервной копии"
else
    echo "- Резервная копия modes.cfg не найдена"
fi
# --- Удаление sudoers ---
if [ -f /etc/sudoers.d/sunshine-drm-switch ]; then
    sudo rm -f /etc/sudoers.d/sunshine-drm-switch
    echo "✓ Удалён /etc/sudoers.d/sunshine-drm-switch"
else
    echo "- /etc/sudoers.d/sunshine-drm-switch файл не найден"
fi
# --- Удаление root-хелпера ---
if [ -f /usr/local/sbin/drm-force.sh ]; then
    sudo rm -f /usr/local/sbin/drm-force.sh
    echo "✓ Удалён /usr/local/sbin/drm-force.sh"
else
    echo "- /usr/local/sbin/drm-force.sh файл не найден"
fi
# --- Удаление firmware edid.bin ---
if [ -f /usr/local/lib/firmware/edid.bin ]; then
    sudo rm -f /usr/local/lib/firmware/edid.bin
    echo "✓ Удалён /usr/local/lib/firmware/edid.bin"
else
    echo "- /usr/local/lib/firmware/edid.bin файл не найден"
fi
# --- Удаление пользовательских скриптов ---
USER_SCRIPTS=(
    "$HOME/.local/bin/switch-to-streaming.sh"
    "$HOME/.local/bin/switch-to-local.sh"
    "$HOME/.local/bin/auto-resolution.sh"
    "$HOME/.local/bin/virtual-modes.txt"
    "$HOME/.local/bin/virtual-display-and-switch.sh"
)
for f in "${USER_SCRIPTS[@]}"; do
    if [ -f "$f" ]; then
        rm -f "$f"
        echo "✓ Удалён $f"
    else
        echo "- $f файл не найден"
    fi
done
# --- Удаление kernel args ---
echo
echo "================================================================"
echo "⚠️  Kernel args необходимо удалить вручную."
echo "    Вот текущие применённые аргументы:"
echo
rpm-ostree kargs
echo
echo "    Найдите аргументы вида:"
echo "    firmware_class.path=... , drm.edid_firmware=... , video=..."
echo
while true; do
    echo "    Введите через пробел аргументы для удаления и нажмите Enter"
    echo "    (или оставьте пустым и нажмите Enter, чтобы пропустить):"
    echo
    read -r -p "> " KARGS_INPUT
    if [ -z "$KARGS_INPUT" ]; then
        echo "    Пропускаем удаление kernel args."
        break
    fi
    CURRENT_KARGS=$(rpm-ostree kargs)
    ALL_VALID=true
    for KARG in $KARGS_INPUT; do
        if ! echo "$CURRENT_KARGS" | grep -qF "$KARG"; then
            ALL_VALID=false
            break
        fi
    done
    if [ "$ALL_VALID" = false ]; then
        echo
        echo "    Присутствует неверный аргумент, повторите ввод"
        echo
        continue
    fi
    DELETE_ARGS=""
    for KARG in $KARGS_INPUT; do
        DELETE_ARGS="$DELETE_ARGS --delete=$KARG"
    done
    if sudo rpm-ostree kargs $DELETE_ARGS; then
        echo "  ✓ Удалены: $KARGS_INPUT"
    else
        echo "  ✗ Не удалось удалить аргументы"
    fi
    break
done
echo "================================================================"
echo
# --- Самоудаление ---
SELF="$HOME/.local/bin/delete-virtual-display.sh"
echo "Удаляю скрипт удаления: $SELF"
rm -f "$SELF"
echo
echo "✓ Удаление завершено."
echo
echo "  После перезагрузки аргументы будут удалены."
echo
echo "  Для применения всех изменений выполните перезагрузку:"
echo "  systemctl reboot"
ENDSCRIPT
```

```bash
chmod +x ~/.local/bin/delete-virtual-display.sh
```

Запустите скрипт и следуйте его инструкциям:

```bash
~/.local/bin/delete-virtual-display.sh
```

После выполнения скрипта перезагрузите систему:

```bash
systemctl reboot
```
