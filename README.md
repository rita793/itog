# Ансибл
Создаем ключ для ансибла, если не сделан.
Ключ требуется паблик полижить при создании машины в клауде для пользователя.
```
ssh-keygen -t ed25519
```
Ключ создается в директории в которой находимся.

Ставим ансибл на тачку с ubuntu/debian без разницы.
```
apt install ansible
```
Создаем иерарзию папок для ансибла, так удобней, чтобы не было путаницы.
```
mkdir -p /opt/ansible/playbook && mkdir -p /opt/ansible/inventory && mkdir -p /opt/ansible/roles
```
в папке /opt/ansible делаем конфиг для ансибла
```
nano ansible.cfg
```
созданный ссх ключ, приватник закидываем в папку рядом с конфигурацией и переименовываем его/
Ключ должен называться и лежать в папке /opt/ansible/key
Требуется ограничить права на ключ, так как это секрность.
```
chmod 600 /opt/ansible/key
```
## Иерархия ансибла.
/opt/ansible/playbook - что будем запускать, какую роль, для какой группы
/opt/ansible/inventory - предназначен для описание групп хостов.
/opt/ansible/roles - описание ролей, тасков - что будем выполнять
Дополнительно в /opt/ansible/roles/tasks/ тут лежат файлы, которые использует ансибл для настройки машин.

-------
-------
На машине centos требуется руками поставить питон
```
yum install -y python3
```
Питон понадобиться для ансибла.
Требования к машине - минимум 4 cpu, 20гб диск если можно то больше, 8гб и более памяти, так как стек elk прожорливый, так как используется Java

-------
# ВАЖНО!!!
гит не позволяет закинуть файлы объемом более 100мб, поэтому стек ЕЛК в рпм вручную нужно закинуть в машину с асиблом.



## Работа с ансиблом.
На тачке где работает ансибл, в инвентарники потребуется сменить IP и название машины на Ваше.
После того как заполнили инвентарь, потребуется перейти в директорию ансибла.
```
cd /opt/ansible
```

Отсюда запускаем нашего монстра
```
ansible-playbook -i /opt/ansible/inventory/inventory.yaml playbook/install_centos.yml
```

После того, когда у нас прокатиться плейбук, у нас на тачке Центос будут установлены:
openvpn и сразу же тачка будет под vpn
ELK стек.

--------

# Настройка тачки с графаной,  и впн сервер.

```
apt-get install -y apt-transport-https software-properties-common wget
```
добавляем репозиторий графаны
```
echo "deb [trusted=yes] https://mirror.yandex.ru/mirrors/packages.grafana.com/enterprise/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
```
Выполняем команду
```
apt update
```
Ставим графану
```
apt install grafana-enterprise
```
Смотрим статус и запускаем ее.
```
systemctl status grafana-enterprise
systemctl start grafana-enterprise
```
Крутиться на порту 3000 по внешнему ip
пароль admin/admin, после первого входа попросит сменить на нужный.

----------------


# Установка OpenVpn сервера
```
apt install openvpn
apt install easy-rsa       для генерации сертов
```

Создаем серты в папке домашней!!!
```
mkdir -p /home/dmitriy/openvpn-ca && cd /home/dmitriy/openvpn-ca
копируем содержимое для сертов cp -r /usr/share/easy-rsa/* .
chmod 700 .
```

Создаем файл конфигурации для сертификатов nano vars
```
set_var EASYRSA_REQ_COUNTRY    "RU"
set_var EASYRSA_REQ_PROVINCE   "Moscow"
set_var EASYRSA_REQ_CITY       "Moscow"
set_var EASYRSA_REQ_ORG        "MyVPN"
set_var EASYRSA_REQ_EMAIL      "admin@myvpn.com"
set_var EASYRSA_REQ_OU         "MyVPN-CA"
set_var EASYRSA_ALGO           "ec"
set_var EASYRSA_DIGEST         "sha512"
set_var EASYRSA_CA_EXPIRE      3650
set_var EASYRSA_CERT_EXPIRE    825
set_var EASYRSA_NS_SUPPORT     "yes"

```
Инициализация 
```
./easyrsa init-pki
./easyrsa build-ca nopass
```

