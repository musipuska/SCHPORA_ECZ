# Шпаргалка к экзамену квалификационному (ПМ.04) — настройка Ubuntu Linux

> **Важно про структуру билетов.** Изучил все 29 вариантов — у них разная «легенда» (геймдев, DevOps, банк, IoT, медицина, SCADA и т.д.), но **критерии оценки, перечень категорий ПО и структура задания везде одинаковые** (40 баллов, 80 минут). Меняется только тематика ПО, которое нужно обосновать и установить. Поэтому эта шпаргалка закрывает **любой** вариант билета — нужно только под свою легенду подставить конкретные названия программ (см. таблицу в разделе 3.6).

Экзамен = 4 блока критериев:
1. Настройка ОС, драйверов, служб, сети, установка базового ПО и виртуального принтера
2. Защита компьютерных систем (бэкап, образы, точки восстановления, пользователи, права, аутентификация, журналы)
3. Установка и настройка прикладного ПО + совместимость
4. Документирование ПО (руководство пользователя по ГОСТ Р 59795-2021)

---

## 0. Быстрый чек-лист перед сдачей (порядок действий на экзамене)

```bash
# 1. Обновить систему в начале — покажет, что вы "живой" пользователь терминала
sudo apt update && sudo apt upgrade -y

# 2. Сеть — сразу проверить
ip a
ping -c 4 8.8.8.8

# 3. SSH — включить и проверить
sudo systemctl enable --now ssh
sudo systemctl status ssh

# 4. Далее по пунктам ниже
```

Держите в голове порядок баллов (из критериев): SSH (2), удалённый доступ к сессии (2), сетевой интерфейс (1), проверка сети (2), базовое ПО (2), принтер (2), обоснование (2) = блок 1 (13 б.); бэкап (2), образ (1), точки восстановления (2), группы (1), права (1), аутентификация (1), журнал (1) = блок 2 (9 б.); установка/настройка ПО (2.5+2+2), совместимость (5×0.5) = блок 3 (9 б.); документация (2.5+4) = блок 4 (6.5 б.). Не забывайте **проговаривать вслух**, что делаете — это часть демонстрации.

---

## 1. Настройка ОС, драйверов, служб, сети

### 1.1 Специальные утилиты для настройки ядра (2,5 балла)

Утилиты для просмотра/настройки параметров ядра и оборудования:

```bash
# Просмотр информации о ядре и системе
uname -a                     # версия ядра, архитектура
lsb_release -a               # версия дистрибутива
hostnamectl                  # общая информация о системе

# Работа с параметрами ядра через sysctl (runtime-параметры)
sysctl -a | less             # все текущие параметры ядра
sudo sysctl -w net.ipv4.ip_forward=1        # изменить параметр "на лету"
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf  # сохранить навсегда
sudo sysctl -p                # применить изменения из /etc/sysctl.conf

# Модули ядра
lsmod                         # список загруженных модулей
sudo modprobe <module>        # загрузить модуль
sudo modprobe -r <module>     # выгрузить модуль
lspci -k                      # PCI-устройства + драйверы
lsusb                         # USB-устройства

# Утилиты диагностики/мониторинга оборудования (аналоги CPU-Z/GPU-Z/HWinfo из билета)
sudo apt install -y inxi hardinfo lm-sensors
sudo sensors-detect --auto
sensors                       # температура, напряжения
inxi -Fxz                     # полная сводка по железу (аналог HWinfo)
```

**Пояснение:** `sysctl` — основной инструмент "тонкой настройки ядра" в Linux, именно на него ссылается критерий "использованы специальные утилиты для настройки ядра". Изменения через `-w` временные, через `/etc/sysctl.conf` — постоянные.

### 1.2 Настройка сетевого протокола SSH (2 балла)

```bash
# Установка сервера SSH
sudo apt update
sudo apt install -y openssh-server

# Запуск и автозапуск
sudo systemctl enable --now ssh
sudo systemctl status ssh

# Проверка порта
sudo ss -tlnp | grep :22

# Базовая настройка безопасности SSH
sudo nano /etc/ssh/sshd_config
```

Ключевые директивы в `/etc/ssh/sshd_config`, которые стоит показать/изменить:
```
Port 22                       # можно сменить для безопасности
PermitRootLogin no            # запрет входа под root
PasswordAuthentication yes    # или no, если настраиваете вход по ключу
PubkeyAuthentication yes
```

```bash
sudo systemctl restart ssh    # применить изменения

# Настройка входа по ключу (демонстрация "продвинутой" настройки SSH)
ssh-keygen -t ed25519 -C "exam@ubuntu"
ssh-copy-id user@remote_host          # копирование публичного ключа на удалённый хост

# Проверка подключения (с другого терминала/машины)
ssh user@localhost
```

