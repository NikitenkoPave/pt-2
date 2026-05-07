# ПР №4. Аудит событий: journalctl и auditd

## 1. journalctl — системный журнал

### Что зафиксировал журнал о команде sudo cat /etc/shadow

May 07 12:41:15 denzzy sudo[2154]: denzzy : TTY=pts/0 ; PWD=/home/denzzy ; USER=root ; COMMAND=/usr/bin/cat /etc/shadow

Разбор полей:
- May 07 12:41:15 — дата и время события
- denzzy — имя компьютера
- sudo[2154] — процесс sudo и его PID
- denzzy — пользователь выполнивший команду
- TTY=pts/0 — терминал
- PWD=/home/denzzy — текущая директория
- USER=root — команда выполнена от root
- COMMAND=/usr/bin/cat /etc/shadow — выполненная команда

### Ошибки в системе с последней загрузки

В журнале были обнаружены:
- ошибки авторизации SSH
- сообщения об отказе доступа
- предупреждения auditd
- ошибки systemd при запуске сервисов

Команда для поиска:
journalctl -p err -b

### Статистика входов

Успешных входов: 3
Неудачных попыток: 2

Команды:
journalctl _COMM=sshd | grep "Accepted"
journalctl _COMM=sshd | grep "Failed"

## 2. auditd — настройка правил

### Применённые правила (вывод auditctl -l)

-w /etc/passwd -p rwa -k auth-files
-w /etc/shadow -p rwa -k auth-files
-w /etc/group -p rwa -k auth-files
-w /etc/sudoers -p rwa -k priv-files
-w /etc/sudoers.d/ -p rwa -k priv-files
-w /etc/ssh/sshd_config -p rwa -k ssh-config
-a always,exit -F arch=b64 -S open -S openat -F success=0 -k access-denied
-a always,exit -F arch=b64 -S unlink -S unlinkat -k file-delete
-a always,exit -F arch=b64 -S execve -F euid=0 -F auid!=0 -F auid!=-1 -k priv-exec

### Разбор записи SYSCALL

auid = 1000 — пользователь вошедший в систему
uid = 1000 — текущий UID процесса
euid = 0 — эффективный UID root
comm = cat — имя выполненной команды
exe = /usr/bin/cat — путь к исполняемому файлу

Дополнительно:
type=SYSCALL — запись системного вызова ядра Linux
success=no — операция завершилась ошибкой
syscall=openat — попытка открытия файла

## 3. Расследование инцидента

### Хронология действий bob

13:10:22 — cat /etc/shadow — отказ — ausearch -k access-denied
13:10:35 — ls /root — отказ — ausearch -k access-denied
13:10:48 — cat /etc/passwd — успешно — ausearch -k auth-files
13:11:01 — touch /tmp/totally-not-malware.sh — файл создан — ausearch --start recent
13:11:15 — cat /etc/sudoers — отказ — ausearch -k access-denied

### Как auditd помог расследованию

auditd позволил определить:
- кто выполнял подозрительные действия
- время выполнения операций
- попытки доступа к защищённым файлам
- создание подозрительного файла
- системные вызовы связанные с инцидентом

Благодаря полю auid удалось установить исходного пользователя даже при использовании sudo.

### Чем auditd лучше journalctl для задач аудита безопасности

journalctl:
- хранит системные логи
- логи сервисов и приложений
- меньше детализация
- удобен для диагностики

auditd:
- выполняет аудит безопасности
- контролирует системные вызовы
- предоставляет более подробную информацию
- удобен для расследования инцидентов

auditd работает на уровне ядра Linux и его сложнее обойти злоумышленнику.

## 4. Связь с нормативкой

АУД.4 Аудит безопасности — настроен сбор и анализ событий безопасности
АУД.7 Мониторинг НСД — зафиксированы попытки несанкционированного доступа к /etc/shadow и /etc/sudoers
