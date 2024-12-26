---
layout: post
title: "Очистка истории git"
---
Иногда необходимо очистить коммиты в репозитории.

Сделайте бэкап локального репозитория. Можно запушить на резервный удаленный репозиторий, а можно просто взять и переместить папку .git в другое место.
```
mkdir ../git-backup
mv .git ../git-backup/.git
```

Если не переместили локально, а забэкапили куда-то еще: удаляем папку.
`rm -Rf .git`

Чтобы создать локальный репозиторий с главной веткой "main" а не "master" по умолчанию, необходимо настроить имя основной ветки по умолчанию командой:
`git config --global init.defaultBranch main`

Теперь заново инициализируем репозиторий:
`git init`

Добавляем все файлы в рабочей области и делаем коммит.
```
git add *
git commit -m 'начал с нуля'
```

## Когда все готово
Подключаем удаленный репозиторий и заливаем на него изменения:
```
git remote add origin <url>
git push -f origin main
```