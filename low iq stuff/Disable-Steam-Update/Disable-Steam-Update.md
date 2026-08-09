# Steam - Enabling/Disabling automatic and manual updates

#### [🇺🇸English](https://github.com/sjauijn/wonderful-bazzite/blob/main/low%20iq%20stuff/Disable-Steam-Update/Disable-Steam-Update.md) [🇷🇺Русский](https://github.com/sjauijn/wonderful-bazzite/blob/main/low%20iq%20stuff/Disable-Steam-Update/RU-Disable-Steam-Update.md)

### Enabling client updates

Deleting the configuration file `steam.cfg`:

```bash
rm ~/.steam/steam/steam.cfg
```

### Disabling client updates

Creating the configuration file `steam.cfg` with the following contents:

```bash
cat > ~/.steam/steam/steam.cfg << 'EOF'
BootStrapperInhibitAll=enable
BootStrapperForceSelfUpdate=disable
EOF
```
