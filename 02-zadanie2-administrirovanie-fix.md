# Задание 2 - Организация сетевого администрирования (исправленная версия)

Отчёт: ФамилияУчастникаЗадание2
Выполняется на отдельном стенде (см. условия ниже).
Топология та же, что в Задании 1 (BR-FW между BR-RTR и BR-SRV) - адресация из 00-interfaces-addressing.md.
Перед началом сверить фактические адреса на стенде: `ip a`.

## Что исправлено относительно 02-zadanie2-administrirovanie.md

- п.1: добавлены ввод HQ-CLI в домен, DNS-делегирование на BR-SRV, sssd, sudo через /etc/sudoers.d
- п.2: создаётся раздел на md0 (в задании "создайте и отформатируйте раздел")
- п.3: autofs - добавлена строка в /etc/auto.master (без неё не работает), права на папку
- п.4: allow на ISP - адреса после NAT (172.16.1.0/28, 172.16.2.0/28), а не сети офисов
- п.5: ansible.cfg для работы "без предупреждений", пользователи/порты для каждого узла
- п.6: монтирование ISO, проверка compose.yaml, вариант docker-compose v1
- п.7: добавлены php, php-mysqlnd, php-fpm (без них index.php не работает)
- п.8: HQ-RTR 8080 -> HQ-SRV:**80** (apache), а не 8080; заметка про BR-FW

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

## Адреса (из 00-interfaces-addressing.md)

| Узел | Адрес |
|---|---|
| ISP | 172.16.1.1 (к HQ-RTR), 172.16.2.1 (к BR-RTR) |
| HQ-RTR | 172.16.1.2, 192.168.100.1, 192.168.200.1, tun0 10.10.10.1 |
| BR-RTR | 172.16.2.2, 172.30.200.1, tun0 10.10.10.2 |
| BR-FW | 172.30.200.2, 172.30.100.1 |
| HQ-SRV | 192.168.100.10 |
| BR-SRV | 172.30.100.10 |
| HQ-CLI | 192.168.200.x (DHCP, в DNS - 192.168.200.2) |

## Рекомендуемый порядок

п.4 (время - нужно для Kerberos) -> п.1 -> п.2 -> п.3 -> п.7 -> п.6 -> п.8 -> п.5 -> п.9

---

## 1. Samba DC (BR-SRV) + ввод HQ-CLI в домен

- [ ] ввести HQ-CLI в домен au-team.irpo
- [ ] создать 5 пользователей: hquser1 ... hquser5
- [ ] политика паролей: мин. длина 8 символов, блокировка после 5 неудачных попыток
- [ ] создать группу hq, добавить всех 5 пользователей
- [ ] убедиться, что группа hq может логиниться на HQ-CLI
- [ ] группа hq может sudo только cat, grep, id, остальное - нет

### 1.1 На BR-SRV - пользователи, группа, политика паролей
```bash
for i in 1 2 3 4 5; do samba-tool user add hquser$i P@ssw0rd; done
samba-tool group add hq
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5

samba-tool domain passwordsettings set --min-pwd-length=8 --account-lockout-threshold=5
```
Проверка (в отчёт):
```bash
samba-tool user list
samba-tool group listmembers hq
samba-tool domain passwordsettings show
```

### 1.2 На HQ-SRV - DNS: делегировать служебные зоны AD на BR-SRV
HQ-CLI использует DNS 192.168.100.10 (HQ-SRV, bind с зоной au-team.irpo).
Контроллер домена на BR-SRV с той же зоной, поэтому SRV-записи AD bind не найдёт
и HQ-CLI не увидит домен. Решение - делегировать служебные поддомены на BR-SRV.

В /opt/dns/au-team.irpo добавить (и увеличить serial, напр. 0 -> 1):
```
_msdcs  NS      br-srv.au-team.irpo.
_sites  NS      br-srv.au-team.irpo.
_tcp    NS      br-srv.au-team.irpo.
_udp    NS      br-srv.au-team.irpo.
```
```bash
named-checkconf -z
systemctl restart named
host -t SRV _ldap._tcp.au-team.irpo 192.168.100.10
host -t SRV _kerberos._udp.au-team.irpo 192.168.100.10
```
> Должны вернуться записи с br-srv.au-team.irpo. Запасной вариант - на HQ-CLI
> временно указать DNS 172.30.100.10 (BR-SRV) только на время ввода в домен.