### 1.3 Удалённый доступ к активной сессии (2 балла)

Это ключевой пункт: нужно продемонстрировать доступ **не просто по SSH**, а именно к **активной графической/терминальной сессии**. Варианты:

**Вариант A — графический удалённый рабочий стол (xrdp, совместим с RDP-клиентами Windows):**
```bash
sudo apt install -y xrdp
sudo systemctl enable --now xrdp
sudo adduser xrdp ssl-cert     # права на сертификат
sudo systemctl status xrdp
sudo ss -tlnp | grep 3389      # проверка порта RDP
```

**Вариант B — VNC-сервер (доступ к текущему рабочему столу):**
```bash
sudo apt install -y tigervnc-standalone-server tigervnc-common
vncserver :1                    # запуск сессии, задать пароль при первом запуске
vncserver -list                 # список активных сессий
# подключение с клиента: <IP>:5901
```

**Вариант C — "прицепиться" к активной терминальной сессии (screen/tmux):**
```bash
# создать именованную сессию
tmux new -s work
# отцепиться: Ctrl+B, затем D
# найти и подключиться к активной сессии с другого терминала/по SSH
tmux ls
tmux attach -t work

# аналог на screen
screen -S work
screen -ls
screen -r work
```

> Для экзамена обычно достаточно **xrdp** или **VNC** — это буквально "удалённый доступ к активной сессии" в терминах критерия.

### 1.4 Настройка интернет-соединения (1 + 2 балла)

```bash
# Выбор правильного сетевого интерфейса
ip a                              # список всех интерфейсов и их состояние
ip link show                      # состояние интерфейсов (UP/DOWN)
nmcli device status               # список устройств через NetworkManager
nmcli connection show             # список сетевых подключений

# Включить интерфейс
sudo ip link set eth0 up
sudo nmcli device connect eth0

# Настройка IP (статический адрес) через nmcli
sudo nmcli connection modify eth0 ipv4.addresses 192.168.1.100/24
sudo nmcli connection modify eth0 ipv4.gateway 192.168.1.1
sudo nmcli connection modify eth0 ipv4.dns "8.8.8.8 8.8.4.4"
sudo nmcli connection modify eth0 ipv4.method manual
sudo nmcli connection up eth0

# Настройка через netplan (актуально для Ubuntu Server)
sudo nano /etc/netplan/00-installer-config.yaml
```
Пример конфигурации netplan:
```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
```
```bash
sudo netplan apply

# Проверка сетевого соединения (демонстрация для критерия!)
ping -c 4 ya.ru
ping -c 4 8.8.8.8
traceroute ya.ru
curl -I https://ya.ru
dig ya.ru                       # проверка DNS
nslookup ya.ru
ip route                        # таблица маршрутизации
```

**Пояснение:** покажите на экзамене и `ip a` (какой интерфейс активен и какой у него IP), и `ping`/`curl` (что интернет реально работает) — это два разных подкритерия.

### 1.5 Установка базового программного обеспечения (2 балла)

```bash
sudo apt update && sudo apt upgrade -y

# "Базовое" ПО — минимальный набор для работы
sudo apt install -y curl wget git nano vim htop tree unzip build-essential \
    software-properties-common apt-transport-https ca-certificates gnupg lsb-release

# Проверка установки
dpkg -l | grep <package>
which <command>
```

### 1.6 Установка виртуального принтера (2 балла)

```bash
# Установка системы печати CUPS + виртуальный PDF-принтер
sudo apt install -y cups cups-pdf printer-driver-cups-pdf

# Запуск службы
sudo systemctl enable --now cups
sudo systemctl status cups

# Добавить текущего пользователя в группу lpadmin
sudo usermod -aG lpadmin $USER

# Проверка списка принтеров
lpstat -p -d
lpinfo -v

# Веб-интерфейс администрирования CUPS (продемонстрировать)
# http://localhost:631

# Тестовая печать в PDF (виртуальный принтер "PDF")
echo "Test print" > test.txt
lp -d PDF test.txt
ls ~/PDF/                      # cups-pdf по умолчанию сохраняет туда
```

**Пояснение:** `cups-pdf` создаёт виртуальный принтер с именем `PDF`, печать на который сохраняет файл как PDF-документ — это стандартный способ продемонстрировать "виртуальный принтер" в Linux.

### 1.7 Обоснование выбора программных ресурсов (2 балла)

