---
layout: post
title: "Запуск браузера и Xray-core в фоне на MacOS"
---

Задача: запустить локальный прокси-сервер и выйти через него браузером в интернет без изменения системных настроек, легко, за два клика мышкой по ярлыку. После того, как сёрфинг через прокси закончился, фоновый процесс `xray-core` будет завершён после закрытия окна браузера.

Для создания ярлыка, который одновременно запустит браузер с прокси и консольный клиент `xray-core` в фоновом режиме, можно использовать AppleScript в Automator. Мы будем запускать `xray-core` в фоне и отслеживать процесс браузера. Как только браузер закрывается, скрипт завершит `xray-core`.

```applescript
-- Запуск xray-core в фоне
do shell script "/path/to/xray-core -c /path/to/config.json > /dev/null 2>&1 &"
delay 2 -- Даем немного времени, чтобы xray-core запустился

-- Запуск браузера с указанием прокси
do shell script "open -a 'Google Chrome' --args --proxy-server=127.0.0.1:1080"

-- Проверка, запущен ли процесс браузера
repeat
    delay 5 -- Проверяем каждые 5 секунд
    set browserRunning to false
    try
        do shell script "pgrep -x 'Google Chrome'" -- Проверяем, запущен ли браузер
        set browserRunning to true
    end try

    -- Если браузер закрыт, завершаем xray-core
    if not browserRunning then
        do shell script "pkill -f 'xray-core'"
        exit repeat
    end if
end repeat
```

## Что делает этот скрипт:
### Запускает `xray-core` в фоне.
- `do shell script` — команда, которая позволяет выполнить команду оболочки (bash или zsh) из AppleScript.
- `/path/to/xray-core -c /path/to/config.json` — это путь к исполняемому файлу xray-core, который мы запускаем с параметром конфигурации config.json. Чтобы узнать путь до исполняемого файла `xray-core` в macOS или Linux, можно воспользоваться командой `which` в терминале.
- `> /dev/null 2>&1` — это перенаправление вывода:
  - `> /dev/nul`l — игнорирует стандартный вывод (stdout), т.е. все выводимые сообщения не будут отображаться.
  - `2>&1` — перенаправляет ошибочный вывод (stderr) в стандартный вывод, который мы тоже игнорируем, перенаправляя его в /dev/null.
  - `&` — это символ, который запускает процесс в фоне, чтобы он продолжал работать, не блокируя выполнение других команд.

### Запускает браузер (Google Chrome) с настройкой прокси.
```applescript
do shell script "open -a 'Google Chrome' --args --proxy-server=127.0.0.1:1080"
```

### Проверка, запущен ли процесс браузера
```applescript
repeat
    delay 5 -- Проверяем каждые 5 секунд
    set browserRunning to false
    try
        do shell script "pgrep -x 'Google Chrome'" -- Проверяем, запущен ли браузер
        set browserRunning to true
    end try
```
Каждые 5 секунд проверяет, работает ли процесс браузера (используя команду `pgrep` для поиска процесса).

### Если браузер закрыт, завершает процесс `xray-core` с помощью `pkill`.
```applescript
    if not browserRunning then
        do shell script "pkill -f 'xray-core'"
        exit repeat
    end if
end repeat
```

## Создание ярлыка через Automator
Чтобы упростить запуск этого скрипта, ты можешь использовать Automator::
- Открой Automator и выбери Приложение.
- Добавь действие "Запустить AppleScript".
- Вставь скрипт.
- Сохрани его как приложение (например, Запуск_и_Завершение_Сессии).
- Перетащи ярлык на рабочий стол для удобства.

Теперь ты можешь запускать браузер с прокси и `xray-core`, просто щелкнув по ярлыку.