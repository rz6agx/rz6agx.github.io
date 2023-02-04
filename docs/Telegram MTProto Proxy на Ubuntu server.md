# Telegram MTProto Proxy на Ubuntu server
MTProto — протокол разработанный командой Дурова предназначенный для шифрования трафика месенджера Telegram.
MTProto Proxy — промежуточный сервер, выполняющий роль посредника между пользователем и целевым сервером позволяя получать доступ к сервису если по каким-то причинам он недоступен скрывая свой IP адрес и маскируя трафик.

Для установки нам понадобится общий набор инструментов для сборки из исходного кода и пакеты разработки для **openssl** и **zlib**.
`apt install git curl build-essential libssl-dev zlib1g-dev`

Клонируем из репозиторий **Telegram MTProto Proxy** на GitHub и компилируем:
```bash
cd ~
git clone https://github.com/TelegramMessenger/MTProxy.git
cd MTProxy/
make
```

Раскидываем файлы по своим местам:
```bash
cp objs/bin/mtproto-proxy /usr/bin/
chmod 777 /usr/bin/mtproto-proxy
cd /etc
mkdir mtproto-proxy
cd mtproto-proxy
//Получить секретный файл, используемый для подключения к серверам телеграмм:
curl -s https://core.telegram.org/getProxySecret -o proxy-secret
//Получить текущую конфигурацию. Может меняться и требует обновления:
curl -s https://core.telegram.org/getProxyConfig -o proxy-multi.conf
```

Telegram предупреждает, что иногда этот файл может изменяться. И то что было бы хорошо обновлять его один раз в день. Сделаем так, чтобы файл обновлялся раз в сутки, редактируем crontab: `nano /etc/crontab`, вставим следующую строчку в конец файла, не забудьте отредактировать под ваши данные:
```bash
0 0 * * * curl -s https://core.telegram.org/getProxyConfig -o /etc/mtproto-proxy/proxy-multi.conf && systemctl restart mtproto-proxy
```
Перезапускаем cron: `/etc/init.d/cron restart`

Создаем секрет, который будет использоваться пользователями для подключения к вашему прокси.
`head -c 16 /dev/urandom | xxd -ps`

Создаем конфигурационный файл:
`sudo nano /etc/systemd/system/mtproto-proxy.service`

Копируем туда текст с содержимым:
```bash
[Unit]
Description=MTProxy
After=network.target

[Service]
ExecStart=/usr/bin/mtproto-proxy -u nobody -p 8888 --http-stats -H 443 -S <secret> --aes-pwd /etc/mtproto-proxy/proxy-secret /etc/mtproto-proxy/proxy-multi.conf -M 1
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
где:
443 — порт, используемый клиентами для подключения к прокси.
8888 — локальный порт для получения статистики. Получить ее можно только локально через loopback. Пример: wget localhost:8888/stats. 
На сервере может быть несколько секретов. Пример: -S <secret1> -S <secret2>
Если сервер расположен за NAT, требуется добавить параметр --nat-info <arg> <Внутренний адрес>:<Внешний адрес>. Не забываем прокинуть порты.
Другие команды можно посмотреть после старта сервиса командой: `mtproto-proxy -h`

Перезагружаем systemd для поиска новых или измененных юнитов:
`systemctl daemon-reload`

Запускаем службу с добавлением ее в автозагрузку:
```bash
systemctl restart mtproto-proxy
systemctl enable mtproto-proxy
```

Все готово. Для того, чтобы пользователи смогли подключиться к прокси генерируем ссылку:
`tg://proxy?server=5.181.202.204&port=443&secret=73b86d7de03d7653069d228f45cfb585`

#Web