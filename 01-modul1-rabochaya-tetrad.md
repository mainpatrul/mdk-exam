# Модуль 1 - Настройка сетевой инфраструктуры (по рабочей тетради)

Переписано из `Рабочая_тетраль_ДЭ_Модуль_1_финал_проверенно.docx` один в один, скриншоты переведены в текст.
Для тренировки на своём Proxmox. Разбор с исправлениями - в `01-zadanie1-baznaya-nastroyka.md`.

Итоговый отчёт: одна таблица и пять отчётов о ходе работы, сохранить на диске рабочего места.

---

## 0. Линки в Proxmox

Мосты: узел -> System -> Network -> Create -> Linux Bridge (IP не указывать, Bridge ports пусто). vmbr0 - выход в интернет (уже есть).

| Мост | Линк | Кто подключён |
|---|---|---|
| vmbr0 | Internet | ISP |
| vmbr1 | ISP-HQ | ISP, HQ-RTR |
| vmbr2 | ISP-BR | ISP, BR-RTR |
| vmbr3 | HQ-NET / SRV-NET / CLI-NET (HQ-SW) | HQ-RTR, HQ-SRV, HQ-CLI |
| vmbr4 | FW-NET | BR-RTR, BR-FW |
| vmbr5 | BR-NET | BR-FW, BR-SRV |

Hardware -> Add -> Network Device, по порядку:

| ВМ | ens33 | ens34 | ens35 |
|---|---|---|---|
| ISP | vmbr0 | vmbr1 | vmbr2 |
| HQ-RTR | vmbr1 | vmbr3 | - |
| BR-RTR | vmbr2 | vmbr4 | - |
| BR-FW | vmbr4 | vmbr5 | - |
| HQ-SRV | vmbr3 | - | - |
| HQ-CLI | vmbr3 | - | - |
| BR-SRV | vmbr5 | - | - |

VLAN Tag - пусто, Firewall - снять. Имена внутри ВМ проверить `ip a`.

---

## 1. Базовая настройка устройств

**Имена** - полное доменное имя:
```bash
hostnamectl hostname host-name.au-team.irpo    # host-name: hq-srv, br-rtr и т.д.
```

**IPv4:**
```
nmtui > Изменить подключение > Выбираем нужный интерфейс > Стрелочка вправо > Изменить >
Конфигурация IPv4: Автоматически -> Вручную > Показать > Адреса > Добавить
задаём IP-адрес и при необходимости шлюз и серверы DNS > OK
```

Скриншот nmtui (ens33):
```
Имя профиля   ens33
Устройство    ens33
КОНФИГУРАЦИЯ IPv4  <Вручную>                <Скрыть>
  Адреса       <Добавить...>
  Шлюз         ____
  Серверы DNS  <Добавить...>
  Домены поиска <Добавить...>
[X] Подключаться автоматически
[X] Доступно всем пользователям
                                         <Отменить> <OK>
```

На этом пункте настраиваем все интерфейсы на ISP, BR-RTR, BR-FW, BR-SRV. На HQ-RTR - только интерфейс в сторону ISP.
Интерфейсы HQ-RTR, HQ-SRV и HQ-CLI в локальной сети HQ настраиваются в пункте 4.

Применить изменения:
```bash
nmcli connection up INTERFACE     # например: nmcli connection up ens33
```

