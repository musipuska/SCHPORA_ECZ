# one_more.md — подтверждённые команды для экзамена

> Ниже — команды, которые точно будут спрашиваться (подтверждено), плюс минимально необходимые соседние пункты по тем же критериям, чтобы не потерять баллы рядом.

---

## 1. Проверка сетевого соединения

```bash
ping google.com
# или просто открыть браузер и зайти на любой сайт
```

Дополнительно можно показать выбор интерфейса (соседний подкритерий, 1 балл):
```bash
ip a
```

---

## 2. Установка базового ПО

```bash
sudo apt install -y libreoffice
sudo apt install -y gimp
sudo apt install -y p7zip-full p7zip-rar
```

---

## 3. Установка виртуального принтера

```bash
sudo apt install cups-pdf
```

Проверить, что принтер появился (на всякий случай, если попросят продемонстрировать):
```bash
lpstat -p -d
```

---

## 4. Резервное копирование ОС

```bash
sudo tar -cvpzf /home/student/system-backup.tar.gz \
    --exclude=/proc --exclude=/sys --exclude=/dev --exclude=/run /
```

---

## 5. Точки восстановления системы

```bash
sudo apt install -y timeshift
sudo timeshift --create --comment "Точка восстановления"
```

Проверить список созданных точек:
```bash
sudo timeshift --list
```

---

## 6. Группы пользователей

```bash
sudo groupadd developers
sudo useradd -m -G developers ivanov
sudo passwd ivanov
sudo usermod -aG developers petrov

# проверка
groups ivanov
cat /etc/group | grep developers
```

Права доступа (соседний подкритерий):
```bash
sudo chown ivanov:developers /opt/project
sudo chmod 750 /opt/project
```

---

## Не забыть рядом (та же группа критериев, но не упомянуто выше)

```bash
# SSH
sudo apt install -y openssh-server
sudo systemctl enable --now ssh

# Удалённый доступ к активной сессии
sudo apt install -y xrdp
sudo systemctl enable --now xrdp

# Аутентификация/авторизация
sudo visudo   # либо usermod -aG sudo ivanov

# Журнал мониторинга
journalctl -xe
tail -f /var/log/auth.log
```
