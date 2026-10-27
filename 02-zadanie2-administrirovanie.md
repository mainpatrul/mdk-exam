# Задание 2 - Организация сетевого администрирования

Отчёт: ФамилияУчастникаЗадание2
Выполняется на отдельном стенде (см. условия ниже).

## Преднастроено на стенде

- контроллер домена Samba DC на BR-SRV (установлен и работает, без пользователей/групп/OU)
- IP-адреса, маски, шлюзы
- NAT
- GRE-туннель
- динамическая маршрутизация
- пользователи sshuser (серверы) и net_admin (роутеры) с админ-правами
- порты SSH на серверах
- DHCP-сервер
- DNS-сервер
- HQ-SRV имеет 2 дополнительных диска по 1 ГБ

---

## 1. Samba DC (BR-SRV)

- [ ] ввести HQ-CLI в домен au-team.irpo
- [ ] создать 5 пользователей: hquser1, hquser2, hquser3, hquser4, hquser5
- [ ] политика паролей: мин. длина 8 символов, блокировка после 5 неудачных попыток
- [ ] создать группу hq, добавить всех 5 пользователей
- [ ] убедиться, что группа hq может логиниться на HQ-CLI
- [ ] группа hq может sudo только cat, grep, id (без пароля), остальное - нет

Пример команд (Samba AD DC):
```bash
samba-tool user create hquser1 P@ssw0rd
samba-tool group add hq
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5
```
Политика паролей:
```bash
samba-tool domain passwordsettings set --min-pwd-length=8
samba-tool domain passwordsettings set --account-lockout-threshold=5
```
Ограниченный sudo для группы (через visudo на HQ-CLI):
```
%hq ALL=(ALL) NOPASSWD: /bin/cat, /bin/grep, /usr/bin/id
```

---

## 2. Файловое хранилище (HQ-SRV) - RAID0

- [ ] создать RAID0 (уровень 0) из двух дисков 1 ГБ
- [ ] имя устройства: md0
- [ ] конфигурация массива в /etc/mdadm.conf (при необходимости)
- [ ] создать и отформатировать раздел в ext4
- [ ] автомонтирование в /raid

```bash
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
mdadm --detail --scan >> /etc/mdadm.conf
mkfs.ext4 /dev/md0
mkdir /raid
echo "/dev/md0 /raid ext4 defaults 0 0" >> /etc/fstab
mount -a
```
> Имена дисков /dev/sdb /dev/sdc - проверить фактические через lsblk перед созданием массива.

---

## 3. NFS-сервер (HQ-SRV)

- [ ] папка общего доступа: /raid/nfs
- [ ] доступ RW только для сети в сторону HQ-CLI
- [ ] на HQ-CLI - автомонтирование в /mnt/nfs
- [ ] параметры сервера - в отчёт

```bash
dnf install nfs-utils -y
mkdir -p /raid/nfs
echo "/raid/nfs <сеть_HQ-CLI>/28(rw,sync,no_root_squash)" >> /etc/exports
systemctl enable --now nfs-server
exportfs -ra
```
На HQ-CLI (autofs):
```bash
dnf install autofs -y
echo "/mnt/nfs -rw,soft <IP_HQ-SRV>:/raid/nfs" >> /etc/auto.direct
# либо через /etc/fstab + systemd.automount
systemctl enable --now autofs
```

---

## 4. chrony (ISP как NTP-сервер)

- [ ] вышестоящий сервер NTP на ISP - на выбор
- [ ] stratum сервера: 5
- [ ] клиенты: HQ-SRV, HQ-CLI, BR-RTR, BR-SRV

На ISP (/etc/chrony.conf):
```
server pool.ntp.org iburst
local stratum 5
allow <сеть офисов>
```
```bash
systemctl restart chronyd
```
На клиентах (/etc/chrony.conf):
```
server <IP ISP> iburst
```
```bash
systemctl restart chronyd
chronyc sources
```

---

## 5. Ansible (BR-SRV)

- [ ] inventory: HQ-SRV, HQ-CLI, HQ-RTR, BR-RTR
- [ ] рабочий каталог: /etc/ansible
- [ ] все машины отвечают pong на ansible ping

