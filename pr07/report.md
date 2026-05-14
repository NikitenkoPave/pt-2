# ПР №7. AppArmor, Capabilities и Docker

## 1. Linux Capabilities

### Разбор getcap /usr/bin/ping
`cap_net_raw=ep` 
- `cap_net_raw` — позволяет работать с raw-сокетами (нужен для ping). 
- `e` (effective) — capability активно применяется. 
- `p` (permitted) — capability разрешена процессу.

### CapPrm / CapEff / CapBnd — в чём разница
- CapPrm (Permitted) — capabilities, которые процесс может использовать.
- CapEff (Effective) — capabilities, которые сейчас действуют.
- CapBnd (Bounding) — максимальный набор, который нельзя превысить даже root'у.

### setcap — демонстрация
До выдачи capability:
- Порт 80 → `DENIED` (PermissionError) 
- Порт 8080 → `OK`

После `sudo setcap cap_net_bind_service=ep`: 
- Порт 80 → `OK: привязался к порту 80`

Почему лучше чем sudo?
Процесс получает только одну нужную capability, а не все права root. При взломе ущерб сильно ограничен.

### Флаги e, i, p в cap_net_raw+eip
- `e` — Effective (действует сейчас)
- `i` — Inheritable (наследуется дочерними процессами)
- `p` — Permitted (разрешено)

## 2. AppArmor

### Количество профилей
enforce: ~50–70 
complain: 0–10

### Результаты pr07-reader

| Действие                        | Без профиля | complain       | enforce     |
|-------------------------------|-------------|----------------|-------------|
| Читать /tmp/pr07-allowed.txt  | OK          | OK             | OK          |
| Читать /etc/shadow            | OK          | OK (логирует)  | DENIED      |
| Писать в /tmp/pr07-output.txt | OK          | OK             | OK          |
| Писать в /etc/                | OK          | OK (логирует)  | DENIED      |

### Разбор строки DENIED
apparmor="DENIED" operation="open" profile="/usr/local/bin/pr07-reader" name="/etc/shadow" pid=... comm="cat" requested_mask="r" denied_mask="r"
text- `operation="open"` — операция открытия файла 
- `profile=...` — какой профиль применился 
- `name="/etc/shadow"` — к какому файлу был доступ 
- `denied_mask="r"` — запрещено чтение

## 3. Docker — изоляция

| Ресурс                  | Хост                    | Контейнер                     |
|-------------------------|-------------------------|-------------------------------|
| Количество процессов    | 150–400                 | 1–3                           |
| Сетевые интерфейсы      | eth0, lo, docker0 и др. | eth0, lo                      |
| Корневая ФС             | Полная система хоста    | Изолированная                 |
| /etc/shadow хоста       | доступен                | недоступен                    |

### Capabilities: обычный vs --privileged
Обычный контейнер — CapEff сильно урезан. 
--privileged — почти полный набор capabilities (как root на хосте).

Почему --privileged опасен?
При взломе контейнера атакующий получает практически полный контроль над хостовой системой.

### Итоговый nginx
Запущен с минимальными capabilities:
- `NET_BIND_SERVICE` — для бинда на 80 порт
- `CHOWN`, `DAC_OVERRIDE`, `SETGID`, `SETUID` — для работы nginx

## 4. Эшелонированная защита

| Слой         | Инструмент              | Что ограничивает                              |
|--------------|-------------------------|-----------------------------------------------|
| DAC          | chmod/chown             | Права доступа по владельцу                    |
| Capabilities | --cap-drop / --cap-add  | Гранулярные привилегии вместо root            |
| MAC          | AppArmor                | Мандатный контроль по профилям                |
| Изоляция     | Docker namespaces+cgroups | Полная изоляция процессов, сети, файловой системы |
