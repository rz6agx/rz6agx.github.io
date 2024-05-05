---
layout: post
title: "Git. Несколько удалённых репозиториев"
---
Я использую несколько серверов репозиториев: github, gitlab, bitbucket. Возникает необходимость использовать несколько удалённых репозиториев в проекте.

## Общая информация
Локальный репозиторий можно связать с несколькими удалёнными репозиториями. Однако только одна из этих ссылок может называться origin. Остальные ссылки должны иметь другие имена. Команда `git remote -v` отображает все удалённые репозитории, связанные с вашим локальным репозиторием. Для отправки или получения кода из вашего удалённого репозитория по умолчанию используется короткое имя origin.

## Несколько удалённых репозиториев
Можно добавить несколько удалённых репозиториев по https:
```
git remote add github https://github.com/your_name/repository_name.git
git remote add gitlab https://gitlab.com/your_name/repository_name.git
git remote add bitbucket https://bitbucket.org/your_name/repository_name.git
```

или по ssh:
```
git remote add github git@github.com:your_name/repository_name.git
git remote add gitlab git@gitlab.com:your_name/repository_name.git
git remote add bitbucket git@bitbucket.org:your_name/repository_name.git
```

По команде git remote -v получим список репозиториев:
```
github git@github.com:your_name/repository_name.git (fetch)
github git@github.com:your_name/repository_name.git (push)
gitlab git@gitlab.com:your_name/repository_name.git (fetch)
gitlab git@gitlab.com:your_name/repository_name.git (push)
bitbucket git@bitbucket.org:your_name/repository_name.git (fetch)
bitbucket git@bitbucket.org:your_name/repository_name.git (push)
```

Для отправки кода в репозиторий необходимо указать его имя:
```
git push github
git push gitlab
git push bitbucket
```

## Замена репозитория по умолчанию
Любой из репозиториев можно назвать origin, тогда он будет репозиторием по умолчанию. Также можно заменить текущий удалённый репозиторий:
`git remote set-url <remote_name> <remote_url>`

Например:
`git remote set-url origin https://github.com/your_name/repository_name.git`