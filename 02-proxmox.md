# Задание 2 (Модуль 2) - Организация сетевого администрирования (Proxmox, ens18/19/20)

Источники: `Рабочая_тетраль_ДЭ_Модуль_2_финал_поверенно.docx` (команды), `Additional.iso` (имена файлов, compose.yaml), адреса - `00-interfaces-addressing.md`.
Отчёт: ФамилияУчастникаЗадание2 - скриншоты пунктов "в отчёт", читаемые.
Статус: команды прогнаны на стенде 2026-11-02, настройка переживает холодный запуск всех ВМ. Тогда шли по устройствам, порядок по пунктам целиком не прогонялся.

Настройка разложена **по пунктам тетради** - номер раздела = номер задания, внутри указано, на какой ВМ что делать.

**Порядок: 4 -> 1 -> 2 -> 3 -> 5 -> 6 -> 7 -> 8 -> 9.** Время настраиваем первым: без него не работает Kerberos. Остальное идёт по номерам.

| Пункт | Что | Где | В отчёт |
|---|---|---|---|
| 1 | Samba DC, домен, группа hq, sudo | HQ-SRV (DNS), BR-SRV, HQ-CLI | да |
| 2 | RAID0 | HQ-SRV | да |
| 3 | NFS | HQ-SRV, HQ-CLI | да |
| 4 | chrony | ISP, затем HQ-SRV, HQ-CLI, BR-RTR, BR-SRV | да |
| 5 | Ansible | HQ-CLI (SSH), BR-SRV | да |
| 6 | Docker | BR-SRV | да |
| 7 | Веб-приложение | HQ-SRV | да |
| 8 | Проброс портов | BR-RTR, HQ-RTR | да |
| 9 | Яндекс Браузер | HQ-CLI | да |

## Где мы отличаемся от тетради (и почему)

- **Образ mariadb**: в `compose.yaml` указан `image: mariadb` (это `latest`), а в архиве из ISO лежит `mariadb:10.11`. Без `docker tag` compose полезет в интернет
- **compose.yaml**: в тетради `compose.yml`, в ISO файл называется `compose.yaml`
- **realm join**: в тетради `-U -v Administrator@...` - похоже на опечатку, у нас `-v -U ... au-team.irpo`. Если не сработает - попробовать вариант тетради
- **Пакеты**: `nfs-utils` вместо `exportfs`; `mdadm`, `chrony` и клиент домена ставим явно (в чистой системе могут отсутствовать)
- **firewalld на BR-SRV**: выключаем, иначе режет Samba и Docker (HQ-SRV и роутеры выключены ещё в Модуле 1)
- **fstab на HQ-CLI**: добавлен `_netdev`, чтобы NFS монтировался после поднятия сети
- **Кавычки**: везде прямые `"`. «Ёлочки» и “фигурные” из Word ломают `/etc/hosts` и sudoers

---

## Адреса

| Узел | Адрес |
|---|---|
| ISP | 172.16.1.1 (к HQ-RTR), 172.16.2.1 (к BR-RTR) |
| HQ-RTR | внешний 172.16.1.2, внутренний 192.168.100.1 |
| BR-RTR | внешний 172.16.2.2, внутренний 172.30.100.1 |
| HQ-SRV | 192.168.100.10, SSH 2027 |
| BR-SRV | 172.30.200.10, SSH 2027 |
| HQ-CLI | 192.168.200.2 (DHCP, проверить `ip a`) |

Пароль везде P@ssw0rd.

---

## 1. Samba DC (в отчёт!)

> Сначала пункт 4: без времени не работает Kerberos.

- [ ] домен au-team.irpo, в него введена HQ-CLI
- [ ] пароль от 8 символов, блокировка после 5 попыток
- [ ] hquser1-5 в группе hq, могут входить на HQ-CLI
- [ ] sudo для группы hq: только cat, grep, id

### Samba DC: тетрадь и условия экзамена

По нашим заметкам к заданию DC на экзамене может быть **уже установлен и работать** (без пользователей, групп и OU), а тетрадь разворачивает его с нуля. Перед началом проверить на BR-SRV:
```bash
systemctl is-active samba
samba-tool domain info 127.0.0.1
```
Если отвечает - блок "Развернуть контроллер домена" пропустить и начать с `kinit`. Пароль administrator - уточнить у техэксперта.

