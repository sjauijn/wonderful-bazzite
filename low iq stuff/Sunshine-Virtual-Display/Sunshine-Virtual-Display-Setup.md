# Sunshine - Virtual Display Setup

# [🇺🇸English](https://github.com/sjauijn/wonderful-bazzite/blob/main/low%20iq%20stuff/Sunshine-Virtual-Display/Sunshine-Virtual-Display-Setup.md) [🇷🇺Русский](https://github.com/sjauijn/wonderful-bazzite/blob/main/low%20iq%20stuff/Sunshine-Virtual-Display/RU-Sunshine-Virtual-Display-Setup.md)

### Step 1. Set the physical monitor as the primary output

Set the physical monitor as the primary output for `Gamescope`, as described in the official `Bazzite` documentation.

`⚠️This setup is not required if only one monitor is connected, and no other virtual video outputs are being used.`

### Step 2. Find the physical monitor's path for DRM switching

To turn the physical monitor on/off via commands, you first need to find the correct path for the physical monitor. Run the command:

```bash
kscreen-doctor -o | grep Output:
```

In the output, find a line like this one — this is your monitor:

```bash
Output: 2 DP-2 4a12b364-330b-4083-a5b3-ce74e7155c15
```

Now let's determine all available video output ports on the GPU:

```bash
sudo find /sys/kernel/debug/dri -type f -name force -print
```

Example output:

```bash
/sys/kernel/debug/dri/0000:28:00.0/Writeback-1/force
/sys/kernel/debug/dri/0000:28:00.0/HDMI-A-1/force
/sys/kernel/debug/dri/0000:28:00.0/DP-2/force
/sys/kernel/debug/dri/0000:28:00.0/DP-1/force
```

In this case, port `DP-2` is occupied by the physical monitor, and there are two ports available to choose from for the virtual monitor - `DP-1` and `HDMI-A-1`.

We'll choose port `DP-1` for the virtual monitor.

Create the script `virtual-display-and-switch.sh`:

```bash
cat > ~/.local/bin/virtual-display-and-switch.sh << 'ENDSCRIPT'
#!/usr/bin/env bash
set -e
USER_NAME=$(whoami)
BIN_DIR="$HOME/.local/bin"
FW_DIR="/usr/local/lib/firmware"
EDID_NAME="edid.bin"
ROOT_HELPER="/usr/local/sbin/drm-force.sh"
echo "== Virtual display script installer for Bazzite OS =="
echo
echo "Available force files:"
echo
sudo find /sys/kernel/debug/dri -type f -name force -print 2>/dev/null || true
echo
echo "All connected monitors:"
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
            echo "Path cannot be empty."
            continue
        fi
        if [[ "$PATH_VALUE" != /sys/kernel/debug/dri/*/force ]]; then
            echo
            echo "Invalid path."
            echo "Expected format:"
            echo "/sys/kernel/debug/dri/0000:28:00.0/DP-1/force"
            echo
            continue
        fi
        if ! sudo test -f "$PATH_VALUE"; then
            echo
            echo "File not found:"
            echo "$PATH_VALUE"
            echo "Check the path and try again."
            echo
            continue
        fi
        CONNECTOR=$(basename "$(dirname "$PATH_VALUE")")
        case "$CONNECTOR" in
            DP-*|HDMI-A-*)
                ;;
            *)
                echo
                echo "Unsupported output type specified: $CONNECTOR"
                echo "Choose a DP-* or HDMI-A-* output."
                echo
                continue
                ;;
        esac
        REPLY="$PATH_VALUE"
        return 0
    done
}
echo "First, specify the force file of the physical monitor."
echo "It will be used to turn it off during streaming."
echo
read_force_path \
    "Enter the full path to the physical monitor's force file: "
FORCE_PATH="$REPLY"
echo
echo "Now specify the force file of the free port."
echo "It will be used for the virtual monitor."
echo
read_force_path \
    "Enter the full path to the free port's force file for the virtual monitor: "
VIRTUAL_PATH="$REPLY"
if [[ "$FORCE_PATH" == "$VIRTUAL_PATH" ]]; then
    echo
    echo "Error: the physical and virtual ports are the same."
    echo "You need to choose two different force files."
    exit 1
fi
VIRTUAL_CONNECTOR=$(basename "$(dirname "$VIRTUAL_PATH")")
echo
echo "The following will be used:"
echo
echo "Physical monitor:"
echo "  $FORCE_PATH"
echo
echo "Virtual monitor:"
echo "  $VIRTUAL_PATH"
echo "  Connector: $VIRTUAL_CONNECTOR"
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
    echo "DRM force file not found:"
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
echo "✓ DRM auto-switch installed"
echo
echo "Physical monitor force file in use:"
echo "  $FORCE_PATH"
echo
echo "Virtual monitor port in use:"
echo "  $VIRTUAL_PATH"
echo
echo "The scripts are located at:"
echo "  $BIN_DIR/switch-to-streaming.sh"
echo "  $BIN_DIR/switch-to-local.sh"
echo
echo "Now use the CRU program to create the edid.bin file and transfer it to the path:"
echo "  $FW_DIR/$EDID_NAME"
echo
echo "After copying edid.bin, add the kernel args:"
echo
echo "  sudo rpm-ostree kargs --append-if-missing=\"firmware_class.path=$FW_DIR drm.edid_firmware=${VIRTUAL_CONNECTOR}:$EDID_NAME video=${VIRTUAL_CONNECTOR}:e\""
echo
echo "After adding the kernel args, a reboot will be required."
ENDSCRIPT
```

