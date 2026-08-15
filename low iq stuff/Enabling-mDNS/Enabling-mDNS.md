# Bazzite — Enabling mDNS for Access to Local Services (*.local)

```bash
resolvectl status
```

mDNS (Multicast DNS) allows resolving hostnames on a local network without a central DNS server. This makes it possible to access local services by names like `myserver.local:8080` or `nas.local`, instead of using IP addresses only. In Bazzite, mDNS support is disabled by default, so without manual configuration such names will not work in browsers or other applications.

## Step 1. Check the current DNS/mDNS status

```bash
resolvectl status
```

## Step 2. Create a drop-in directory for systemd-resolved configuration

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
```

## Step 3. Enable mDNS in systemd-resolved

```bash
sudo tee /etc/systemd/resolved.conf.d/10-mdns.conf << EOF
[Resolve]
MulticastDNS=yes
EOF
```

## Step 4. Create a directory for NetworkManager configuration

```bash
sudo mkdir -p /etc/NetworkManager/conf.d
```

## Step 5. Enable mDNS at the connection level

```bash
sudo tee /etc/NetworkManager/conf.d/mdns.conf << EOF
[connection]
connection.mdns=yes
EOF
```

## Step 6. Restart the services

```bash
sudo systemctl restart systemd-resolved NetworkManager
```

## Step 7. Verify that mDNS is enabled

```bash
resolvectl status
```

## Step 8. Test *.local resolution

```bash
getent hosts <hostname>.local
```

or open in a browser:

```text
http://<hostname>.local:port
```