### HQ-SRV: DNS
Шаг из тетради перед настройкой домена:
```bash
systemctl restart named
nslookup br-srv.au-team.irpo 127.0.0.1
```

### BR-SRV: развернуть контроллер домена
> Пропустить, если DC уже работает (см. выше).

```bash
systemctl disable --now firewalld     # иначе режет Samba и Docker
dnf install -y samba* krb5*
mv /etc/samba/smb.conf /etc/samba/smb.conf.bak
mv /etc/krb5.conf /etc/krb5.conf.bak
nano /etc/krb5.conf.d/crypto-policies
```
Курсор на строку `permitted_enctypes`, `Ctrl+K`, затем три раза `Ctrl+U` - будет три одинаковые строки. У первых двух поменять только начало:
```
[libdefaults]
default_tgs_enctypes = <значение как было>
default_tkt_enctypes = <значение как было>
permitted_enctypes = <значение как было>
```
```bash
samba-tool domain provision --use-rfc2307 --interactive
```
Enter 3 раза, на `DNS backend` написать **NONE**, пароль администратора P@ssw0rd два раза.

```bash
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
nano /etc/krb5.conf
```
В `[realms]` добавить `kdc`:
```
AU-TEAM.IRPO = {
        default_domain = au-team.irpo
        kdc = br-srv.au-team.irpo
}
```
```bash
systemctl enable --now samba
```
> Службы `smb`, `nmb`, `winbind`, `krb5kdc` **не запускать** - HQ-CLI не войдёт в домен. Если запустил - перезагрузить BR-SRV.

### BR-SRV: политика, пользователи, группа
> firewalld на BR-SRV выключить и тогда, когда DC уже работал и блок выше пропущен: `systemctl disable --now firewalld`.

```bash
kinit administrator@AU-TEAM.IRPO      # P@ssw0rd
samba-tool domain passwordsettings set --min-pwd-length=8 --account-lockout-threshold=5
samba-tool domain passwordsettings show
samba-tool group add hq
for i in {1..5}; do samba-tool user add hquser$i P@ssw0rd; samba-tool group addmembers hq hquser$i; done
samba-tool group listmembers hq
```

### HQ-CLI: ввод в домен
```bash
sudo -i                # если вошёл обычным пользователем
dnf install realmd sssd oddjob oddjob-mkhomedir adcli samba-common-tools krb5-workstation -y
echo "172.30.200.10 au-team.irpo" >> /etc/hosts
nano /etc/krb5.conf
```
В конец `[libdefaults]` и разделы ниже:
```
udp_preference_limit = 0
default_realm = AU-TEAM.IRPO

[realms]
AU-TEAM.IRPO = {
    kdc = br-srv.au-team.irpo
    admin_server = br-srv.au-team.irpo
}

[domain_realm]
.au-team.irpo = AU-TEAM.IRPO
au-team.irpo = AU-TEAM.IRPO
```
```bash
realm join -v -U Administrator@AU-TEAM.IRPO au-team.irpo     # P@ssw0rd
```
В конце должно быть `Successfully enrolled machine in realm`.

```bash
nano /etc/sssd/sssd.conf
```
В секции `[domain/au-team.irpo]` дописать `br-srv.` в строку:
```
ad_server = br-srv.au-team.irpo
```
```bash
systemctl restart sssd
```
Настройки -> Сетевые учётные записи -> Kerberos -> `administrator@AU-TEAM.IRPO`, пароль P@ssw0rd.

### HQ-CLI: sudo для группы hq
```bash
sss_cache -E
nano /etc/sudoers
```
Под строкой `%wheel ALL=(ALL) ALL`:
```
%hq@au-team.irpo ALL=(ALL) NOPASSWD: /bin/cat, /bin/grep, /bin/id
```
```bash
visudo -c
id hquser1@AU-TEAM.IRPO
su - hquser1@AU-TEAM.IRPO
sudo cat /etc/hosts | grep au-team
sudo id                # работает
sudo ls /root          # отказ
exit
```
Плюс скриншот входа в графическую сессию доменным пользователем.

---

## 2. RAID0 на HQ-SRV (в отчёт!)