```bash
chmod +x ~/.local/bin/virtual-display-and-switch.sh
```

This installer script creates the scripts `switch-to-streaming.sh` and `switch-to-local.sh`, used to turn the physical monitor off/on during streaming, as well as the `drm-force.sh` script, which uses `sudoers` to allow the two switch scripts to run as root.

To get the `edid.bin` file with custom resolutions, you need to use a Windows program called `Custom Resolution Utility` `(CRU)`. On `Bazzite OS` it can be run with `Proton`.

While in the program, click `Add...` in the **Detailed resolutions** section.

<p>
  <img src="https://raw.githubusercontent.com/sjauijn/wonderful-bazzite/refs/heads/main/low%20iq%20stuff/Sunshine-Virtual-Display/1.jpg" alt="1">
</p>

In the menu that opens, specify the aspect ratio in the `Active` field, and set the refresh rate to be used by the virtual monitor in the `Refresh rate` field, then click `OK`.

<p>
  <img src="https://raw.githubusercontent.com/sjauijn/wonderful-bazzite/refs/heads/main/low%20iq%20stuff/Sunshine-Virtual-Display/2.jpg" alt="2">
</p>

Create the required number of custom screen resolutions to be used by the virtual monitor and click `Export`. The file must be saved with the name `edid.bin`.

<p>
  <img src="https://raw.githubusercontent.com/sjauijn/wonderful-bazzite/refs/heads/main/low%20iq%20stuff/Sunshine-Virtual-Display/3.jpg" alt="3">
</p>

Move the resulting `edid.bin` file to the directory `/usr/local/lib/firmware/`.

Now run the script virtual-display-and-switch.sh and follow its instructions:

```bash
~/.local/bin/virtual-display-and-switch.sh
```

After the script finishes, **don't forget** to run the kernel args command from the script's output, and after that, reboot the PC.

### Step 3. Configuring modes.cfg for the virtual monitor to work

In `Sunshine`, create an application named `Virtual Display`, replace `NAME_USER` with your `username`, and put the commands into the `Preparation Commands` section:

**Do Command**:

```bash
/home/NAME_USER/.local/bin/switch-to-streaming.sh
```

**Undo Command**:

```bash
/home/NAME_USER/.local/bin/switch-to-local.sh
```

Install the `Moonlight` application on the device you plan to display the image on. In the application, go to settings and in the `Resolution and FPS` section specify the aspect ratio and refresh rate of your `Moonlight` device's display.

Connect your host PC and `Moonlight` device using the Pin code.

Now switch to `Gamescope` mode. While in it, connect your `Moonlight` device to the PC via the `Virtual Display` application. The physical monitor should turn off, and output should appear on the `Moonlight` device, though the device's screen resolution will not be applied automatically. This is normal.

Go to the display settings, disable the `Automatically set screen resolution` option. Disconnect from the host PC by exiting the session.

This way the virtual monitor gets detected in `modes.cfg`.

Next, go back to `Desktop Mode`. You need to find out the virtual display's name in the `modes.cfg` file:

```bash
nano ~/.config/gamescope/modes.cfg
```

If the virtual display has an empty name, for example:

```bash
@@@ :2400x1080@120
```

then you need to delete this line, save the file, reboot the PC, and again in `Gamescope` start a `Sunshine` session via the `Virtual Display` application, go to the display settings, disable the `Automatically set screen resolution` option, and after that check the `modes.cfg` file again in `Desktop Mode`.

Here's what the needed line might look like:

```bash
Microsoft :2400x1080@90
```

In this case, the monitor name is `"Microsoft "` (with a space, this matters).

Create the script `auto-resolution.sh`:

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

Add the virtual display's name obtained from `modes.cfg` to the `auto-resolution.sh` script, in the line:

```bash
DISPLAY_NAME="Microsoft "
```

(taking into account the space in the name, if there is one) and save the script.

Also, on the following line, change the virtual monitor's port type:

```bash
VIRTUAL_CONNECTOR="DP-1"
```

You can find it using the command:

```bash
kscreen-doctor -o | grep Output:
```

The output will look something like this. Here `DP-2` is the physical monitor, and `DP-1` is the virtual one:

```bash
Output: 1 DP-1 2609a03c-80b6-4061-b81e-4075ce94764e
Output: 2 DP-2 4a12b364-330b-4083-a5b3-ce74e7155c15
```

### Step 4. Create the virtual-modes.txt file

This lists the screen resolutions available for selection for the virtual display. These resolutions must also be present in the created `edid.bin`.

```bash
cat > ~/.local/bin/virtual-modes.txt << 'ENDSCRIPT'
2560x1440@60
2400x1080@120
2400x1080@90
1920x1080@60
ENDSCRIPT
```

### Step 5. Add the scripts to the Sunshine do/undo list

Replace `NAME_USER` with your `username` and put the commands into the `Virtual Display` application in `Sunshine`:

**Do Command**:

```bash
/home/NAME_USER/.local/bin/auto-resolution.sh
```

```bash
/home/NAME_USER/.local/bin/switch-to-streaming.sh
```

**Undo Command**:

```bash
/home/NAME_USER/.local/bin/switch-to-local.sh
```

### Final.

Now in `Desktop Mode` and `Gamescope`, when connecting to the PC through `Sunshine` via the `Virtual Display` application, the physical monitor should turn off, and output should appear on the `Moonlight` device with the `Moonlight` device's screen resolution automatically applied.

When exiting the `Sunshine` session, the physical monitor once again becomes the working output.

### Removal.

If you want to roll back all changes applied by this tutorial, create and run the following script:

```bash
cat > ~/.local/bin/delete-virtual-display.sh << 'ENDSCRIPT'
#!/usr/bin/env bash
set -e
echo "== Removing Sunshine virtual display =="
echo
# --- Restoring modes.cfg from backup ---
MODES_CFG="$HOME/.config/gamescope/modes.cfg"
if [ -f "${MODES_CFG}.bak" ]; then
    cp "${MODES_CFG}.bak" "$MODES_CFG"
    rm -f "${MODES_CFG}.bak"
    echo "✓ Restored $MODES_CFG from backup"
else
    echo "- Backup copy of modes.cfg not found"
fi
# --- Removing sudoers ---
if [ -f /etc/sudoers.d/sunshine-drm-switch ]; then
    sudo rm -f /etc/sudoers.d/sunshine-drm-switch
    echo "✓ Removed /etc/sudoers.d/sunshine-drm-switch"
else
    echo "- /etc/sudoers.d/sunshine-drm-switch file not found"
fi
# --- Removing root helper ---
if [ -f /usr/local/sbin/drm-force.sh ]; then
    sudo rm -f /usr/local/sbin/drm-force.sh
    echo "✓ Removed /usr/local/sbin/drm-force.sh"
else
    echo "- /usr/local/sbin/drm-force.sh file not found"
fi
# --- Removing edid.bin firmware ---
if [ -f /usr/local/lib/firmware/edid.bin ]; then
    sudo rm -f /usr/local/lib/firmware/edid.bin
    echo "✓ Removed /usr/local/lib/firmware/edid.bin"
else
    echo "- /usr/local/lib/firmware/edid.bin file not found"
fi
# --- Removing user scripts ---
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
        echo "✓ Removed $f"
    else
        echo "- $f file not found"
    fi
done
# --- Removing kernel args ---
echo
echo "================================================================"
echo "⚠️  Kernel args must be removed manually."
echo "    Here are the currently applied arguments:"
echo
rpm-ostree kargs
echo
echo "    Find arguments like:"
echo "    firmware_class.path=... , drm.edid_firmware=... , video=..."
echo
while true; do
    echo "    Enter the arguments to remove separated by spaces and press Enter"
    echo "    (or leave it empty and press Enter to skip):"
    echo
    read -r -p "> " KARGS_INPUT
    if [ -z "$KARGS_INPUT" ]; then
        echo "    Skipping kernel args removal."
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
        echo "    An invalid argument is present, re-enter your input"
        echo
        continue
    fi
    DELETE_ARGS=""
    for KARG in $KARGS_INPUT; do
        DELETE_ARGS="$DELETE_ARGS --delete=$KARG"
    done
    if sudo rpm-ostree kargs $DELETE_ARGS; then
        echo "  ✓ Removed: $KARGS_INPUT"
    else
        echo "  ✗ Failed to remove arguments"
    fi
    break
done
echo "================================================================"
echo
# --- Self-deletion ---
SELF="$HOME/.local/bin/delete-virtual-display.sh"
echo "Removing the deletion script: $SELF"
rm -f "$SELF"
echo
echo "✓ Removal complete."
echo
echo "  After rebooting, the arguments will be removed."
echo
echo "  To apply all changes, perform a reboot:"
echo "  systemctl reboot"
ENDSCRIPT
```

```bash
chmod +x ~/.local/bin/delete-virtual-display.sh
```

Run the script and follow its instructions:

```bash
~/.local/bin/delete-virtual-display.sh
```

After the script finishes, reboot the system:

```bash
systemctl reboot
```