### 1.3 На HQ-CLI - ввод в домен
```bash
dnf install realmd sssd oddjob oddjob-mkhomedir adcli samba-common-tools krb5-workstation -y
realm discover au-team.irpo
realm join -U administrator au-team.irpo
# пароль administrator - тот, что задан при преднастройке стенда (уточнить у техэксперта)
```
> В RED OS есть и графический/скриптовый вариант `join-to-domain` - можно им.

Имена без @домена и домашние каталоги - /etc/sssd/sssd.conf, в секции [domain/au-team.irpo]:
```
use_fully_qualified_names = False
fallback_homedir = /home/%u
```
```bash
authselect select sssd with-mkhomedir --force
systemctl enable --now oddjobd
systemctl restart sssd
```
> `realm permit -g hq` НЕ делать: по умолчанию вход разрешён всем доменным
> пользователям, а в Задании 3 на HQ-CLI должны входить и импортированные пользователи.

### 1.4 На HQ-CLI - ограниченный sudo для группы hq
```bash
visudo -f /etc/sudoers.d/hq
```
```
%hq ALL=(ALL) NOPASSWD: /usr/bin/cat, /usr/bin/grep, /usr/bin/id
```

### 1.5 Проверка (в отчёт)
```bash
realm list
id hquser1                 # должна быть группа hq
su - hquser1
sudo -l                    # только cat, grep, id
sudo id                    # работает
sudo ls /root              # отказ
```
Плюс скриншот входа hquser1 в графическую сессию HQ-CLI.

---

## 2. Файловое хранилище (HQ-SRV) - RAID0

- [ ] RAID0 из двух дисков 1 ГБ, имя устройства md0
- [ ] конфигурация в /etc/mdadm.conf
- [ ] раздел, ext4
- [ ] автомонтирование в /raid

```bash
lsblk                                   # проверить имена дисков (здесь sdb, sdc)
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
mdadm --detail --scan >> /etc/mdadm.conf
cat /etc/mdadm.conf

# раздел на массиве
parted -s /dev/md0 mklabel gpt mkpart primary ext4 0% 100%
# (или fdisk /dev/md0 -> n, p, 1, Enter, Enter, w)
mkfs.ext4 /dev/md0p1

mkdir /raid
echo "/dev/md0p1 /raid ext4 defaults 0 0" >> /etc/fstab
systemctl daemon-reload
mount -a
```
Проверка (в отчёт):
```bash
cat /proc/mdstat
mdadm --detail /dev/md0
lsblk
df -h /raid
```

---

## 3. NFS-сервер (HQ-SRV), автомонтирование на HQ-CLI

- [ ] общая папка /raid/nfs
- [ ] RW только для сети HQ-CLI (192.168.200.0/28)
- [ ] на HQ-CLI автомонтирование в /mnt/nfs
- [ ] параметры сервера - в отчёт

На HQ-SRV:
```bash
dnf install nfs-utils -y
mkdir -p /raid/nfs
chmod 777 /raid/nfs
echo "/raid/nfs 192.168.200.0/28(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
systemctl enable --now nfs-server
exportfs -arv
```
Если на HQ-SRV включён firewalld (`systemctl is-active firewalld`):
```bash
firewall-cmd --permanent --add-service={nfs,mountd,rpc-bind}
firewall-cmd --reload
```

На HQ-CLI (autofs, прямая карта):
```bash
dnf install nfs-utils autofs -y
echo "/- /etc/auto.direct" >> /etc/auto.master
echo "/mnt/nfs -rw,soft 192.168.100.10:/raid/nfs" > /etc/auto.direct
systemctl enable --now autofs
```
> Альтернатива без autofs - /etc/fstab:
> `192.168.100.10:/raid/nfs /mnt/nfs nfs defaults,_netdev 0 0` + `mkdir /mnt/nfs && mount -a`

Проверка (в отчёт):
```bash
# HQ-SRV
exportfs -v
cat /etc/exports
# HQ-CLI
ls /mnt/nfs
touch /mnt/nfs/test && ls -l /mnt/nfs
mount | grep nfs
```
Основные параметры для отчёта: экспорт /raid/nfs, клиенты 192.168.200.0/28, rw, sync, no_root_squash; на клиенте точка /mnt/nfs через autofs.

---

## 4. chrony (ISP как NTP-сервер)

- [ ] вышестоящий сервер на выбор
- [ ] stratum 5
- [ ] клиенты: HQ-SRV, HQ-CLI, BR-RTR, BR-SRV