- [ ] RAID0 из двух дисков 1 ГБ, md0, конфиг в /etc/mdadm.conf
- [ ] ext4, автомонтирование в /raid

```bash
dnf install mdadm -y
lsblk                  # должны быть sdb и sdc
fdisk /dev/sdb         # g, n, Enter 3 раза, w
fdisk /dev/sdc         # то же
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb1 /dev/sdc1
lsblk                  # у md0 тип raid0
mdadm --detail --scan --verbose >> /etc/mdadm.conf
mkfs.ext4 /dev/md0
mkdir /raid
nano /etc/fstab
```
В конец:
```
/dev/md0  /raid  ext4  defaults  0  0
```
```bash
systemctl daemon-reload
mount -a
df -h                  # есть строка /raid
cat /proc/mdstat
```

---

## 3. NFS (в отчёт!)

- [ ] /raid/nfs, чтение и запись только для сети HQ-CLI
- [ ] на HQ-CLI автомонтирование в /mnt/nfs

### HQ-SRV: сервер
```bash
dnf install nfs-utils nfs4-acl-tools -y
mkdir /raid/nfs
chown nobody:nobody /raid/nfs
chmod 777 /raid/nfs
nano /etc/exports
```
```
/raid/nfs 192.168.200.0/28(rw)
```
```bash
exportfs -a
systemctl enable --now nfs-server
exportfs -v
```
Для отчёта: NFS-сервер HQ-SRV, экспорт /raid/nfs, клиенты 192.168.200.0/28, права rw.

### HQ-CLI: клиент
```bash
dnf install nfs-utils -y
mkdir /mnt/nfs
nano /etc/fstab
```
В конец:
```
192.168.100.10:/raid/nfs  /mnt/nfs  nfs  auto,_netdev  0  0
```
```bash
systemctl daemon-reload
mount -a
df -h                  # есть строка 192.168.100.10:/raid/nfs
touch /mnt/nfs/test && ls -l /mnt/nfs
```

---

## 4. chrony (в отчёт!) - делать ПЕРВЫМ

- [ ] вышестоящий сервер на выбор, стратум 5
- [ ] клиенты: HQ-SRV, HQ-CLI, BR-RTR, BR-SRV

### ISP: сервер
```bash
nano /etc/chrony.conf
```
Оставить первый сервер, остальные закомментировать и дописать три строки:
```
server ntp1.vniiftri.ru iburst
#server ntp2.vniiftri.ru iburst
#server ntp3.vniiftri.ru iburst
#server ntp4.vniiftri.ru iburst
local stratum 5
allow 0/0
hwtimestamp *
```
```bash
systemctl restart chronyd
chronyc sources
chronyc tracking
```

### Клиенты: одинаково на всех четырёх
| Узел | Строка server |
|---|---|
| HQ-SRV, HQ-CLI (`sudo -i`) | `server 172.16.1.1 iburst prefer` |
| BR-RTR, BR-SRV | `server 172.16.2.1 iburst prefer` |

```bash
dnf install chrony -y
nano /etc/chrony.conf
```
Все `server`/`pool` закомментировать, добавить строку из таблицы.
```bash
systemctl restart chronyd
chronyc sources        # у ISP должна появиться *
```

После всех клиентов на ISP: `chronyc clients`.

---

## 5. Ansible (в отчёт!)

- [ ] инвентарь: HQ-SRV, HQ-CLI, HQ-RTR, BR-RTR
- [ ] рабочий каталог /etc/ansible
- [ ] все отвечают pong без предупреждений и ошибок

### HQ-CLI: SSH
```bash
systemctl enable --now sshd
whoami                 # имя обычного пользователя, в тетради username - нужно ниже
```

### BR-SRV
```bash
dnf install ansible -y
mkdir -p /etc/ansible
nano /etc/ansible/demo.ini
```
```ini
[HQ_SRV]
192.168.100.10 ansible_user=sshuser ansible_port=2027
[HQ_CLI]
192.168.200.2 ansible_user=username
[HQ_RTR]
192.168.100.1 ansible_user=net_admin
[BR_RTR]
172.30.100.1 ansible_user=net_admin
```
> `username` заменить на имя из `whoami` на HQ-CLI, адрес HQ-CLI сверить с `ip a`.

