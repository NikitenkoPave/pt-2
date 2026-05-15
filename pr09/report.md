# ПР №9. Следы вредоносного ПО в Linux

## 1. Что было посажено

| Механизм       | Место                              | Команда/файл                              |
|----------------|------------------------------------|-------------------------------------------|
| Cron           | crontab пользователя               | /tmp/.hidden_malware/backdoor.sh (@reboot + */5) |
| Systemd        | ~/.config/systemd/user/            | system-helper.service                     |
| Shell profile  | ~/.bashrc                          | /tmp/.hidden_malware/backdoor.sh          |
| Процесс        | /tmp/.hidden_malware/              | listener.sh (nc на порту 4444)            |

## 2. Что нашли — процессы

Команда: ps aux | grep '/tmp'

Результат: 
- denzzy  3903 ... /bin/bash /tmp/.hidden_malware/backdoor.sh 
- denzzy  3925 ... /bin/bash /tmp/.hidden_malware/listener.sh 
- denzzy  3926 ... nc -l -p 4444 ...

Что подозрительно: 
Процессы запущены из скрытой папки /tmp/.hidden_malware/. Имена скриптов (backdoor.sh, listener.sh) и расположение типичны для вредоносного ПО. Один процесс держит backdoor-порт.

Команда: ps auxf | grep -A3 -B3 'hidden'

Дерево процессов показывает, что backdoor.sh и listener.sh запущены напрямую пользователем.

## 3. Что нашли — сетевые соединения

Команда: ss -tulnp

Подозрительный порт: 4444 (TCP LISTEN) 
Процесс: nc (PID 3926)

Команда: sudo lsof -i :4444

Результат: 
COMMAND  PID   USER   FD   TYPE  ...  NAME
nc      3926  denzzy  3u  IPv4  ...  TCP *:4444 (LISTEN)

Как lsof связывает порт с процессом: 
lsof напрямую показывает, какой процесс (nc, PID 3926) и какой файл (/tmp/.hidden_malware/listener.sh) держит сокет. Это позволяет быстро перейти от сетевой активности к исполняемому файлу.

## 4. Что нашли — автозапуск

### Cron
Результат crontab -l: 
@reboot /tmp/.hidden_malware/backdoor.sh &
*/5 * * * * /tmp/.hidden_malware/backdoor.sh &

Что подозрительно: Запуск каждые 5 минут и при перезагрузке из скрытой директории /tmp/.

### Systemd
Результат systemctl --user list-unit-files: 
system-helper.service — enabled.

Содержимое unit-файла:
[Unit]
Description=System Helper Service
After=default.target

[Service]
ExecStart=/tmp/.hidden_malware/backdoor.sh
Restart=always
RestartSec=10

[Install]
WantedBy=default.target

Что подозрительно: 
- Описание слишком общее («System Helper Service»)
- ExecStart указывает на /tmp/.hidden_malware)
- Сервис пользовательский (не требует root).

### ~/.bashrc
Строка которую нашли: 
# system update helper  
/tmp/.hidden_malware/backdoor.sh &

Где находится: в конце файла (добавлено вручную).

## 5. Итоговая таблица следов

| Место                  | Инструмент обнаружения     | Что нашли                                      |
|------------------------|----------------------------|------------------------------------------------|
| Процессы               | ps aux                     | backdoor.sh, listener.sh из /tmp               |
| Порт 4444              | ss -tulnp                  | listener.sh (nc)                               |
| Файлы процесса         | lsof -p PID / lsof -i      | activity.log, listener.sh, backdoor.sh         |
| Cron                   | crontab -l                 | @reboot и */5 записи                           |
| Systemd                | systemctl --user           | system-helper.service                          |
| Bashrc                 | tail / grep                | строка запуска backdoor                        |

## 6. Связь с нормативкой

Какие меры ФСТЭК №17 реализует эта проверка:
- АНЗ.2 — обнаружение вредоносного кода (поиск по автозапуску, процессам, файлам)
- АУД.4 — анализ безопасности (использование ps, ss, lsof)
- ЗИС.17 — управление сетевыми соединениями (мониторинг listening-портов)
- АУД.7 — мониторинг событий НСД (выявление аномалий в процессах и соединениях)