На маршрутизаторах (ISP / BR-RTR / HQ-RTR) и межсетевом экране (BR-FW) - пересылка пакетов:
```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

**Адресация** (приватные диапазоны RFC1918: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16):

| Сеть | Требование | Маска | Сеть |
|---|---|---|---|
| к HQ-SRV (VLAN100) | не более 32 адресов | 255.255.255.224 /27 | 192.168.100.0/27 |
| к HQ-CLI (VLAN200) | не более 16 адресов | 255.255.255.240 /28 | 192.168.200.0/28 |
| управление (VLAN999) | не более 8 адресов | 255.255.255.248 /29 | 192.168.99.0/29 |
| BR-FW - BR-RTR | минимальная | 255.255.255.252 /30 | 172.30.100.0/30 |
| к BR-SRV | не более 16 адресов | 255.255.255.240 /28 | 172.30.200.0/28 |

**Таблица адресации (в отчёт):**

| Имя устройства | IP-адрес | Шлюз по умолчанию |
|---|---|---|
| ISP | ens33: DHCP<br>ens34: 172.16.1.1/28<br>ens35: 172.16.2.1/28 | |
| HQ-RTR | ens33: 172.16.1.2/28<br>ens34.vlan100: 192.168.100.1/27<br>ens34.vlan200: 192.168.200.1/28<br>ens34.vlan999: 192.168.99.1/29 | ens33: 172.16.1.1 |
| BR-RTR | ens33: 172.16.2.2/28<br>ens34: 172.30.100.1/30 | ens33: 172.16.2.1 |
| HQ-SRV | ens33.vlan100: 192.168.100.10/27 | ens33: 192.168.100.1 |
| BR-FW | ens33: 172.30.100.2/30<br>ens34: 172.30.200.1/28 | ens33: 172.30.100.1 |
| BR-SRV | ens33: 172.30.200.10/28 | ens33: 172.30.200.1 |
| HQ-CLI | ens33.vlan200: DHCP | DHCP |

---

## 2. Настройка ISP

- ens33 (к магистральному провайдеру) - DHCP
- маршруты по умолчанию там, где необходимо
- к HQ-RTR - сеть 172.16.1.0/28, к BR-RTR - 172.16.2.0/28
- динамическая трансляция в сторону HQ-RTR и BR-RTR

На ISP:
```bash
dnf install iptables-services -y
systemctl enable --now iptables
iptables -F
iptables -A FORWARD -s 172.16.0.0/16 -j ACCEPT
iptables -A FORWARD -d 172.16.0.0/16 -j ACCEPT
iptables -t nat -A POSTROUTING -o ens33 -s 172.16.0.0/16 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
```
Проверяем пинги на 8.8.8.8 с HQ-RTR и BR-RTR.

---

## 3. Локальные учётные записи

**sshuser** на HQ-SRV и BR-SRV (пароль P@ssw0rd, UID 2027, sudo без пароля):
```bash
useradd -m -U -s /bin/bash -u 2027 sshuser
passwd sshuser
# P@ssw0rd
# P@ssw0rd
echo "sshuser ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
```

**net_admin** на HQ-RTR и BR-RTR (пароль P@ssw0rd, sudo без пароля):
```bash
useradd -m -U -s /bin/bash net_admin
passwd net_admin
# P@ssw0rd
# P@ssw0rd
echo "net_admin ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
```

---

## 4. Виртуальный коммутатор (VLAN) на HQ-RTR

- HQ-SRV - VLAN 100, HQ-CLI - VLAN 200, подсеть управления - VLAN 999
- сведения о настройке и выборе реализации VLAN - в отчёт

```
nmtui > Изменить подключение > Добавить > VLAN
```
Выполняем на HQ-RTR - ens34, HQ-SRV - ens33, HQ-CLI - ens33. Для HQ-CLI и HQ-SRV указываем шлюз и серверы DNS.

Скриншот nmtui (HQ-RTR, VLAN100):
```
Имя профиля         VLAN100
Устройство          ens34.100
VLAN
  Родительский      ens34
  Идентификатор VLAN 100
КОНФИГУРАЦИЯ IPv4   <Вручную>
  Адреса            <Добавить...>
  Шлюз              ____
  Серверы DNS       <Добавить...>
```

| Где | Профиль | Устройство | Родитель | ID | Адрес |
|---|---|---|---|---|---|
| HQ-RTR | VLAN100 | ens34.100 | ens34 | 100 | 192.168.100.1/27 |
| HQ-RTR | VLAN200 | ens34.200 | ens34 | 200 | 192.168.200.1/28 |
| HQ-RTR | VLAN999 | ens34.999 | ens34 | 999 | 192.168.99.1/29 |
| HQ-SRV | VLAN100 | ens33.100 | ens33 | 100 | 192.168.100.10/27, шлюз 192.168.100.1 |
| HQ-CLI | VLAN200 | ens33.200 | ens33 | 200 | DHCP |

---

## 5. Безопасный удалённый доступ на HQ-SRV и BR-SRV

- порт 2027, только sshuser, 2 попытки входа, баннер «Authorized access only»

Создаём баннер:
```bash
echo "Authorized access only" > /etc/ssh/banner.txt
```

Настраиваем SSH:
```bash
nano /etc/ssh/sshd_config
```
```
Port 2027
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/ssh/banner.txt
```

Открываем порт:
```bash
semanage port -a -t ssh_port_t -p tcp 2027
```
(либо отключаем SELinux: в `/etc/selinux/config` меняем `SELINUX=enforcing` на `SELINUX=disabled` и `reboot`)

Проверяем с HQ-RTR и BR-FW доступ до серверов своей локальной сети:
```bash
ssh -l sshuser 172.30.200.10 -p 2027     # с BR-FW
ssh -l sshuser 192.168.100.10 -p 2027    # с HQ-RTR
```

---

## 6. IP-туннель между офисами HQ и BR

- GRE, диапазон 10.10.10.0/24, минимальная маска; сведения - в отчёт

```
nmtui > стрелочка вправо > Добавить > IP-туннель   (не забыть режим GRE)
```

Скриншот HQ-RTR:
```
Имя профиля   tun0
Устройство    tun0
Туннель IP
  Режим           <GRE>
  Родительский    ens33
  Локальный IP    172.16.1.2
  Удалённый IP    172.16.2.2
