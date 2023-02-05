# VPN на StrongSwan и прокси ShadowSocks в Ubuntu 20.04 LTS

## Ставим и настраиваем strongswan на сервере

* Первый шаг стандартный для установки чего угодно:
  `apt update`
* Теперь ставим нужные пакеты:
  `apt install strongswan strongswan-pki libcharon-extra-plugins libcharon-extauth-plugins libstrongswan-extra-plugins libtss2-tcti-tabrmd-dev`

## Генерируем сертификаты

* создаем каталоги для сертификатов и ключей:
  `mkdir -p ~/pki/{cacerts,certs,private}`
* Генерируем корневой ключ RSA:
  `pki --gen --type rsa --size 4096 --outform pem > ~/pki/private/ca-key.pem`
* И корневой сертификат:
  `pki --self --ca --lifetime 3650 --in ~/pki/private/ca-key.pem --type rsa --dn "CN=LLHOST VPN" --outform pem > ~/pki/cacerts/ca-cert.pem`
  Число 3650 — это десять лет (в днях), которые будет работать сертификат. Можешь изменить это значение на свое усмотрение. 
* Корневые сертификаты готовы, теперь нам нужен сертификат для нашего сервера.
  Делаем приватный ключ:
  `pki --gen --type rsa --size 4096 --outform pem > ~/pki/private/server-key.pem`
* И сам сертификат. Обрати внимание, что IP в трех местах нужно изменить на IP твоего сервака.
  `pki --pub --in ~/pki/private/server-key.pem --type rsa | pki --issue --lifetime 1825 --cacert ~/pki/cacerts/ca-cert.pem --cakey ~/pki/private/ca-key.pem --dn "CN=NNN.NNN.NNN.NNN" --san @NNN.NNN.NNN.NNN --san NNN.NNN.NNN.NNN --flag serverAuth --flag ikeIntermediate --outform pem > ~/pki/certs/server-cert.pem`
* С этим закончили, можем перемещать ключи и сертификаты из домашнего каталога в папку настроек strongSwan:
  `cp -r ~/pki/* /etc/ipsec.d/`

## Настраиваем strongSwan

На всякий случай переименуем старый файл с настройками, если он есть:
  `mv /etc/ipsec.conf{,.original}`
* Откроем его /etc/ipsec.conf в nano:
  `nano /etc/ipsec.conf`
* Добавляем раздел `config setup`:
```
config setup
  charondebug="ike 1, knl 1, cfg 0"
  uniqueids=no
```
* Дальше добавляем настройки туннеля IKEv2 (не забудь подставить свой IP):
```
conn ikev2-vpn
  auto=add
  compress=no
  type=tunnel
  keyexchange=ikev2
  fragmentation=yes
  forceencaps=yes
  dpdaction=clear
  dpddelay=300s
  rekey=no
  left=%any
  leftid=NNN.NNN.NNN.NNN
  leftcert=server-cert.pem
  leftsendcert=always
  leftsubnet=0.0.0.0/0
  right=%any
  rightid=%any
  rightauth=eap-mschapv2
  rightsourceip=10.10.10.0/24
  rightdns=8.8.8.8,8.8.4.4
  rightsendcert=never
  eap_identity=%identity
  ike=chacha20poly1305-sha512-curve25519-prfsha512,aes256gcm16-sha384-prfsha384-ecp384,aes256-sha1-modp1024,aes128-sha1-modp1024,3des-sha1-modp1024!
  esp=chacha20poly1305-sha512,aes256gcm16-ecp384,aes256-sha256,aes256-sha1,3des-sha1!
```
  Здесь забиты адреса гугловского DNS `(8.8.8.8 и 8.8.4.4)`, ты можешь использовать их или заменить, например, адресом своего роутера (его можно узнать, набрав `ip route show default`).
  сохраняем конфиг и выходим из nano (Ctrl-X, Y, Enter).
* сервер VPN мы настроили, осталось создать креды, с которыми клиент сможет авторизоваться. За них отвечает файл `/etc/ipsec.secrets`. Открываем его в nano:
   `nano /etc/ipsec.secrets`
* Добавляем в него строчку:
  `: RSA "server-key.pem"`
  Двоеточие и пробел после него важны, не убирай их!