Ключи (Enter до конца, на отпечаток `yes`, пароли P@ssw0rd):
```bash
ssh-keygen -t rsa
ssh-copy-id -p 2027 sshuser@192.168.100.10
ssh-copy-id username@192.168.200.2
ssh-copy-id net_admin@192.168.100.1
ssh-copy-id net_admin@172.30.100.1
```
> На HQ-SRV `MaxAuthTries 2` - пароль вводить без ошибок.

```bash
nano /etc/ansible/ansible.cfg
```
```ini
[defaults]
inventory = /etc/ansible/demo.ini
ask_pass = False
host_key_checking = False
interpreter_python = /usr/bin/python
```
```bash
cd /etc/ansible
ansible -m ping all    # 4 раза SUCCESS и pong
```

---

## 6. Docker на BR-SRV (в отчёт!)

- [ ] образы из ISO, compose.yaml из ISO, testapp на порту 8080

PS: при сдаче в VMware перед `mount` подключить ISO в настройках этой ВМ: CD/DVD driver 1 > Use ISO image file > Browse > Additional.iso > Open > OK, в уведомлении согласиться, снова настройки ВМ > CD/DVD driver 1 > Connected > OK

```bash
mount /dev/cdrom /mnt
dnf install docker-ce docker-compose -y
systemctl enable --now docker
docker --version
docker load < /mnt/docker/site_latest.tar
docker load < /mnt/docker/mariadb_latest.tar
docker tag mariadb:10.11 mariadb:latest      # compose ищет mariadb:latest
docker images
cp /mnt/docker/compose.yaml .
docker-compose -f compose.yaml up -d
docker ps                                    # контейнеры db и testapp
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080   # 200; curl -I даёт 405, testapp не принимает HEAD
```
Проверка с HQ-CLI в браузере: http://172.30.200.10:8080

Проверка как у экзаменатора (из readme в ISO): заполнить через сайт 3 строки, `reboot` BR-SRV, снова открыть сайт - записи должны остаться.

---

## 7. Веб-приложение на HQ-SRV (в отчёт!)

- [ ] apache, mariadb, БД webdb, пользователь web
- [ ] index.php и logo.png в корне сайта

PS: при сдаче в VMware перед `mount` подключить ISO в настройках этой ВМ: CD/DVD driver 1 > Use ISO image file > Browse > Additional.iso > Open > OK, в уведомлении согласиться, снова настройки ВМ > CD/DVD driver 1 > Connected > OK

```bash
mount /dev/cdrom /mnt
dnf install mariadb-server httpd php php-mysqlnd php-cli -y
systemctl enable --now mariadb httpd
mysql_secure_installation
```
Ответы по порядку: `P@ssw0rd`, `n`, `n`, `y`, `y`, `y`, `y`.

```bash
mysql -u root -p       # P@ssw0rd
```
```sql
CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';
CREATE DATABASE webdb DEFAULT CHARACTER SET utf8;
GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';
flush privileges;
\q
```
```bash
mysql -u root -p webdb < /mnt/web/dump.sql
cp /mnt/web/index.php /var/www/html
cp /mnt/web/logo.png /var/www/html
nano /var/www/html/index.php
```
Верхние строки файла:
```php
$servername = "localhost";
$username = "web";
$password = "P@ssw0rd";
$dbname = "webdb";
```
```bash
systemctl restart httpd
curl -s http://localhost | head
```
Проверка с HQ-CLI в браузере: http://192.168.100.10

Для отчёта: apache (httpd), корень /var/www/html, порт 80, mariadb, БД webdb, пользователь web.

---

## 8. Проброс портов (в отчёт!)

### BR-RTR
- [ ] 8080 -> testapp на BR-SRV, 2027 -> SSH на BR-SRV

```bash
iptables -t nat -A PREROUTING -p tcp -m tcp -d 172.16.2.2 --dport 8080 -j DNAT --to-destination 172.30.200.10:8080
iptables -t nat -A PREROUTING -p tcp -m tcp -d 172.16.2.2 --dport 2027 -j DNAT --to-destination 172.30.200.10:2027
iptables-save > /etc/sysconfig/iptables
iptables -t nat -L PREROUTING -n
```

### HQ-RTR
- [ ] 8080 -> веб-приложение HQ-SRV (**порт 80**), 2027 -> SSH на HQ-SRV