Это не команда, а **устный/письменный** пункт — во время демонстрации проговорите для каждой установленной программы: почему выбрана именно она (открытый исходный код / бесплатна / совместима с Linux / поддерживает нужный стек технологий из легенды билета) и подтвердите документально (в "Руководстве пользователю", см. раздел 4).

---

## 2. Реализация защиты компьютерных систем

### 2.1 Резервное копирование установленной ОС (2 балла)

**Вариант A — Timeshift (рекомендуется, есть GUI и CLI, стандарт для Ubuntu):**
```bash
sudo apt install -y timeshift
sudo timeshift --create --comments "Backup before exam" --tags D
sudo timeshift --list                    # список снапшотов
```

**Вариант B — rsync (копирование файловой системы):**
```bash
sudo rsync -aAXv --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"} \
    / /backup/system_backup/
```

**Вариант C — tar (архив всей системы):**
```bash
sudo tar -cvpzf backup.tar.gz --exclude=/backup.tar.gz \
    --exclude=/proc --exclude=/tmp --exclude=/mnt --exclude=/dev \
    --exclude=/sys --exclude=/run --one-file-system /
```

### 2.2 Создание установочного образа системы (1 балл)

```bash
# Вариант A — dd (посекторный образ диска, "живой" образ системы)
sudo dd if=/dev/sda of=/backup/system_image.img bs=4M status=progress

# Вариант B — Clonezilla (специализированный инструмент клонирования)
sudo apt install -y clonezilla
# запуск: sudo ocs-sr -q2 -c -j2 -z1p -i 4096 -sc -p true savedisk <image_name> sda

# Вариант C — squashfs (образ "только для чтения", как LiveCD)
sudo mksquashfs / /backup/system.squashfs -e /backup -e /proc -e /tmp -e /sys
```

### 2.3 Точки восстановления системы (2 балла)

```bash
# Timeshift — самый явный аналог "точек восстановления Windows" для Linux
sudo timeshift --create --comments "Restore point 1"
sudo timeshift --list
# восстановление:
sudo timeshift --restore --snapshot '2026-07-01_12-00-00'

# Автоматизация — настроить регулярные точки восстановления по расписанию (через GUI Timeshift или cron)
sudo timeshift --create --scheduled --comments "Auto snapshot"
```

### 2.4 Создание групп пользователей и прав доступа (1 + 1 балл)

```bash
# Создание групп
sudo groupadd developers
sudo groupadd testers
sudo groupadd admins

# Создание пользователей с привязкой к группе
sudo useradd -m -G developers -s /bin/bash ivanov
sudo passwd ivanov

# Добавление существующего пользователя в группу
sudo usermod -aG developers petrov

# Просмотр групп
groups ivanov
cat /etc/group | grep developers
getent group developers

# Настройка прав доступа (соответствие набору разрешённых действий!)
sudo chown ivanov:developers /opt/project
sudo chmod 750 /opt/project              # rwx для владельца и группы, ничего для остальных
sudo chmod -R g+rw /opt/project/shared   # группе — чтение/запись

# ACL для более гибких прав (если недостаточно chmod)
sudo apt install -y acl
sudo setfacl -m u:petrov:rwx /opt/project
getfacl /opt/project

# Настройка sudo-прав (разграничение админских полномочий)
sudo visudo
# добавить строку: ivanov ALL=(ALL) NOPASSWD: /usr/bin/apt update
```

### 2.5 Настройка аутентификации и авторизации (1 балл)

```bash
# Политика паролей (PAM)
sudo apt install -y libpam-pwquality
sudo nano /etc/security/pwquality.conf
# minlen=10
# dcredit=-1
# ucredit=-1

# Блокировка после неудачных попыток входа
sudo nano /etc/pam.d/common-auth
# добавить: auth required pam_tally2.so deny=5 unlock_time=900

# Двухфакторная аутентификация (SSH + Google Authenticator) — хороший балл за "продвинутую" настройку
sudo apt install -y libpam-google-authenticator
google-authenticator
sudo nano /etc/pam.d/sshd        # добавить: auth required pam_google_authenticator.so
sudo nano /etc/ssh/sshd_config   # ChallengeResponseAuthentication yes
sudo systemctl restart ssh

# Настройка авторизации через sudo-группу
sudo usermod -aG sudo ivanov
```

### 2.6 Настройка журнала мониторинга (1 балл)