```bash
dnf install ansible -y
mkdir -p /etc/ansible
nano /etc/ansible/hosts
```
```ini
[all]
hq-srv ansible_host=<IP>
hq-cli ansible_host=<IP>
hq-rtr ansible_host=<IP>
br-rtr ansible_host=<IP>
```
```bash
ansible all -m ping
```
> Понадобится настроить SSH-ключи (ssh-copy-id) для беспарольного подключения ко всем узлам, включая нестандартный порт 2027 для серверов (прописать в inventory: `ansible_port=2027`, `ansible_user=sshuser`).

---

## 6. Docker-стек (BR-SRV)

- [ ] стек контейнеров: веб-приложение + БД
- [ ] образы site_latest, mariadb_latest (директория docker в Additional.iso)
- [ ] docker-файл: compose.yaml (из Additional.iso)
- [ ] testapp доступен снаружи через порт 8080

```bash
dnf install docker docker-compose -y
systemctl enable --now docker
# смонтировать Additional.iso, скопировать образы и compose.yaml
docker load -i site_latest.tar
docker load -i mariadb_latest.tar
cd <директория с compose.yaml>
docker compose up -d
docker ps
```

---

## 7. Веб-приложение (HQ-SRV) - apache + mariadb

- [ ] веб-сервер: apache
- [ ] СУБД: mariadb
- [ ] файлы приложения + dump.sql (директория web в Additional.iso)
- [ ] импорт схемы/данных в БД webdb
- [ ] пользователь БД: web / P@ssw0rd, права на webdb
- [ ] index.php + logo.png в корень сайта
- [ ] в index.php - верные данные подключения к БД
- [ ] проверить работоспособность

```bash
dnf install httpd mariadb-server -y
systemctl enable --now mariadb httpd
mysql -u root -e "CREATE DATABASE webdb;"
mysql -u root webdb < dump.sql
mysql -u root -e "CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';"
mysql -u root -e "GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';"
mysql -u root -e "FLUSH PRIVILEGES;"
cp index.php logo.png /var/www/html/
# отредактировать index.php - указать хост/пользователь/пароль/базу
systemctl restart httpd
```

---

## 8. Статическая трансляция портов (роутеры)

- [ ] BR-RTR: 8080 -> testapp (BR-SRV)
- [ ] HQ-RTR: 8080 -> веб-приложение (HQ-SRV)
- [ ] HQ-RTR: 2027 -> HQ-SRV:2027 (ssh)
- [ ] BR-RTR: 2027 -> BR-SRV:2027 (ssh)

На HQ-RTR:
```bash
firewall-cmd --zone=trusted --add-forward-port=port=8080:proto=tcp:toport=8080:toaddr=192.168.100.10 --permanent
firewall-cmd --zone=trusted --add-forward-port=port=2027:proto=tcp:toport=2027:toaddr=192.168.100.10 --permanent
firewall-cmd --reload
```
На BR-RTR:
```bash
firewall-cmd --zone=trusted --add-forward-port=port=8080:proto=tcp:toport=8080:toaddr=172.30.100.10 --permanent
firewall-cmd --zone=trusted --add-forward-port=port=2027:proto=tcp:toport=2027:toaddr=172.30.100.10 --permanent
firewall-cmd --reload
```

---

## 9. Яндекс Браузер на HQ-CLI

- [ ] установить удобным способом (репозиторий / .deb / .rpm - зависит от дистрибутива)
- [ ] зафиксировать установку в отчёте

```bash
# пример для rpm-based:
curl -o yandex-browser.rpm https://<ссылка_на_актуальный_пакет>
dnf install ./yandex-browser.rpm -y
```

---

## Приложения к заданию

- Прил_5_ОЗ_КИМ_09.02.06-1-2027.txt

## Заметки

- Инструкция по настройке стенда для техэксперта - отдельный файл (см. приложения к спецификации КИМ)
- Файлы образов (site_latest, mariadb_latest, compose.yaml, dump.sql, index.php, logo.png) - все лежат в Additional.iso, смонтировать перед началом
