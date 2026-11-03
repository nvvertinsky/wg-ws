# Настройка iptables (добавить -A, удалить -D, выполнится в конце -P)

```
sudo iptables -S                                       # Посмотреть все текущие правила
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
iptables -A INPUT -p udp --dport 19998 -j ACCEPT       # obsfucator
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
chmod +x ck
```
```
sudo mv ck /usr/local/bin
```
```
ck --help
```
```
ck -k    # PrivateKey и PublicKey
ck -uid  # UID
```
```
nano /etc/ck.json
```
Сервер
```
{
  "ProxyBook": {
    "wireguard": ["udp", "127.0.0.1:51830"]
  },
  "BindAddr": [":443", ":80"],
  "BypassUID": ["<ВСТАВЬТЕ_UID>"],
  "RedirAddr": "cloudflare.com",
  "PrivateKey": "<ПРИВАТНЫЙ_КЛЮЧ>",
  "AdminUID": "",
  "DatabasePath": "userinfo.db"
}
```
Клиент
```
{
  "Transport": "direct",
  "ProxyMethod": "wireguard",
  "EncryptionMethod": "chacha20-poly1305",
  "UID": "<ВСТАВЬТЕ_UID>",
  "PublicKey": "<ПУБЛИЧНЫЙ_КЛЮЧ>",
  "ServerName": "vertinskylab.ru",
  "NumConn": 4,
  "BrowserSig": "chrome",
  "StreamTimeout": 300
}
```

```
sudo nano /etc/systemd/system/ck.service
```
```
[Unit]
Description=wg-obfuscator
After=network-online.target
Wants=network-online.target

[Service]
User=root
Type=exec
ExecStart=/usr/local/bin/ck-server -c /etc/ck.json
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
```
sudo systemctl enable ck
```
```
sudo systemctl start ck
```

<br>
<br>
<br>

# Маршрутизация

Как работает сейчас:
1. Зашифрованный UDP-пакет идет на сервер на интерфейс eth0
2. Wireguard сервер принимает пакет и расшифровывает
3. После расшифровки чистый пакет помещается в интерфейс wg0 как входящий
4. Ядро смотрит на IP-адрес назначения пакета и, в соответствии с таблицей маршрутизации, решает отправить его через eth0


Как должно работать:
1. Зашифрованный UDP-пакет идет на сервер на интерфейс eth0
2. Wireguard сервер принимает пакет и расшифровывает
3. После расшифровки чистый пакет помещается в интерфейс wg0 как входящий
4. Ядро смотрит на IP-адрес назначения пакета. Но теперь оно сначала проверяет: есть ли этот адрес в специальных списках (наборах IP-адресов)
5. Если адрес есть в списке, ядро помечает этот пакет специальной меткой.
6. Далее ядро смотрит на правила маршрутизации
7. Для непомеченных пакетов ничего не меняется: они идут по обычной таблице маршрутизации через eth0 (с NAT) прямо в интернет.
8. Для помеченных пакетов срабатывает отдельное правило, которое говорит: «Пакеты с такой меткой отправляй не через основной шлюз, а в специальную таблицу маршрутизации». В этой таблице прописано: «Отправлять через интерфейс wg1» — это туннель до Сервера 2.
9. На Сервере 2: зашифрованный пакет приходит через туннель wg1, WireGuard расшифровывает его, и Сервер 2 отправляет чистый пакет в интернет через свой eth0 (со своим NAT).

<br>

```
nano /etc/wireguard/wg1.conf         # Настройки для клиента (сервер 1)
```
```
[Interface]
PrivateKey =                         # Приватный ключ клиента
Address = 10.0.0.2/32                   
ListenPort = 51830

[Peer]
PublicKey =                          # Публичный ключ Сервера
AllowedIPs = 0.0.0.0/0
Endpoint = :51830
PersistentKeepalive = 25
```
```
ipset create ru_domains hash:net     # Таблица с доменами
```
```
nano /etc/dnsmasq.conf               # Все IP-адреса, полученные в ответ на запросы доменов, заканчивающихся на .ru, добавляй в набор ru_domains
ipset=/ru/ru_domains
```
```
# Помечаем пакеты с ru адресом, меткой 0x1
# Делать это лучше в цепочке PREROUTING таблицы mangle, так как она обрабатывает пакеты до принятия решения о маршрутизации.
iptables -t mangle -A PREROUTING -m set --match-set ru_domains dst -j MARK --set-xmark 0x2
```
```
nano /etc/wireguard/wg1.conf            # Сервер
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
ip rule add fwmark 0x2 table 100 pref 100   # Пакеты с меткой 0x1 должны маршрутизироваться с использованием таблицы с номером 100
```
```
ip route add default dev eth0 table 100     # Маршрут по умолчанию через eth0 (Сервер 1)
```
