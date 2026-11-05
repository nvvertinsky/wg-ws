# Настройка iptables (добавить -A, удалить -D, выполнится в конце -P)

```
sudo iptables -S                                       
```
```
iptables -A INPUT -i lo -j ACCEPT                      # Все порты доступны для запросов с самого сервера
```
```
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT   # Разрешаем Ответные пакеты для уже установленных соединений
```
```
iptables -A INPUT -p tcp --dport 22 -j ACCEPT          # Разрешаем ssh
```
```
iptables -A INPUT -p tcp --dport 80  -j ACCEPT         # Разрешаем HTTP
iptables -A INPUT -p tcp --dport 443 -j ACCEPT         # Разрешаем HTTPS
```
```
iptables -A INPUT -p udp --dport 19998 -j ACCEPT       # Дополнительный порт который нужно открыть на вход
```
```
iptables -A INPUT -p icmp -j ACCEPT                    # чтобы ping работал
```
```
iptables -P INPUT DROP                                 # Запрещаем все остальное
```


<br>
<br>
<br>

# wstunnel
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

# wireguard

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
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf   # Это пересылка IP-пакетов между интерфейсами eth0 и wg0
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



<br>
<br>
<br>


# Cloak
```
cd ~
```
```
curl -fLo ck https://github.com/cbeuw/Cloak/releases/download/v2.12.0/ck-server-linux-amd64-v2.12.0
```
```
curl -fLo ck-client https://github.com/cbeuw/Cloak/releases/download/v2.12.0/ck-client-linux-amd64-v2.12.0
```
```
chmod +x ck
chmod +x ck-client
```
```
sudo mv ck /usr/local/bin
sudo mv ck-client /usr/local/bin
```
```
ck --help
ck-client --help
```
```
ck -k    # PrivateKey и PublicKey
ck -uid  # UID
```
```
nano /etc/ck.json
nano /etc/ck-client.json
```
Сервер
```
{
  "ProxyBook": {
    "wireguard": [
      "udp",
      "127.0.0.1:51830"
    ]
  },
  "BindAddr": [
    ":443",
    ":80"
  ],
  "BypassUID": [
    "---Bypass UID here---"
  ],
  "RedirAddr": "cloudflare.com",
  "PrivateKey": "---Private key here---",
  "AdminUID": "---Admin UID here (optional)---",
  "DatabasePath": "userinfo.db"
}
```
Клиент
```
{
  "Transport": "direct",
  "ProxyMethod": "wireguard",
  "EncryptionMethod": "plain",
  "UID": "---Your UID here---",
  "PublicKey": "---Public key here---",
  "ServerName": "www.bing.com",
  "NumConn": 4,
  "BrowserSig": "chrome",
  "StreamTimeout": 300
}
```
```
sudo nano /etc/systemd/system/ck.service
sudo nano /etc/systemd/system/ck-client.service
```
```
[Unit]
Description=cl
After=network-online.target
Wants=network-online.target

[Service]
User=root
Type=exec
ExecStart=/usr/local/bin/ck -c /etc/ck.json
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
```
[Unit]
Description=cl
After=network-online.target
Wants=network-online.target

[Service]
User=root
Type=exec
ExecStart=/usr/local/bin/ck-client -s <server> -c /etc/ck.json -u -i 0.0.0.0 -l 51830
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
```
sudo systemctl enable ck
sudo systemctl enable ck-client
```
```
sudo systemctl start ck
sudo systemctl start ck-client
```

<br>
<br>
<br>

# Маршрутизация

```
apt update 
apt install -y wireguard iptables ipcalc curl jq traceroute dnsutils ufw
```
```
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
echo "net.ipv4.conf.all.forwarding=1" >> /etc/sysctl.conf
sysctl -p /etc/sysctl.conf
```
```
hostnamectl set-hostname moscow  # Переименовываем хост
hostnamectl set-hostname berlin  # Переименовываем хост
```
```
wg genkey   # Приватный ключ для Москвы
wg genkey   # Приватный ключ для Берлина
echo "kOd3FVBggwpjD3AlZKXUxNTzJT0+f3MJdUdR8n6ZBn8=" | wg pubkey    # Генерим публичный ключ для Москвы
echo "6CCRP42JiTObyf64Vo0BcqsX6vptsqOU+MKUslUun28=" | wg pubkey    # Генерим публичный ключ для Берлина
```
```
nano /etc/wireguard/wg-moscow.conf
```
```
[Interface]
Address = 10.20.30.1/32
ListenPort = 17968
PrivateKey = <private_key>
PostUp = iptables -t nat -A POSTROUTING -o `ip route | awk '/default/ {print $5; exit}'` -j MASQUERADE
PostUp = ip rule add from `ip addr show $(ip route | awk '/default/ { print $5 }') | grep "inet" | grep -v "inet6" | head -n 1 | awk '/inet/ {print $2}' | awk -F/ '{print $1}'` table main
PostDown = iptables -t nat -D POSTROUTING -o `ip route | awk '/default/ {print $5; exit}'` -j MASQUERADE
PostDown = ip rule del from `ip addr show $(ip route | awk '/default/ { print $5 }') | grep "inet" | grep -v "inet6" | head -n 1 | awk '/inet/ {print $2}' | awk -F/ '{print $1}'` table main

#Berlin
[Peer] 
PublicKey = <public_key>
AllowedIPs = 10.20.30.2/32, 0.0.0.0/0

#notebook-client
[Peer]
PublicKey = <public_key>
AllowedIPs = 10.20.30.3/32
```
```
nano /etc/wireguard/wg-berlin.conf
```
```
[Interface]
Address = 10.20.30.2/32
ListenPort = 17968
PrivateKey = <private_key>
PostUp = iptables -t nat -A POSTROUTING -o `ip route | awk '/default/ {print $5; exit}'` -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o `ip route | awk '/default/ {print $5; exit}'` -j MASQUERADE

# Moscow
[Peer]
PublicKey=<public_key>
AllowedIPs=10.20.30.0/24
Endpoint=<ip>
PersistentKeepalive=25
```
```
wg-quick down wg-berlin ; wg-quick up wg-berlin
```
```
wg-quick down wg-moscow ; wg-quick up wg-moscow
```
```
wg
```
```
systemctl enable wg-quick@wg-moscow
```
```
systemctl enable wg-quick@wg-berlin
```
```
/etc/wireguard/wg-notebook-client.conf   # Создаем конфиг для клиента
```
```
[Interface]
Address = 10.20.30.3/32
PrivateKey = <private_key client>
DNS = 1.1.1.1, 8.8.8.8

[Peer]
PublicKey = <public_key moscow>
AllowedIPs = 0.0.0.0/0
Endpoint = <moscow>
PersistentKeepalive = 25
```
```
ip r                                       # Выполняем на сервере Moscow
default via 195.2.79.1 dev ens3 onlink     # Дефолтный шлюз через который выходит в интернет
```
```
target_ip="212.193.158.175/32"                               # nslookup sbermarket.ru
gateway=`ip route | awk '/default/ {print $3; exit}'`        
gateway_device=`ip route | awk '/default/ {print $5; exit}'`
ip route add $target_ip via $gateway dev $gateway_device     # Будем выходить sbermarket.ru из Moscow сервера
```

Не исключать российские IP из туннеля, а наоборот, отправлять только определенные IP в туннель?
```
nano /etc/wireguard/wg-moscow.conf
```
```
PostUp = ip route delete default dev wg-moscow table 51820
```
```
ip route add 8.8.8.8 dev wg-moscow
```
