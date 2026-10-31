# Установка socat
```
sudo apt install socat
```
```
nano /etc/systemd/system/socat-udp-forward.service
```
```
[Unit]
Description=UDP forward with socat
After=network.target

[Service]
ExecStart=/usr/bin/socat UDP4-LISTEN:51830,fork UDP:<SERVER>:51830
Restart=always
User=nobody

[Install]
WantedBy=multi-user.target
```
```
sudo systemctl enable --now socat-udp-forward.service
```


# Установка wstunnel
```
cd ~ && curl -fLo wstunnel.tar.gz https://github.com/erebe/wstunnel/releases/download/v10.1.9/wstunnel_10.1.9_linux_amd64.tar.gz
```
```
tar -xzf wstunnel.tar.gz wstunnel
```
```
chmod +x wstunnel
```
```
sudo mv wstunnel /usr/local/bin
```
```
wstunnel --version
```

wstunnel server
```
read -r -n 64 path_prefix < <(LC_ALL=C tr -dc "[:alnum:]" < /dev/urandom); echo $path_prefix
```
```
sudo nano /etc/systemd/system/wstunnel.service
```
```
[Unit]
Description=Wstunnel for WireGuard
After=network-online.target
Wants=network-online.target

[Service]
User=root
Type=exec
ExecStart=/usr/local/bin/wstunnel server --restrict-http-upgrade-path-prefix "<secret>" --restrict-to localhost:51830 wss://0.0.0.0:443
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

wstunnel client
```
[Unit]
Description=Wstunnel for WireGuard
After=network-online.target
Wants=network-online.target

[Service]
User=root
Type=exec
ExecStart=/usr/local/bin/wstunnel client --http-upgrade-path-prefix "<secret>" -L "udp://0.0.0.0:51830:localhost:51830?timeout_sec=0" wss://<server>:443
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl start wstunnel && sudo systemctl enable wstunnel
```


# Настройка Wireguard

```
apt update && apt upgrade -y
```
```
apt install -y wireguard
```
```
wg genkey | tee /etc/wireguard/privatekey | wg pubkey | tee /etc/wireguard/publickey
```
```
chmod 600 /etc/wireguard/privatekey
```
```
ip a
```

### Полученный интефейс (eth0, ens3) инспользуем в /etc/wireguard/wg0.conf:

```
nano /etc/wireguard/wg0.conf
```
```
[Interface]
PrivateKey = <privatekey>
Address = 10.0.0.1/24
ListenPort = 51830
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

### Настраиваем IP форвардинг:
```
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
```
```
sysctl -p
```

### Включаем:
```
systemctl enable wg-quick@wg0.service
```
```
systemctl start wg-quick@wg0.service
```
```
systemctl status wg-quick@wg0.service
```

### Создаём ключи клиента:
```
wg genkey | tee /etc/wireguard/goloburdin_privatekey | wg pubkey | tee /etc/wireguard/goloburdin_publickey
```

### Добавляем клиента в конфиг сервера:
```
nano /etc/wireguard/wg0.conf
```
```
[Peer]
PublicKey = <nvvertinsky_publickey>
AllowedIPs = 10.0.0.2/32
```


### Перезагружаем systemd сервис с wireguard:
```
systemctl restart wg-quick@wg0
```
```
systemctl status wg-quick@wg0
```

### На локальной машине создаём текстовый файл с конфигом клиента:
```
nano nvvertinsky_wb.conf

[Interface]
PrivateKey = <CLIENT-PRIVATE-KEY>
Address = 10.0.0.2/32
DNS = 8.8.8.8

[Peer]
PublicKey = <SERVER-PUBKEY>
Endpoint = <SERVER-IP>:51830
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 20
```

### Проверить нагрузку на порте
````
vnstat -l
rx (receive)
tx (transmit)
мгновенный / средний / пиковый
````

# udp2raw
```
cd ~ && curl -fLo udp2raw_binaries.tar.gz https://github.com/wangyu-/udp2raw/releases/download/20230206.0/udp2raw_binaries.tar.gz
```
```
tar -xzf udp2raw_binaries.tar.gz udp2raw
```
```
chmod +x udp2raw
```
```
sudo mv udp2raw /usr/local/bin
```
```
udp2raw --version
```

```
sudo nano /etc/systemd/system/udp2raw.service
```
Для сервера 
```
[Unit]
Description=udp2raw
After=network-online.target
Wants=network-online.target

[Service]
User=root
Type=exec
ExecStart=/usr/local/bin/udp2raw -s -l0.0.0.0:51830 -r localhost:51830 -k "passwd" --raw-mode faketcp -a
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Для клиента:
```
[Unit]
Description=udp2raw
After=network-online.target
Wants=network-online.target

[Service]
User=root
Type=exec
ExecStart=/usr/local/bin/udp2raw -c -l0.0.0.0:51830  -r44.55.66.77:51830  -k "passwd" --raw-mode faketcp -a
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl enable udp2raw
```
```
sudo systemctl start udp2raw
```