Генерация сертов для сервера, везде ставим yes
```
./easyrsa gen-req server nopass
./easyrsa sign-req server server
```

Создаем Тлс для ключа
```
openvpn --genkey --secret /home/dmitriy/openvpn-ca/pki/ta.key
```
Делаем отдельную папку для сертов
```
mkdir -p /etc/openvpn/server
```

Копируем серты
```
cp /home/dmitriy/openvpn-ca/pki/ca.crt /etc/openvpn/server/
cp /home/dmitriy/openvpn-ca/pki/issued/server.crt /etc/openvpn/server/
cp /home/dmitriy/openvpn-ca/pki/private/server.key /etc/openvpn/server/
cp /home/dmitriy/openvpn-ca/pki/ta.key /etc/openvpn/server/
```

Создаем серты для клинта, везде жмкаем yes
```
./easyrsa gen-req client1 nopass
./easyrsa sign-req client client1
```

генерируем DH параметры
```
openssl dhparam -out pki/dh.pem 2048
cp /home/dmitriy/openvpn-ca/pki/dh.pem /etc/openvpn/server/
```

Делаем конфиг для впн /etc/openvpn/server.conf
```
port 443
proto tcp
dev tun
ca /etc/openvpn/server/ca.crt
cert /etc/openvpn/server/server.crt
key /etc/openvpn/server/server.key
tls-auth /etc/openvpn/server/ta.key 0
dh /etc/openvpn/server/dh.pem
topology subnet
server 10.8.0.0 255.255.255.0
push "redirect-gateway def1"
push "dhcp-option DNS 8.8.8.8"
keepalive 10 60
cipher AES-256-GCM
auth SHA512
user nobody
group nogroup
persist-key
persist-tun
status /var/log/openvpn-status.log
verb 3
```

Стартуем сервис
```
systemctl start openvpn@server.service
```

Открытие портов
```
sudo ufw allow 443/tcp
sudo ufw allow ssh
sudo ufw enable
```
потом echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sysctl -p
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

