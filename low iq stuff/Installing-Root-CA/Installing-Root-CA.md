# Bazzite — Installing a Custom Certificate Authority (CA)

#### [🇺🇸English](https://github.com/sjauijn/wonderful-bazzite/blob/main/low%20iq%20stuff/Installing-Root-CA/Installing-Root-CA.md) [🇷🇺Русский](https://github.com/sjauijn/wonderful-bazzite/blob/main/low%20iq%20stuff/Installing-Root-CA/RU-Installing-root-CA.md)

## Step 1. Copy the CA certificate to the trust store

```bash
sudo cp rootCA.crt /etc/pki/ca-trust/source/anchors/
```

## Step 2. Update the trusted CA certificates configuration

```bash
sudo update-ca-trust extract
```

## Step 3. Reboot to apply the changes

```bash
sudo systemctl reboot
```

### Removal.

If you want to roll back all changes applied by this tutorial, create and run the following script:

Remove the certificate from the trust store

```bash
sudo rm /etc/pki/ca-trust/source/anchors/rootCA.crt
```

Rebuild the trust store without this CA

```bash
sudo update-ca-trust extract
```

Reboot to fully apply the changes

```bash
sudo systemctl reboot
```