* Дальше задаем логин и пароль пользователя. Замени их своей комбинацией. Хороший стойкий пароль можешь создать в парольном менеджере.
  `логин : EAP "пароль"`
* сохраняй файл, и перезапустим strongSwan, чтобы он прочитал новые конфиги:
  `systemctl restart strongswan-starter`

## Настраиваем сеть

Наш VPN уже работает и готов принимать соединения, но сетевой трафик до него пока не доходит. Нужно настроить файрвол и IP-форвардинг.
* Разрешаем пропускать трафик OpenSSH, а также UDP на портах 500 и 4500:
  `ufw allow OpenSSH`
  `ufw enable`
  `ufw allow 500,4500/udp`
* Осталось сделать так, чтобы входящие пакеты IPSec обрабатывались нашей службой. Узнаем адрес нашего сетевого интерфейса:
  `ip route show default`
  Посмотри, что будет написано после слова dev. Это может быть, например, eth0 или ens2 либо еще что‑нибудь в таком духе. Запомни или запиши куда‑нибудь эти буквы.
* Открываем файл с настройками правил файрвола:
  `nano /etc/ufw/before.rules`
* В самом начале файла, до секции *filter, нам надо добавить два блока. Замени в трех местах ИНТЕРФЕЙс тем, что мы нашли выше.
```
*nat
-A POSTROUTING -s 10.10.10.0/24 -o ens3 -m policy --pol ipsec --dir out -j ACCEPT
-A POSTROUTING -s 10.10.10.0/24 -o ens3 -j MASQUERADE
COMMIT

*mangle
-A FORWARD --match policy --pol ipsec --dir in -s 10.10.10.0/24 -o ens3 -p tcp -m tcp --tcp-flags SYN,RST SYN -m tcpmss --mss 1361:1536 -j TCPMSS --set-mss 1360
COMMIT
```
* А в конце *filter — такие две строки:
```
-A ufw-before-forward --match policy --pol ipsec --dir in --proto esp -s 10.10.10.0/24 -j ACCEPT
-A ufw-before-forward --match policy --pol ipsec --dir out --proto esp -d 10.10.10.0/24 -j ACCEPT
```
* Закрываем, сохраняем. Теперь открываем файл с настройками службы файрвола:
  `nano /etc/ufw/sysctl.conf`
* Добавляем в конец вот такие строки:
```
net/ipv4/ip_forward=1
net/ipv4/conf/all/accept_redirects=0
net/ipv4/conf/all/send_redirects=0
net/ipv4/ip_no_pmtu_disc=1
```
* Выключаем и включаем файрвол для применения настроек:
  `ufw disable`
  `ufw enable`

## Настраиваем подключение

* Нам нужно забрать с сервера свой сертификат, с которым мы будем подключаться. Он лежит в файле по такому пути:
  `/etc/ipsec.d/cacerts/ca-cert.pem`
  Например это можно сделать при помощи sftp, командой ` get /etc/ipsec.d/cacerts/ca-cert.pem`
  Файл с сертификатом будет скачан в домашний каталог.
* Для подключения к VPN тебе понадобятся три вещи: IP-адрес твоего сервера, файл с сертификатом и пара из логина и пароля, которые ты задал в файле `/etc/ipsec.secrets`.

## В macOS тебе нужно:

* Дважды кликнуть на файл с сертификатом и подтвердить его добавление в цепочку ключей, введя пароль.
* Открыть цепочку (Keychain Access), найти VPN root CA и задать нужные разрешения. Дважды кликни по нему, раскрой раздел «Доверять» (Trust) и выбери «Доверять всегда» (Always Trust) напротив строки IPSec.
* создать соединение в настройках сети. Нажимай плюсик внизу, выбирай VPN, IKEv2 и жми «создать» (Create), вписывай IP сервера в строки «Адрес сервера» и «Удаленный ID».
* В настройках соединения выбирай «Имя пользователя» и укажи свои логин и пароль.

## В Windows процесc немного другой:

* Открывай «Консоль управления», выбирай «Файл → Добавить или удалить оснастку → сертификаты → Добавить».
* Чтобы VPN работал для любого пользователя, выбирай «Учетная запись компьютера» и жми «Далее». Затем жми «Локальный компьютер» и «Закончить».
* В «Корне консоли» выбирай «Доверенные корневые центры сертификации для доверия федерации» и «сертификаты».
* В меню «Действия» в правой панели нажми «Все задачи» и «Импортировать».
* Вызови меню выбора файла, выставь тип X.509 и импортируй свой сертификат.
* Теперь нужно создать подключение к VPN. Зайди в «Панель управления → сеть интернет». В Windows 7 может понадобиться создать новое подключение к рабочему месту, в Windows 10 и 11 должен быть специальный раздел VPN. В любом случае вписывай адрес сервера, логин и пароль.
* Можешь подключаться!

## Ставим и настраиваем shadowsocks на сервере

* Устанавливаем Shadowsocks
  `apt install shadowsocks-libev`
* создаём конфигурационный файл
  `nano /etc/shadowsocks-libev/config.json`
* Вставляем в файл следующий текст, заменив IP-адрес сервера на ваш, и придумав новый пароль в разделе `password`:
{
"server": "NNN.NNN.NNN.NNN",
"server_port": "8388",
"password": "your_password",
"timeout": 600,
"method": "aes-256-gcm",
"fast_open": true
}
* Запускаем Shadowsocks:
  `systemctl restart shadowsocks-libev`
* Добавляем его в автозапуск:
  `systemctl enable shadowsocks-libev`
* Разрешаем пропускать трафик:
  `ufw allow 8388/tcp`
  `ufw allow 8388/udp`
* Выключаем и включаем файрвол для применения настроек:
  `ufw disable`
  `ufw enable`

## Step 2, Tune the kernel parameters (необязательно)

The priciples of tuning parameters for shadowsocks are:
* Reuse ports and conections as soon as possible.
* Enlarge the queues and buffers as large as possible.
* Choose the TCP congestion algorithm for large latency and high throughput.

Here is an example /etc/sysctl.conf of our production servers:
```
fs.file-max = 51200

net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.core.netdev_max_backlog = 250000
net.core.somaxconn = 4096

net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.ip_local_port_range = 10000 65000
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_mem = 25600 51200 102400
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_congestion_control = hybla
```

## Настраиваем ПК Windows:

* скачайте [Shadowsocks for Windows](https://shadowsocks.org/)
* Запустите Shadowsocks.
* Нажмите правой кнопкой на значок Shadowsocks в трее рядом с часами, перейдите в Servers > Edit server
* Введите параметры:
  * IP-адрес вашего сервера
  * Порт 8388
  * Метод aes-256-gcm
  * Ваш пароль
* Сохраните настройки.
* Снова нажмите на Shadowsocks правой кнопкой, выберите Enable.

## Настраиваем ПК MacOS:

* Установим Command-line Client:
  `brew install shadowsocks-libev`
* Используем тот-же конфигурационный файл config.json с добавлением опции local_port:
{
"server":"NNN.NNN.NNN.NNN",
"server_port":8388,
"local_port":1080,
"password":"barfoo!",
"timeout": 600,
"method": "aes-256-gcm",
"fast_open": true
}
* Запускаем в командной строке:
  `ss-local -c config.json`
* Заходим в настройки: Система - Сеть - Дополнительно - Включаем SOCKS5 прокси

## Настройка мобильных устройств:

Тут как раз максимально просто: скачайте бесплатное приложение [Outline](https://getoutline.org/ru/) для своего мобильного устройства, на официальном сайте есть ссылки для AppStore и Google Play. Для добавления в Outline необходимо сделать специфичную ссылку на свой сервер. Наберите в любом текстовом редакторе: method:password@hostname:port, именно так, без пробелов, это:
* aes-256-gcm;
* далее ваш пароль;
* адрес сервера и порт.

Скопируйте method:password@hostname:port и идите на сайт [Base64](https://base64.guru/converter), куда вставьте текст в окно конвертера и нажмите **Encode text to Base64**, получите текст примерно такого вида: `bWV0aG9kOnBhc3N3b3JkQGhvc3RuYW1lOnBvcnQ=`
Добавьте впереди `ss://`, получив ссылку вида:
`ss://bWV0aG9kOnBhc3N3b3JkQGhvc3RuYW1lOnBvcnQ=`
Эту ссылку отправьте на своё мобильное устройство (через Телеграмм, например), запустите программу Outline, скопируйте ссылку, Outline предложит вам добавить сервер. Согласитесь, с добавлением сервера и с созданием профиля VPN.

#Web #VPN