```bash
# systemd journal — встроенный журнал событий
journalctl -xe                    # последние события с пояснениями
journalctl -u ssh                 # журнал конкретной службы
journalctl --since "1 hour ago"
journalctl -f                     # "живой" просмотр (аналог tail -f)

# rsyslog — классический системный журнал
sudo apt install -y rsyslog
sudo systemctl enable --now rsyslog
tail -f /var/log/syslog
tail -f /var/log/auth.log         # журнал попыток входа/аутентификации

# auditd — детальный аудит действий (журнал безопасности)
sudo apt install -y auditd audispd-plugins
sudo systemctl enable --now auditd
sudo auditctl -w /etc/passwd -p wa -k passwd_changes    # отслеживать изменения файла
sudo ausearch -k passwd_changes
sudo aureport

# Ротация логов, чтобы не переполнялся диск
sudo nano /etc/logrotate.conf
```

---

## 3. Установка, настройка и конфигурирование ПО

### 3.1 Общие способы установки ПО в Ubuntu

```bash
# APT (основной пакетный менеджер Debian/Ubuntu)
sudo apt update
sudo apt install -y <package>
apt search <keyword>
apt show <package>
sudo apt remove <package>
sudo apt autoremove

# Snap (универсальные пакеты, много IDE ставятся именно так)
sudo snap install <package>
sudo snap install <package> --classic     # для IDE обычно нужен --classic
snap list

# Flatpak (альтернатива snap)
sudo apt install -y flatpak
flatpak install flathub <app_id>

# Установка .deb пакета вручную (если скачан файл)
sudo dpkg -i package.deb
sudo apt install -f                       # доустановить зависимости при ошибке

# Добавление стороннего репозитория (PPA) — часто нужно для новых версий ПО
sudo add-apt-repository ppa:<repo>
sudo apt update
```

### 3.2 Настройка обмена данными с другими системами

```bash
# Firewall — разрешение/ограничение соединений между системами
sudo ufw status
sudo ufw allow 22/tcp        # SSH
sudo ufw allow 5432/tcp      # PostgreSQL, например
sudo ufw enable

# Проброс портов, если ПО работает в контейнере/VM
sudo ss -tlnp                # проверка занятых портов
```

### 3.3 Компилятор / SDK (общий принцип для любой "среды разработки" из легенды)

```bash
# Пример: универсальный набор компиляторов (упоминается в билетах как "изменённая версия набора компиляторов")
sudo apt install -y build-essential gcc g++ gdb cmake make

# Проверка версии
gcc --version
g++ --version
```

### 3.4 СУБД — быстрая установка (для критерия "СУБД")

```bash
# PostgreSQL
sudo apt install -y postgresql postgresql-contrib
sudo systemctl enable --now postgresql
sudo -u postgres psql -c "\l"

# MySQL
sudo apt install -y mysql-server
sudo systemctl enable --now mysql
sudo mysql_secure_installation

# MongoDB (NoSQL — для веб/чат-бот сценариев)
sudo apt install -y mongodb
sudo systemctl enable --now mongodb

# SQLite (лёгкая встраиваемая СУБД)
sudo apt install -y sqlite3
sqlite3 test.db
```

### 3.5 Виртуализация / контейнеризация / эмуляторы (частая тема многих легенд)

```bash
# Docker — контейнеризация
sudo apt install -y docker.io docker-compose
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
docker --version

# VirtualBox / QEMU-KVM — эмуляторы ОС ("программа-эмулятор, создающая среду разных ОС")
sudo apt install -y virtualbox
# или
sudo apt install -y qemu-kvm libvirt-daemon-system virt-manager
sudo usermod -aG libvirt,kvm $USER

# Android Studio (эмулятор Android)
sudo snap install android-studio --classic
```

### 3.6 Категории ПО из «Перечня ресурсов» — что ставить под свою легенду

Во всех билетах одинаковый список категорий, но конкретное ПО подбирается под вашу легенду. Держите под рукой:

| Категория (из билета) | Примеры для установки в Ubuntu |
|---|---|
| Графический редактор | `sudo apt install -y gimp inkscape` |
| Офисный пакет | `sudo apt install -y libreoffice` |
| Архиватор | `sudo apt install -y p7zip-full peazip` |
| Утилита диагностики | `sudo apt install -y hardinfo inxi` |
| Антивирус | `sudo apt install -y clamav clamtk` (+ `sudo freshclam` для обновления баз) |
| IDE / среда разработки | `code` (VS Code, см. ниже), `sudo snap install pycharm-community --classic`, `sudo snap install android-studio --classic`, `sudo apt install -y netbeans codeblocks` |
| СУБД | см. п. 3.4 |

**Установка VS Code (популярна почти для всех легенд — Python/JS/DevOps):**
```bash
sudo apt install -y wget gpg
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
sudo install -D -o root -g root -m 644 packages.microsoft.gpg /etc/apt/keyrings/packages.microsoft.gpg
echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" | sudo tee /etc/apt/sources.list.d/vscode.list
sudo apt update
sudo apt install -y code
```

