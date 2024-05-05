# Подготовка Git
> Начальная настройка Git для работы с репозиториями.
## 1. Установка имени и электронной почты
Если вы никогда ранее не использовали git, для начала вам необходимо осуществить установку. Выполните следующие команды, чтобы git узнал ваше имя и электронную почту.
`git config --global user.name "Your Name"`
`git config --global user.email "your_email@whatever.com"`

## 2. Параметры установки окончаний строк
Для пользователей Unix/Mac:
`git config --global core.autocrlf input`
`git config --global core.safecrlf warn`

Для пользователей Windows:
`git config --global core.autocrlf true`
`git config --global core.safecrlf warn`

## 3. Установка отображения unicode в выводе git
По умолчанию, git будет печатать не-ASCII символы в именах файлов в виде восьмеричных последовательностей \nnn. Чтобы избежать нечитаемых строк, установите соответствующий флаг:
`git config --global core.quotepath off`