На ISP (/etc/chrony.conf) - закомментировать строки `pool ...` по умолчанию и добавить:
```
pool pool.ntp.org iburst
local stratum 5
allow 172.16.1.0/28
allow 172.16.2.0/28
```
> Клиенты из офисов приходят на ISP через NAT роутеров, т.е. с адресов
> 172.16.1.2 (HQ) и 172.16.2.2 (BR) - поэтому allow на эти сети.
> `local stratum 5` - сервер объявляет stratum 5, когда вышестоящий недоступен;
> при доступном вышестоящем stratum = его stratum + 1 (это видно в `chronyc tracking`).

```bash
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
chronyc tracking
```

На клиентах (/etc/chrony.conf) - закомментировать `pool ...`, добавить:
```
# HQ-SRV, HQ-CLI
server 172.16.1.1 iburst
# BR-RTR, BR-SRV
server 172.16.2.1 iburst
```
```bash
systemctl restart chronyd
chronyc sources        # у ISP должна стоять * (синхронизирован)
```
На ISP проверка клиентов (в отчёт):
```bash
chronyc clients
```

---

## 5. Ansible (BR-SRV)

- [ ] inventory: HQ-SRV, HQ-CLI, HQ-RTR, BR-RTR
- [ ] рабочий каталог /etc/ansible
- [ ] все отвечают pong без предупреждений и ошибок

На HQ-CLI (если ещё нет SSH и пользователя для ansible):
```bash
useradd -m sshuser && passwd sshuser        # P@ssw0rd
echo "sshuser ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/sshuser
systemctl enable --now sshd
```

На BR-SRV:
```bash
dnf install ansible -y
mkdir -p /etc/ansible
```
/etc/ansible/hosts:
```ini
[hq]
hq-srv ansible_host=192.168.100.10 ansible_port=2027 ansible_user=sshuser
hq-cli ansible_host=192.168.200.2 ansible_user=sshuser

[routers]
hq-rtr ansible_host=10.10.10.1 ansible_user=net_admin
br-rtr ansible_host=172.30.200.1 ansible_user=net_admin

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```
/etc/ansible/ansible.cfg (без этого будут предупреждения):
```ini
[defaults]
inventory = /etc/ansible/hosts
host_key_checking = False
interpreter_python = auto_silent
```
SSH-ключи:
```bash
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
ssh-copy-id -p 2027 sshuser@192.168.100.10
ssh-copy-id sshuser@192.168.200.2
ssh-copy-id net_admin@10.10.10.1
ssh-copy-id net_admin@172.30.200.1
```
> Адрес HQ-CLI выдаётся по DHCP - проверить на HQ-CLI `ip a` и подставить
> фактический (или закрепить 192.168.200.2 резервацией в dhcpd.conf на HQ-RTR).
> На HQ-SRV в sshd_config `MaxAuthTries 2` - при ssh-copy-id вводить пароль без ошибок.

Проверка (в отчёт):
```bash
cd /etc/ansible
ansible all -m ping
```

---

## 6. Docker-стек (BR-SRV)

- [ ] стек: веб-приложение + БД
- [ ] образы site_latest, mariadb_latest (директория docker в Additional.iso)
- [ ] compose.yaml из Additional.iso
- [ ] testapp доступен снаружи через порт 8080

```bash
lsblk                                   # найти CD (sr0)
mount /dev/sr0 /mnt
ls -R /mnt/docker

dnf install docker-ce docker-ce-cli docker-compose -y   # имена пакетов в RED OS; в других - docker
systemctl enable --now docker

docker load -i /mnt/docker/site_latest.tar      # имена файлов - по факту из ls
docker load -i /mnt/docker/mariadb_latest.tar
docker images

mkdir -p /root/testapp
cp /mnt/docker/compose.yaml /root/testapp/      # путь к compose.yaml - по факту
cd /root/testapp
```
Проверить в compose.yaml:
- имена образов совпадают с `docker images` (site:latest, mariadb:latest и т.п.)
- у веб-сервиса проброс `8080:<порт приложения>`
- переменные БД (пароль, имя базы) у site и mariadb совпадают

```bash
docker compose up -d        # или docker-compose up -d, если стоит compose v1
docker ps
curl http://localhost:8080
```

---

## 7. Веб-приложение (HQ-SRV) - apache + mariadb

- [ ] apache, mariadb
- [ ] файлы и dump.sql - директория web в Additional.iso
- [ ] импорт в БД webdb
- [ ] пользователь web / P@ssw0rd с правами на webdb
- [ ] index.php + logo.png в корень сайта, верные данные подключения
- [ ] проверить, параметры - в отчёт