**Антивирус ClamAV — базовая настройка (для критерия "защита от вредоносного ПО"):**
```bash
sudo apt install -y clamav clamav-daemon
sudo systemctl stop clamav-freshclam
sudo freshclam                       # обновить базы сигнатур
sudo systemctl start clamav-freshclam
clamscan -r /home                    # сканирование каталога
```

### 3.7 Настройка параметров совместимости ПО с ОС (5 подкритериев × 0,5 балла)

Это отдельный явный блок критериев — не пропустите на экзамене, обычно про него забывают:

```bash
# 1) Работа с ограниченной цветовой палитрой
xdpyinfo | grep "depth"              # посмотреть текущую глубину цвета
# запуск X-сессии с ограниченной палитрой (демонстрация):
startx -- -depth 8

# 2) Работа с низким разрешением экрана
xrandr                                # список доступных режимов
xrandr --newmode "800x600_60.00" 38.25 800 832 912 1024 600 603 607 624 -hsync +vsync
xrandr --addmode Virtual1 "800x600_60.00"
xrandr --output Virtual1 --mode 800x600

# 3) Решение проблем с отображением меню и кнопок (актуально для старых Java/GTK-приложений)
export GDK_BACKEND=x11                # форс X11 вместо Wayland — частая причина "битых" меню
export _JAVA_OPTIONS="-Dsun.java2d.uiScale=1"   # для Java-приложений с некорректным UI
sudo apt install -y gtk2-engines-murrine          # исправление тем GTK

# 4) Отключение композиции рабочего стола (desktop compositing)
# GNOME:
gsettings set org.gnome.desktop.interface enable-animations false
# для полного отключения composite-менеджера в GNOME (через Xorg, если используется):
# Settings → Windows → "Disable window compositing" либо:
busctl --user call org.gnome.Mutter.DisplayConfig /org/gnome/Mutter/DisplayConfig org.gnome.Mutter.DisplayConfig GetResources
# XFCE:
xfconf-query -c xfwm4 -p /general/use_compositing -s false
# KDE Plasma:
kwriteconfig5 --file kwinrc --group Compositing --key Enabled false && qdbus org.kde.KWin /Compositor suspend

# 5) Отключение масштабирования изображения при высоком разрешении (HiDPI)
gsettings set org.gnome.desktop.interface scaling-factor 1
xrandr --output eDP-1 --scale 1x1
```

**Пояснение:** этот блок — про "старое"/специфическое ПО, которое плохо работает на современных экранах (HiDPI-мониторы, Wayland). Экзаменатор проверяет понимание типичных проблем совместимости legacy-приложений с современным Linux-десктопом.

---

## 4. Документирование программного обеспечения

Это не про терминал, но не забудьте про эти 6,5 баллов:

- **Документация пользователя** (2,5 балла): краткое описание программы, её назначение, способ запуска, штатный выход из программы.
- **Руководство по использованию ПО** (4 балла), оформленное **по ГОСТ Р 59795-2021**: перечень выполняемых функций программы + основные приёмы работы (пошагово, со скриншотами по возможности).

Быстро получить справочную информацию для документации прямо из терминала:
```bash
man <program>                 # штатная документация Unix
<program> --help
dpkg -L <package>             # список файлов пакета (для описания структуры)
dpkg -s <package>              # описание пакета, версия, зависимости
```

Оформляйте руководство как отдельный текстовый/офисный документ (LibreOffice Writer) — структура по ГОСТ обычно требует: титульный лист → назначение → условия применения → подготовка к работе → описание операций (функций) → аварийные ситуации → рекомендации по освоению.

---

## 5. Частые ошибки и на что обратить внимание

1. **Не забывайте `sudo`** — большинство команд настройки системы требуют прав root.
2. Перед экзаменом сделайте `sudo apt update` — без интернета часть пунктов (SSH, антивирус, IDE) не поставится.
3. Не забывайте **проверять** результат действия (не просто настроить SSH, а показать `systemctl status ssh` / попробовать подключиться) — по критериям баллы часто идут именно за "продемонстрировано", а не просто "выполнено".
4. Блок "совместимости ПО" (5 подпунктов по 0,5 балла) — маленький по весу, но про него легко забыть целиком, потеряв 2,5 балла.
5. Обоснование выбора ПО и документация — тоже часть оценки, не только команды в терминале.
6. Следите за временем — 80 минут на весь билет, поэтому имеет смысл заранее выучить команды из разделов 1–2 наизусть (они одинаковы во всех вариантах), чтобы больше времени осталось на специфичное для вашей легенды ПО (раздел 3) и документ (раздел 4).
