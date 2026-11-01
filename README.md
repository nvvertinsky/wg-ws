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

<br>
<br>
<br>

# Установка wstunnel
```
cd ~
```
```
curl -fLo wstunnel.tar.gz https://github.com/erebe/wstunnel/releases/download/v11.1.0/wstunnel_11.1.0_linux_amd64.tar.gz
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

server
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

client
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

<br>
<br>
<br>

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

Полученный интефейс (eth0, ens3) инспользуем в /etc/wireguard/wg0.conf:
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
```
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
```
```
sysctl -p
```
```
systemctl enable wg-quick@wg0.service
```
```
systemctl start wg-quick@wg0.service
```
```
systemctl status wg-quick@wg0.service
```

Создаём ключи клиента:
```
wg genkey | tee /etc/wireguard/goloburdin_privatekey | wg pubkey | tee /etc/wireguard/goloburdin_publickey
```

Добавляем клиента в конфиг сервера:
```
nano /etc/wireguard/wg0.conf
```
```
[Peer]
PublicKey = <nvvertinsky_publickey>
AllowedIPs = 10.0.0.2/32
```
```
systemctl restart wg-quick@wg0
```
```
systemctl status wg-quick@wg0
```

На локальной машине создаём текстовый файл с конфигом клиента:
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

Проверить нагрузку на порте
````
vnstat -l
rx (receive)
tx (transmit)
мгновенный / средний / пиковый
````
<br>
<br>
<br>

# udp2raw
```
cd ~
```
```
curl -fLo udp2raw_binaries.tar.gz https://github.com/wangyu-/udp2raw/releases/download/20230206.0/udp2raw_binaries.tar.gz
```
```
tar -xzf udp2raw_binaries.tar.gz udp2raw_amd64
```
```
chmod +x udp2raw_amd64
```
```
sudo mv udp2raw_amd64 /usr/local/bin
```
```
udp2raw_amd64 --help
```

```
sudo nano /etc/systemd/system/udp2raw_amd64.service
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
ExecStart=/usr/local/bin/udp2raw_amd64 -s -l0.0.0.0:51830 -r 127.0.0.1:51830 -k "passwd" --raw-mode faketcp -a
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
ExecStart=/usr/local/bin/udp2raw_amd64 -c -l0.0.0.0:51830  -r44.55.66.77:51830  -k "passwd" --raw-mode faketcp -a
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl enable udp2raw_amd64
```
```
sudo systemctl start udp2raw_amd64
```


<br>
<br>
<br>


# wg-obfuscator
```
cd ~
```
```
curl -fLo wg-obfuscator-v1.6-linux-x64.tar.gz https://github.com/ClusterM/wg-obfuscator/releases/download/v1.6/wg-obfuscator-v1.6-linux-x64.tar.gz
```
```
tar -xzf wg-obfuscator-v1.6-linux-x64.tar.gz wg-obfuscator
```
```
chmod +x wg-obfuscator
```
```
sudo mv wg-obfuscator /usr/local/bin
```
```
wg-obfuscator --help
```
```
nano /etc/wg-obfuscator.conf
```
Сервер
```
[main]
source-lport = 19999        # Внешний порт, на который будут подключаться клиенты
target = 127.0.0.1:51830    # Адрес и порт вашего локального WireGuard-сервера
key = "ВАШ_СЕКРЕТНЫЙ_КЛЮЧ"  # Придумайте надежный пароль-ключ
masking = AUTO              # Автоматически подстраиваться под маскировку клиента
```

Клиент
```
[main]
source-lport = 51830             # Локальный порт, на который будет отправлять трафик ваш WireGuard-клиент
target = IP_ВАШЕГО_СЕРВЕРА:19999 # Внешний IP и порт сервера, где запущен wg-obfuscator
key = "ВАШ_СЕКРЕТНЫЙ_КЛЮЧ"       # Должен совпадать с ключом на сервере
masking = STUN                   # Тип маскировки (например, STUN, чтобы трафик выглядел как видеозвонок)
```

```
sudo nano /etc/systemd/system/wg-obfuscator.service
```
```
[Unit]
Description=wg-obfuscator
After=network-online.target
Wants=network-online.target

[Service]
User=root
Type=exec
ExecStart=/usr/local/bin/wg-obfuscator -c /etc/wg-obfuscator.conf
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
```
sudo systemctl enable wg-obfuscator
```
```
sudo systemctl start wg-obfuscator
```