```bash
mount /dev/sr0 /mnt
ls /mnt/web

dnf install httpd mariadb-server php php-mysqlnd php-fpm -y
systemctl enable --now mariadb php-fpm httpd

mysql -u root -e "CREATE DATABASE webdb;"
mysql -u root -e "CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';"
mysql -u root -e "GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';"
mysql -u root -e "FLUSH PRIVILEGES;"
mysql -u root webdb < /mnt/web/dump.sql
mysql -u root -e "SHOW TABLES FROM webdb;"
```
> Перед импортом глянуть `head -30 /mnt/web/dump.sql`: если там есть
> `CREATE DATABASE`/`USE` с другим именем - поправить на webdb.

```bash
cp /mnt/web/index.php /mnt/web/logo.png /var/www/html/
rm -f /var/www/html/index.html
nano /var/www/html/index.php
```
В index.php указать:
```php
$servername = "localhost";
$username   = "web";
$password   = "P@ssw0rd";
$dbname     = "webdb";
```
```bash
systemctl restart httpd
# если firewalld активен на HQ-SRV:
firewall-cmd --permanent --add-service=http && firewall-cmd --reload
curl http://localhost
```
Проверка в браузере HQ-CLI: http://192.168.100.10 - должна открыться страница с logo.png и данными из БД.

Основные параметры для отчёта: apache (httpd), корень /var/www/html, БД webdb, пользователь web, порт 80.

---

## 8. Статическая трансляция портов (роутеры, firewalld, зона trusted - из Задания 1)

- [ ] BR-RTR: 8080 -> testapp на BR-SRV (172.30.100.10:8080)
- [ ] HQ-RTR: 8080 -> веб-приложение HQ-SRV (192.168.100.10:**80**)
- [ ] HQ-RTR: 2027 -> HQ-SRV:2027
- [ ] BR-RTR: 2027 -> BR-SRV:2027

На HQ-RTR:
```bash
firewall-cmd --zone=trusted --add-forward-port=port=8080:proto=tcp:toport=80:toaddr=192.168.100.10 --permanent
firewall-cmd --zone=trusted --add-forward-port=port=2027:proto=tcp:toport=2027:toaddr=192.168.100.10 --permanent
firewall-cmd --reload
firewall-cmd --zone=trusted --list-forward-ports
```
На BR-RTR:
```bash
firewall-cmd --zone=trusted --add-forward-port=port=8080:proto=tcp:toport=8080:toaddr=172.30.100.10 --permanent
firewall-cmd --zone=trusted --add-forward-port=port=2027:proto=tcp:toport=2027:toaddr=172.30.100.10 --permanent
firewall-cmd --reload
firewall-cmd --zone=trusted --list-forward-ports
```
> Порт приложения testapp на BR-SRV (8080) - сверить с compose.yaml (левая часть `ports:`).
> Трафик до BR-SRV идёт через BR-FW: на BR-FW должен быть включён forwarding и не
> должно быть правил, режущих tcp/8080 и tcp/2027 (если на BR-FW firewalld - зона trusted).

Проверка с ISP (в отчёт):
```bash
curl http://172.16.1.2:8080          # веб-приложение HQ-SRV
curl http://172.16.2.2:8080          # testapp
ssh -p 2027 sshuser@172.16.1.2       # попадаем на HQ-SRV
ssh -p 2027 sshuser@172.16.2.2       # попадаем на BR-SRV
```
> Эти пробросы используются в Задании 3 (реверс-прокси на ISP: web -> HQ-RTR:8080,
> docker -> BR-RTR:8080), поэтому порт 80 на HQ-RTR важен.

---

## 9. Яндекс Браузер на HQ-CLI

- [ ] установить удобным способом
- [ ] зафиксировать в отчёте

```bash
dnf install yandex-browser-stable -y
```
> Если пакета нет в репозитории - скачать .rpm с browser.yandex.ru и
> `dnf install ./yandex-browser-stable*.rpm -y`.

Проверка (в отчёт):
```bash
rpm -q yandex-browser-stable
yandex-browser-stable --version
```
Плюс скриншот запущенного браузера.

---

## Приложения к заданию

- Прил_5_ОЗ_КИМ_09.02.06-1-2027.txt

## Заметки

- Инструкция по настройке стенда для техэксперта - отдельный файл (см. приложения к спецификации КИМ)
- Файлы (site_latest, mariadb_latest, compose.yaml, dump.sql, index.php, logo.png) - в Additional.iso, смонтировать перед началом (`mount /dev/sr0 /mnt`)