# На сервере клиенте делаем настройку
# конфиги уже готовые разложены по папкам, ансиблом полностью настраивается тачка, для убунты свой конфиг
nano /etc/openvpn/client/client1.conf
```
client
dev tun
proto tcp
remote 158.160.163.92 443 #IP сервера впн
route deb.debian.org 255.255.255.255 net_gateway
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-GCM
auth SHA512
verb 3

<ca>
-----BEGIN CERTIFICATE-----
MIIB+zCCAYKgAwIBAgIUBb8BBwWtgaBYadHE3eDxH+alvKIwCgYIKoZIzj0EAwQw
FjEUMBIGA1UEAwwLRWFzeS1SU0EgQ0EwHhcNMjUwNDE1MjMyMjAxWhcNMzUwNDEz
MjMyMjAxWjAWMRQwEgYDVQQDDAtFYXN5LVJTQSBDQTB2MBAGByqGSM49AgEGBSuB
BAAiA2IABEd/TRkLD/5kq7tU6nkfueGzsPbDzUiuod4rYQKD//E+tH7EsZT6+d6K
ph1y4MwWy/IBltWmdX0mwppTwq9fII9NjFH5rTYH0xttqDYSJeKI9sFFpBjtN4bu
X+r9R+8nvKOBkDCBjTAMBgNVHRMEBTADAQH/MB0GA1UdDgQWBBQ8a0VdyVJv8xI4
tqu3iLTGnhxWzjBRBgNVHSMESjBIgBQ8a0VdyVJv8xI4tqu3iLTGnhxWzqEapBgw
FjEUMBIGA1UEAwwLRWFzeS1SU0EgQ0GCFAW/AQcFrYGgWGnRxN3g8R/mpbyiMAsG
A1UdDwQEAwIBBjAKBggqhkjOPQQDBANnADBkAjBDH1GvoOdlCYctR6eRn+g65lr7
eHOH/U2LqLzrM/W+JVRiYLLqd6OAuwkg9nSITWQCMDThUJTabBmS4AvLgTCkf7jh
0Ck7PDbOjxkKJPYmWRy7LI3yNTD18oeiTIyNaDK42A==
-----END CERTIFICATE-----
</ca>

<cert>
-----BEGIN CERTIFICATE-----
MIICUDCCAdagAwIBAgIQRRLaqxapYEb6RxH4iBkLuTAKBggqhkjOPQQDBDAWMRQw
EgYDVQQDDAtFYXN5LVJTQSBDQTAeFw0yNTA0MTUyMzMxMzdaFw0yNzA3MTkyMzMx
MzdaMBIxEDAOBgNVBAMMB2NsaWVudDEwdjAQBgcqhkjOPQIBBgUrgQQAIgNiAAQ6
5dhr78RNNJjaMBVffJxGL1zkLGcdlJTBIlcVOrZ35vwKxHG4bSxkzAAJkiEONENN
RSj8IoJVc3Q04YMQyd9FqIDH5pSjw6CipMDreXzO60VpTWENiJV8btlqL7oQ21Gj
gewwgekwCQYDVR0TBAIwADAdBgNVHQ4EFgQUYa1/XKYhsdMXRTXelGNeUnOIx3ow
UQYDVR0jBEowSIAUPGtFXclSb/MSOLart4i0xp4cVs6hGqQYMBYxFDASBgNVBAMM
C0Vhc3ktUlNBIENBghQFvwEHBa2BoFhp0cTd4PEf5qW8ojATBgNVHSUEDDAKBggr
BgEFBQcDAjALBgNVHQ8EBAMCB4AwEQYJYIZIAYb4QgEBBAQDAgeAMDUGCWCGSAGG
+EIBDQQoFiZFYXN5LVJTQSAoMy4xLjcpIEdlbmVyYXRlZCBDZXJ0aWZpY2F0ZTAK
BggqhkjOPQQDBANoADBlAjAu5mJGRgVX690F6gQHdG87cio+kdEDxfPo6nccHnR6
fLkM9gv+nfZ5ZGiBTwNvRZkCMQDqdv8yL6wK8ruOLXqp8hXgnF1JoAhVL8mBRrpq
CtmWciRGhpKoShxIv29fCjkzWj0=
-----END CERTIFICATE-----
</cert>

<key>
-----BEGIN PRIVATE KEY-----
MIG2AgEAMBAGByqGSM49AgEGBSuBBAAiBIGeMIGbAgEBBDBBQ9eXjT9wxBs2snad
otdB6lbz1q6tOBne9CKFEJgguIiA1p+sAGXknm9PKaHZ56ahZANiAAQ65dhr78RN
NJjaMBVffJxGL1zkLGcdlJTBIlcVOrZ35vwKxHG4bSxkzAAJkiEONENNRSj8IoJV
c3Q04YMQyd9FqIDH5pSjw6CipMDreXzO60VpTWENiJV8btlqL7oQ21E=
-----END PRIVATE KEY-----
</key>

<tls-auth>
-----BEGIN OpenVPN Static key V1-----
d727e4372cd204d022898badb0e015b5
22479b4aadd8927c7ea32788a0d7ce23
b4f30f1930be706f6b437bbab869db6d
fe559fcecfdeef7c17a85707493c76d8
8906a013434fc297345b0aaa7b03fc67
ee72d64a9ec292fbad684f7cbb4c0970
63569827fe4542a3ddbdee02aaf470c9
4045a57c8b1d3b62b0a4ac9f1fcb57e5
b5c04fa1d3fe15ebec59fb8c8a5f0d46
0a74cda6ed8d1f975c75f38471d0797b
24197d7603f020ebc2bcbd1c97349aee
7d5160fb7010af4e9d255ec0e908c92e
d9026632093bfe7fcf046b8476f90d4b
a643d2d2fa0f5b2dce51705d9212d07a
4b46e71813f3f368ac8900a41fbbc8d1
5b0831a443329594fe11b18a6067a44c
-----END OpenVPN Static key V1-----
</tls-auth>
key-direction 1

```
Запускаем клиент сервер
```
systemctl restart openvpn-client@client1
```

На тачке сервера сделать правило
```
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
```

Проверка, сотрим лог файл , что клиент присоединился.
tail -f /var/log/openvpn-status.log


# Проверка работы елк
ЕЛК будет выключен, так как надо на тачку много ресурсов.
Когда ресурсы будут, врубить через сервисы стек елк и проверить работу

curl -X GET "localhost:9200"
systemctl status elasticsearch
systemctl status kibana