КОНФИГУРАЦИЯ IPv4 <Вручную>
  Адреса          10.10.10.1/30
```

Скриншот BR-RTR:
```
Имя профиля   tun0
Устройство    tun0
Туннель IP
  Режим           <GRE>
  Родительский    ens33
  Локальный IP    172.16.2.2
  Удалённый IP    172.16.1.2
КОНФИГУРАЦИЯ IPv4 <Вручную>
  Адреса          10.10.10.2/30
```

После этого на обоих роутерах:
```bash
nmcli connection modify tun0 ip-tunnel.ttl 64
```
и перезапускаем туннель через nmtui (выключаем и включаем интерфейс).
Проверяем пинги с двух роутеров на 10.10.10.1 и 10.10.10.2.

---

## 7. Динамическая маршрутизация (OSPF)

- HQ-RTR, BR-RTR, BR-FW; сети одного офиса доступны из другого
- только на интерфейсах туннеля на HQ-RTR и BR-RTR, на BR-FW - только интерфейс к BR-RTR
- аутентификация на HQ-RTR и BR-RTR, пароль P@ssw0rd, хеш MD5; сведения - в отчёт

На HQ-RTR, BR-RTR и BR-FW:
```bash
dnf install frr
systemctl enable --now frr
nano /etc/frr/daemons       # заменить no на yes: ospfd=yes
systemctl restart frr
```

```
vtysh
conf t
router ospf
```
Дальше работа как в Cisco:

HQ-RTR:
```
network 192.168.100.0/27 area 0
network 192.168.200.0/28 area 0
network 192.168.99.0/29 area 0
network 10.10.10.0/30 area 0
ospf router-id 172.16.1.2
passive-interface ens33
passive-interface ens34
```

BR-RTR:
```
network 172.30.100.0/30 area 0
network 10.10.10.0/30 area 0
ospf router-id 172.16.2.2
passive-interface ens33
```

BR-FW:
```
network 172.30.100.0/30 area 0
network 172.30.200.0/28 area 0
ospf router-id 172.30.100.2
passive-interface ens34
```

HQ-RTR и BR-RTR - аутентификация:
```
exit
interface tun0
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 P@ssw0rd
do wr
exit
exit
exit
```

---

## 8. Динамическая трансляция адресов

На HQ-RTR и BR-RTR:
```bash
dnf install iptables-services
systemctl enable --now iptables
iptables -F
iptables -t nat -A POSTROUTING -o ens33 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
```

---

## 9. DHCP

- сервер - HQ-RTR, клиент - HQ-CLI, адрес роутера исключён
- шлюз - HQ-RTR, DNS - HQ-SRV, суффикс au-team.irpo; сведения - в отчёт

```bash
dnf install dhcp-server
nano /etc/dhcp/dhcpd.conf
```
```
subnet 192.168.200.0 netmask 255.255.255.240 {
range 192.168.200.2 192.168.200.14;
option routers 192.168.200.1;
option domain-name-servers 192.168.100.10;
option domain-name "au-team.irpo";
}
```
```bash
systemctl enable --now dhcpd
dhcpd
```
Получаем адрес на HQ-CLI: выключаем и включаем интерфейс ens33.vlan200.
Проверяем на HQ-RTR запись о выдаче адреса клиенту:
```bash
cat /var/lib/dhcpd/dhcpd.leases
```

---

## 10. DNS для офисов HQ и BR

- основной DNS - HQ-SRV, прямое и обратное разрешение по таблице, пересылка на общедоступный DNS

| Устройство | Запись | Тип |
|---|---|---|
| HQ-RTR | hq-rtr.au-team.irpo | A |
| BR-RTR | br-rtr.au-team.irpo | A |
| BR-FW | br-fw.au-team.irpo | A |
| HQ-SRV | hq-srv.au-team.irpo | A, PTR |
| HQ-CLI | hq-cli.au-team.irpo | A |
| BR-SRV | br-srv.au-team.irpo | A, PTR |
| ISP (к HQ-RTR) | docker.au-team.irpo | A |
| ISP (к BR-RTR) | web.au-team.irpo | A |

```bash
dnf install bind
nano /etc/named.conf
```
Изменить строчки (на скрине отмечены стрелками):
```
options {
        listen-on port 53 { any; };          <--
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        ...
        allow-query     { any; };            <--
        forwarders      { 77.88.8.7; };      <--
```

В конец добавить:
```
zone "au-team.irpo" IN {
        type master;
        file "/opt/dns/au-team.irpo";
};

zone "100.168.192.in-addr.arpa" IN {
        type master;
        file "/opt/dns/100.168.192.in-addr.arpa";
};

zone "200.30.172.in-addr.arpa" IN {
        type master;
        file "/opt/dns/200.30.172.in-addr.arpa";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

Копируем шаблон и заполняем:
```bash
mkdir /opt/dns
cd /opt/dns
cp /var/named/named.empty au-team.irpo
nano au-team.irpo
```
```
$TTL 3H
au-team.irpo.   IN SOA  au-team.irpo. root.au-team.irpo. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      hq-srv.au-team.irpo.
hq-rtr  A       192.168.100.1
hq-rtr  A       192.168.200.1
hq-srv  A       192.168.100.10
hq-cli  A       192.168.200.2
br-rtr  A       172.30.100.1
br-srv  A       172.30.200.10
br-fw   A       172.30.200.1
docker  A       172.16.1.1
web     A       172.16.2.1
```

```bash
cp /var/named/named.empty 100.168.192.in-addr.arpa
nano 100.168.192.in-addr.arpa
```
```
$TTL 3H
@       IN SOA  au-team.irpo. root.au-team.irpo. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      hq-srv.au-team.irpo.
10      PTR     hq-srv.au-team.irpo.
```

```bash
cp /var/named/named.empty 200.30.172.in-addr.arpa
nano 200.30.172.in-addr.arpa
```
```
$TTL 3H
@       IN SOA  au-team.irpo. root.au-team.irpo. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      hq-srv.au-team.irpo.
10      PTR     br-srv.au-team.irpo.
```

```bash
chown -R named:named /opt/dns/
named-checkconf -z          # проверяем конфигурацию, исправляем ошибки если есть
systemctl restart named
```

Далее в nmtui меняем DNS-сервер с 8.8.8.8 (10.39.0.1) на 192.168.100.10, домен поиска au-team.irpo.
Затем nmtui > «Активировать подключение» > выключаем и включаем интерфейс, на который ставили DNS.

Скриншот (HQ-RTR, ens33 после смены DNS):
```
Имя профиля   ens33
КОНФИГУРАЦИЯ IPv4 <Вручную>
  Адреса       172.16.1.2/28
  Шлюз         172.16.1.1
  Серверы DNS  192.168.100.10
```

Проверяем на HQ-CLI:
```bash
ping br-rtr
ping br-srv
ping hq-rtr
ping hq-srv
ping ya.ru
```

---

## 11. Часовой пояс

На всех устройствах, согласно месту проведения экзамена:
```bash
timedatectl set-timezone Europe/Moscow
timedatectl
```
(проверить зону; время по заданию менять не просят)

---

## Если на своём Proxmox что-то не работает

Отличия от тетради, которые вылезают на чистой RED OS (подробно - в `01-zadanie1-baznaya-nastroyka.md`):

| Симптом | Что сделать |
|---|---|
| iptables не применяется / форвард режется | `systemctl disable --now firewalld`, потом заново `systemctl restart iptables` |
| `dnf` не качает | на интерфейсе нет DNS - поставить 8.8.8.8 (или DNS своей сети) |
| SSH на 2027 не пускает снаружи | `firewall-cmd --permanent --add-port=2027/tcp && firewall-cmd --reload` (если firewalld активен) |
| sudoers ругается | кавычки из Word - писать прямые `"` |
| ping по tun0 есть, OSPF-соседа нет | проверить `ip -d link show tun0` - должно быть `ttl 64` |
| HQ не видит BR-SRV | `vtysh -c "show ip ospf neighbor"` на BR-RTR - BR-FW должен быть Full |
| named: permission denied | `restorecon -Rv /opt/dns` или SELinux disabled |