```bash
iptables -t nat -A PREROUTING -p tcp -m tcp -d 172.16.1.2 --dport 8080 -j DNAT --to-destination 192.168.100.10:80
iptables -t nat -A PREROUTING -p tcp -m tcp -d 172.16.1.2 --dport 2027 -j DNAT --to-destination 192.168.100.10:2027
iptables-save > /etc/sysconfig/iptables
iptables -t nat -L PREROUTING -n
```

### Проверка с HQ-CLI
```bash
curl -s -o /dev/null -w "%{http_code}\n" http://172.16.2.2:8080    # testapp, в браузере тот же сайт, что в п.6
curl -s -o /dev/null -w "%{http_code}\n" http://172.16.1.2:8080    # веб-приложение, сайт из п.7
ssh -l sshuser 172.16.2.2 -p 2027            # попадаем на br-srv
ssh -l sshuser 172.16.1.2 -p 2027            # попадаем на hq-srv
```

---

## 9. Яндекс Браузер на HQ-CLI (в отчёт!)

Окно настройки РЕДОС -> Партнёрские приложения -> Яндекс Браузер. После загрузки: клавиша Win -> Другие приложения -> вторая страница -> запустить.
```bash
rpm -qa | grep -i yandex
```
Плюс скриншот запущенного браузера.

---

## Итоговая проверка

На ISP:
```bash
chronyc clients                              # HQ-SRV, HQ-CLI, BR-RTR, BR-SRV (через NAT - адреса роутеров)
curl -s -o /dev/null -w "%{http_code}\n" http://172.16.1.2:8080
curl -s -o /dev/null -w "%{http_code}\n" http://172.16.2.2:8080
```
На HQ-SRV: `cat /proc/mdstat`, `df -h /raid`, `exportfs -v`.
На HQ-CLI: `df -h /mnt/nfs`, `realm list`.
На BR-SRV: `docker ps`, `ansible -m ping all`.

---

## Быстрая диагностика

| Симптом                                          | Причина                                             | Проверка / решение                                                                    |
| ------------------------------------------------ | --------------------------------------------------- | ------------------------------------------------------------------------------------- |
| kinit или realm join: ошибка времени             | часы разошлись                                      | `chronyc sources`, `timedatectl`                                                      |
| realm join не видит домен                        | нет строки в /etc/hosts, закрыт firewalld на BR-SRV | `cat /etc/hosts`, `systemctl is-active firewalld`, `realm discover au-team.irpo`      |
| realm join падает после запуска лишних служб     | запущены smb/nmb/winbind/krb5kdc                    | перезагрузить BR-SRV, `systemctl is-active samba`                                     |
| hquser1 не находится                             | кэш sssd или нет ad_server                          | `sss_cache -E`, `systemctl restart sssd`                                              |
| sudo просит пароль или отказывает на cat         | опечатка или "фигурные" кавычки                     | `visudo -c`, `sudo -l`                                                                |
| samba/nfs/httpd не работают без явной ошибки     | SELinux                                             | `getenforce`, временно `setenforce 0`, `journalctl -xe`                               |
| docker-compose качает mariadb или падает на pull | нет тега latest                                     | `docker tag mariadb:10.11 mariadb:latest`                                             |
| index.php показывает код или ошибку 503          | не запущен обработчик php                           | `dnf install php-fpm -y`, `systemctl enable --now php-fpm`, `systemctl restart httpd` |
| index.php: ошибка подключения к БД               | неверные данные в файле                             | начало `/var/www/html/index.php`, `mysql -u web -p webdb`                             |
| Проброс 8080 не открывается                      | сервис не отвечает внутри или опечатка в правиле    | `curl` с роутера на адрес сервера, `iptables -t nat -L -n -v`                         |
| ansible: нет /usr/bin/python                     | на узле только python3                              | в ansible.cfg `interpreter_python = /usr/bin/python3`                                 |
| ansible: UNREACHABLE                             | не скопирован ключ или не тот пользователь          | `ssh -p 2027 sshuser@192.168.100.10`                                                  |
| RAID после перезагрузки стал md127               | нет записи в mdadm.conf                             | `cat /etc/mdadm.conf`, `cat /proc/mdstat`                                             |
