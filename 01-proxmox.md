# Задание 1 (Модуль 1) - Настройка сетевой инфраструктуры (Proxmox, ens18/19/20)

Продолжительность: 1 ч.
Источники: `Рабочая_тетраль_ДЭ_Модуль_1_финал_проверенно.docx` (адреса, DNS-записи) и `1 модуль Задание 2027.docx` (текст задания).
Отчёт: ФамилияУчастникаЗадание1 - **одна таблица (адресация) и пять отчётов**: VLAN, GRE, OSPF, DHCP + пункты "в отчёт". Скриншоты читаемые.

Настройка разложена **по устройствам** - заходишь на ВМ и делаешь всё сверху вниз.
Порядок ВМ: ISP -> HQ-RTR -> BR-RTR -> BR-FW -> HQ-SRV -> BR-SRV -> HQ-CLI -> общий финал -> проверка.

## Где мы отличаемся от тетради (и почему)

- **OSPF passive**: в тетради `passive-interface ens34` (у нас ens19) - VLAN-ы ens19.100/200/999 это отдельные интерфейсы, OSPF на них остаётся включён. У нас `passive-interface default` + `no passive-interface` только на нужных - ровно как в задании
- **ISP / роутеры**: перед iptables выключаем firewalld, иначе они конфликтуют
- **firewalld на HQ-SRV**: по заданию не нужен - выключаем (`systemctl disable --now firewalld`); на BR-SRV - если активен, открываем 2027/tcp
- **DNS /opt/dns**: если named пишет `permission denied` - восстановить SELinux-контекст
- **HQ-CLI**: DNS вручную не задаём - он должен прийти по DHCP

---

## Схема (Рисунок 1) с адресами

```
                          Internet
                             | ens18 (DHCP)
        ISP-HQ ens19 ——————— ISP ——————— ens20 ISP-BR
        172.16.1.0/28                     172.16.2.0/28
             | ens18 .2                        | ens18 .2
          HQ-RTR ====== GRE tun0 10.10.10.0/30 ====== BR-RTR
             | ens19 (trunk)                   | ens19 .1
          HQ-NET: VLAN 100/200/999          FW-NET 172.30.100.0/30
             |                                 | ens18 .2
           HQ-SW                             BR-FW
          /      \                             | ens19 .1
   SRV-NET        CLI-NET                   BR-NET 172.30.200.0/28
   VLAN100 /27    VLAN200 /28                  | ens18 .10
   192.168.100.0  192.168.200.0              BR-SRV
      | .10          | DHCP
   HQ-SRV         HQ-CLI
```

- HQ-SW - это мост vmbr3 в Proxmox; VLAN-ы ставят сами ВМ (ens19.100 и т.д.).
- VLAN 999 (192.168.99.0/29) - сеть управления, только на HQ-RTR.

## Линки в Proxmox (Hardware -> Add -> Network Device)

Мосты: узел -> System -> Network -> Create -> Linux Bridge (IP не указывать, Bridge ports пусто). vmbr0 уже есть - выход в интернет.

| Мост | Линк на схеме | Кто подключён |
|---|---|---|
| vmbr0 | Internet | ISP |
| vmbr1 | ISP-HQ | ISP, HQ-RTR |
| vmbr2 | ISP-BR | ISP, BR-RTR |
| vmbr3 | HQ-NET + SRV-NET + CLI-NET (HQ-SW) | HQ-RTR, HQ-SRV, HQ-CLI |
| vmbr4 | FW-NET | BR-RTR, BR-FW |
| vmbr5 | BR-NET | BR-FW, BR-SRV |

Карты добавлять строго по порядку: первая - ens18, вторая - ens19, третья - ens20.

| ВМ | ens18 | ens19 | ens20 |
|---|---|---|---|
| ISP | vmbr0 (интернет) | vmbr1 (к HQ-RTR) | vmbr2 (к BR-RTR) |
| HQ-RTR | vmbr1 (к ISP) | vmbr3 (VLAN-ы к HQ-SW) | - |
| BR-RTR | vmbr2 (к ISP) | vmbr4 (к BR-FW) | - |
| BR-FW | vmbr4 (к BR-RTR) | vmbr5 (к BR-SRV) | - |
| HQ-SRV | vmbr3 | - | - |
| HQ-CLI | vmbr3 | - | - |
| BR-SRV | vmbr5 | - | - |

VLAN Tag на картах - пусто, галку Firewall снять. Имена проверять `ip a`. В тетради преподавателя те же карты называются ens33/ens34/ens35 - ens33=ens18, ens34=ens19, ens35=ens20.

## Таблица адресации (в отчёт)

| Имя устройства | IP-адрес | Шлюз по умолчанию |
|---|---|---|
| ISP | ens18: DHCP; ens19: 172.16.1.1/28; ens20: 172.16.2.1/28 | DHCP |
| HQ-RTR | ens18: 172.16.1.2/28; ens19.100: 192.168.100.1/27; ens19.200: 192.168.200.1/28; ens19.999: 192.168.99.1/29; tun0: 10.10.10.1/30 | 172.16.1.1 |
| BR-RTR | ens18: 172.16.2.2/28; ens19: 172.30.100.1/30; tun0: 10.10.10.2/30 | 172.16.2.1 |
| HQ-SRV | ens18.100: 192.168.100.10/27 | 192.168.100.1 |
| BR-FW | ens18: 172.30.100.2/30; ens19: 172.30.200.1/28 | 172.30.100.1 |
| BR-SRV | ens18: 172.30.200.10/28 | 172.30.200.1 |
| HQ-CLI | ens18.200: DHCP (192.168.200.2-14) | DHCP (192.168.200.1) |

Маски: VLAN100 ≤32 -> /27; VLAN200 ≤16 -> /28; VLAN999 ≤8 -> /29; BR-FW-BR-RTR минимальная -> /30; BR-SRV ≤16 -> /28; GRE из 10.10.10.0/24 минимальная -> /30.

## Общие действия (на каждой ВМ)

Имя и часовой пояс (на всех, кроме HQ-SW):
```bash
hostnamectl hostname <имя>.au-team.irpo    # isp / hq-rtr / br-rtr / br-fw / hq-srv / br-srv / hq-cli
timedatectl set-timezone Europe/Moscow     # пояс места проведения экзамена
timedatectl                                # проверить зону
```

IP через nmtui:
```
nmtui -> Изменить подключение -> интерфейс -> Изменить
Конфигурация IPv4: Автоматически -> Вручную -> Показать
Адреса -> Добавить -> IP/маска; Шлюз; Серверы DNS -> OK
nmcli connection up <интерфейс>
```
DNS на этапе настройки - DNS стенда (у преподавателя 10.39.0.1, дома - 8.8.8.8), иначе не работает `dnf`.
После того как поднят DNS на HQ-SRV - меняем на 192.168.100.10 (см. "Общий финал").

Пересылка пакетов (только ISP, HQ-RTR, BR-RTR, BR-FW):
```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

---

## 1. ISP

- [ ] ens18 (к провайдеру) - DHCP
- [ ] ens19 -> 172.16.1.1/28, ens20 -> 172.16.2.1/28
- [ ] NAT для HQ-RTR и BR-RTR

```bash
hostnamectl hostname isp.au-team.irpo
timedatectl set-timezone Europe/Moscow
```

IP (nmtui):
| Интерфейс | IPv4 | Шлюз |
|---|---|---|
| ens18 | Автоматически (DHCP) | - |
| ens19 | 172.16.1.1/28 | - |
| ens20 | 172.16.2.1/28 | - |

```bash
nmcli connection up ens19
nmcli connection up ens20

echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

NAT:
```bash
systemctl disable --now firewalld
dnf install iptables-services -y
systemctl enable --now iptables
iptables -F
iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
```

---

## 2. HQ-RTR

```bash
hostnamectl hostname hq-rtr.au-team.irpo
timedatectl set-timezone Europe/Moscow
```

### IP к ISP + пересылка
| Интерфейс | IPv4 | Шлюз | DNS |
|---|---|---|---|
| ens18 | 172.16.1.2/28 | 172.16.1.1 | DNS стенда |
| ens19 | отключено (только родитель VLAN) | - | - |

```bash
nmcli connection up ens18
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
ping -c 2 8.8.8.8          # проверка NAT на ISP
```

### Пользователь net_admin
```bash
useradd -m -U -s /bin/bash net_admin
passwd net_admin                # P@ssw0rd два раза
echo "net_admin ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
```
> Кавычки только прямые `"`. «Ёлочки» и “фигурные” из Word ломают sudoers.

### NAT
```bash
systemctl disable --now firewalld
dnf install iptables-services -y
systemctl enable --now iptables
iptables -F
iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
```

### VLAN (в отчёт!)
- [ ] VLAN 100 (HQ-SRV), 200 (HQ-CLI), 999 (управление) - на одном адаптере ens19

nmtui -> Изменить подключение -> Добавить -> VLAN:
| Имя профиля | Устройство | Родитель | VLAN ID | IPv4 |
|---|---|---|---|---|
| VLAN100 | ens19.100 | ens19 | 100 | 192.168.100.1/27 |
| VLAN200 | ens19.200 | ens19 | 200 | 192.168.200.1/28 |
| VLAN999 | ens19.999 | ens19 | 999 | 192.168.99.1/29 |

```bash
nmcli connection up VLAN100
nmcli connection up VLAN200
nmcli connection up VLAN999
ip -d link show ens19.100       # скрин в отчёт: vlan id 100
```
Для отчёта: VLAN реализованы на HQ-RTR подинтерфейсами 802.1Q на одном адаптере ens19 (router-on-a-stick); HQ-SW - мост Proxmox, передающий тегированный трафик.

### GRE (в отчёт!)
- [ ] GRE, сеть из 10.10.10.0/24, маска /30

nmtui -> Добавить -> IP-туннель:
| Поле | Значение |
|---|---|
| Имя профиля / Устройство | tun0 |
| Режим | GRE |
| Родительский | ens18 |
| Локальный IP | 172.16.1.2 |
| Удалённый IP | 172.16.2.2 |
| IPv4 (Вручную) | 10.10.10.1/30, шлюз пусто |

```bash
nmcli connection modify tun0 ip-tunnel.ttl 64
nmcli connection down tun0
nmcli connection up tun0
```
> TTL 64 обязателен: по умолчанию внешний GRE-пакет берёт TTL из внутреннего, а у OSPF он = 1 -
> пакеты умирают на ISP. Ping при этом работает, а OSPF-соседства нет.

### OSPF с MD5 (в отчёт!)
- [ ] только на туннеле, MD5, пароль P@ssw0rd

**Шаг 1. Установить FRR** (пакет с протоколами маршрутизации):
```bash
dnf install frr -y
```

**Шаг 2. Включить демон OSPF.** FRR - набор отдельных программ (демонов), по одной на протокол.
Какие запускать, задаёт файл `/etc/frr/daemons`. После установки там `ospfd=no` - надо `yes`.

Вариант А - через nano:
```bash
nano /etc/frr/daemons
```
- найти строку `ospfd=no` (`Ctrl+W`, ввести `ospfd`, `Enter`)
- исправить `no` на `yes` -> `ospfd=yes`
- сохранить `Ctrl+O`, `Enter`, выйти `Ctrl+X`

Вариант Б - одной командой:
```bash
sed -i 's/^ospfd=no/ospfd=yes/' /etc/frr/daemons
```
(`sed` - правка текста из командной строки, `-i` - сохранить в файл, `s/что/на что/` - заменить, `^` - "в начале строки")

**Шаг 3. Проверить:**
```bash
grep ospfd= /etc/frr/daemons      # ожидаем: ospfd=yes
```

**Шаг 4. Запустить FRR** (restart - чтобы перечитал daemons):
```bash
systemctl enable --now frr
systemctl restart frr
```

**Шаг 5. Настроить OSPF:**
```
vtysh
conf t
router ospf
 ospf router-id 172.16.1.2
 passive-interface default
 no passive-interface tun0
 network 10.10.10.0/30 area 0
 network 192.168.100.0/27 area 0
 network 192.168.200.0/28 area 0
 network 192.168.99.0/29 area 0
exit
interface tun0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit
do wr
end
exit
```

### DHCP для HQ-CLI (в отчёт!)
- [ ] адрес роутера исключён, шлюз - HQ-RTR, DNS - HQ-SRV, суффикс au-team.irpo

```bash
dnf install dhcp-server -y
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
dhcpd -t -cf /etc/dhcp/dhcpd.conf     # проверка конфига
systemctl enable --now dhcpd
```
> 192.168.200.1 не входит в range - адрес маршрутизатора исключён.

---

## 3. BR-RTR

```bash
hostnamectl hostname br-rtr.au-team.irpo
timedatectl set-timezone Europe/Moscow
```

### IP + пересылка
| Интерфейс | IPv4 | Шлюз | DNS |
|---|---|---|---|
| ens18 | 172.16.2.2/28 | 172.16.2.1 | DNS стенда |
| ens19 | 172.30.100.1/30 | - | - |

```bash
nmcli connection up ens18
nmcli connection up ens19
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
ping -c 2 8.8.8.8
```

### Пользователь net_admin
```bash
useradd -m -U -s /bin/bash net_admin
passwd net_admin                # P@ssw0rd два раза
echo "net_admin ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
```

### NAT
```bash
systemctl disable --now firewalld
dnf install iptables-services -y
systemctl enable --now iptables
iptables -F
iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
```

### GRE
nmtui -> Добавить -> IP-туннель:
| Поле | Значение |
|---|---|
| Имя профиля / Устройство | tun0 |
| Режим | GRE |
| Родительский | ens18 |
| Локальный IP | 172.16.2.2 |
| Удалённый IP | 172.16.1.2 |
| IPv4 (Вручную) | 10.10.10.2/30, шлюз пусто |

```bash
nmcli connection modify tun0 ip-tunnel.ttl 64
nmcli connection down tun0
nmcli connection up tun0
ping -c 2 10.10.10.1
```

### OSPF с MD5
FRR - так же, как на HQ-RTR (шаги 1-4):
```bash
dnf install frr -y
nano /etc/frr/daemons           # ospfd=no -> ospfd=yes, сохранить
grep ospfd= /etc/frr/daemons    # ожидаем: ospfd=yes
systemctl enable --now frr
systemctl restart frr
```
```
vtysh
conf t
router ospf
 ospf router-id 172.16.2.2
 passive-interface default
 no passive-interface tun0
 no passive-interface ens19
 network 10.10.10.0/30 area 0
 network 172.30.100.0/30 area 0
exit
interface tun0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit
do wr
end
exit
```
> ens19 (к BR-FW) не пассивный - иначе BR-FW не станет соседом и сеть BR-SRV не попадёт в HQ.
> Аутентификация только на tun0 (задание требует её между HQ-RTR и BR-RTR).

---

## 4. BR-FW

- [ ] в OSPF только интерфейс к BR-RTR

```bash
hostnamectl hostname br-fw.au-team.irpo
timedatectl set-timezone Europe/Moscow
```

### IP + пересылка
| Интерфейс | IPv4 | Шлюз | DNS |
|---|---|---|---|
| ens18 | 172.30.100.2/30 | 172.30.100.1 | DNS стенда |
| ens19 | 172.30.200.1/28 | - | - |

```bash
nmcli connection up ens18
nmcli connection up ens19
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
systemctl disable --now firewalld     # чтобы не резал форвард и OSPF
ping -c 2 8.8.8.8
```
> NAT на BR-FW не нужен - его делает BR-RTR.

### OSPF (без аутентификации)
```bash
dnf install frr -y
nano /etc/frr/daemons           # ospfd=no -> ospfd=yes, сохранить
grep ospfd= /etc/frr/daemons    # ожидаем: ospfd=yes
systemctl enable --now frr
systemctl restart frr
```
```
vtysh
conf t
router ospf
 ospf router-id 172.30.100.2
 passive-interface default
 no passive-interface ens18
 network 172.30.100.0/30 area 0
 network 172.30.200.0/28 area 0
exit
do wr
end
exit
```

Проверка OSPF (на HQ-RTR, BR-RTR, BR-FW):
```bash
vtysh -c "show ip ospf neighbor"                         # Full
vtysh -c "show ip route ospf"                            # на HQ-RTR есть 172.30.200.0/28
vtysh -c "show ip ospf interface tun0" | grep -i auth    # в отчёт: MD5 (на роутерах)
```

---

## 5. HQ-SRV

```bash
hostnamectl hostname hq-srv.au-team.irpo
timedatectl set-timezone Europe/Moscow
systemctl disable --now firewalld     # по заданию не нужен, иначе режет SSH 2027 и DNS
```

### VLAN100
nmtui -> Добавить -> VLAN:
| Имя профиля | Устройство | Родитель | VLAN ID | IPv4 | Шлюз | DNS |
|---|---|---|---|---|---|---|
| VLAN100 | ens18.100 | ens18 | 100 | 192.168.100.10/27 | 192.168.100.1 | DNS стенда |

У самого ens18 IPv4 - отключено.
```bash
nmcli connection up VLAN100
ping -c 2 8.8.8.8
```

### Пользователь sshuser
```bash
useradd -m -U -s /bin/bash -u 2027 sshuser
passwd sshuser                  # P@ssw0rd два раза
echo "sshuser ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
id sshuser                      # uid=2027
```

### SSH
- [ ] порт 2027, только sshuser, 2 попытки, баннер «Authorized access only»

```bash
echo "Authorized access only" > /etc/ssh/banner.txt
nano /etc/ssh/sshd_config
```
```
Port 2027
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/ssh/banner.txt
```
```bash
semanage port -a -t ssh_port_t -p tcp 2027
# если semanage нет: dnf install policycoreutils-python-utils -y
sshd -t                          # нет вывода = конфиг без ошибок
systemctl restart sshd
```
> Запасной вариант из тетради: в `/etc/selinux/config` поставить `SELINUX=disabled` и `reboot`.

Проверка с HQ-RTR: `ssh -l sshuser 192.168.100.10 -p 2027`

### DNS (bind)
- [ ] основной DNS, прямые и обратные записи по таблице, пересылка на общедоступный DNS

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
dnf install bind -y
nano /etc/named.conf
```
В блоке `options` изменить четыре строки:
```
listen-on port 53 { any; };
allow-query     { any; };
forwarders      { 77.88.8.7; };
dnssec-validation no;
```
> Без `dnssec-validation no` внешние имена (yandex.ru и т.п.) не резолвятся: в журнале `broken trust chain`, `dig @127.0.0.1` - timed out / SERVFAIL.
В конце файла должно получиться так (строки `include` там уже есть - их не дублировать, зоны вставить перед ними):
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

Прямая зона:
```bash
mkdir /opt/dns
cd /opt/dns
cp /var/named/named.empty au-team.irpo
nano au-team.irpo
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

Обратная зона HQ:
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

Обратная зона BR:
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

Запуск:
```bash
chown -R named:named /opt/dns/
named-checkconf -z            # проверить, исправить ошибки если есть
systemctl enable --now named
systemctl restart named
nslookup hq-srv.au-team.irpo 127.0.0.1
```
> Если в `journalctl -u named` есть `permission denied` на файлы зон - это SELinux:
> `semanage fcontext -a -t named_zone_t "/opt/dns(/.*)?" && restorecon -Rv /opt/dns && systemctl restart named`

> hq-cli = 192.168.200.2 - первый адрес пула. Если HQ-CLI получил другой - поправить A-запись.

---

## 6. BR-SRV

```bash
hostnamectl hostname br-srv.au-team.irpo
timedatectl set-timezone Europe/Moscow
```

### IP
| Интерфейс | IPv4 | Шлюз | DNS |
|---|---|---|---|
| ens18 | 172.30.200.10/28 | 172.30.200.1 | DNS стенда |

```bash
nmcli connection up ens18
ping -c 2 8.8.8.8
```

### Пользователь sshuser
```bash
useradd -m -U -s /bin/bash -u 2027 sshuser
passwd sshuser                  # P@ssw0rd два раза
echo "sshuser ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
id sshuser                      # uid=2027
```

### SSH
```bash
echo "Authorized access only" > /etc/ssh/banner.txt
nano /etc/ssh/sshd_config
```
```
Port 2027
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/ssh/banner.txt
```
```bash
semanage port -a -t ssh_port_t -p tcp 2027
firewall-cmd --permanent --add-port=2027/tcp && firewall-cmd --reload   # только если firewalld активен
sshd -t
systemctl restart sshd
```
Проверка с BR-FW: `ssh -l sshuser 172.30.200.10 -p 2027`

---

## 7. HQ-CLI

```bash
hostnamectl hostname hq-cli.au-team.irpo
timedatectl set-timezone Europe/Moscow
```

### VLAN200 по DHCP
nmtui -> Добавить -> VLAN:
| Имя профиля | Устройство | Родитель | VLAN ID | IPv4 |
|---|---|---|---|---|
| VLAN200 | ens18.200 | ens18 | 200 | Автоматически (DHCP), DNS вручную НЕ задавать |

У самого ens18 IPv4 - отключено.
```bash
nmcli connection up VLAN200
ip a show ens18.200            # 192.168.200.x/28
cat /etc/resolv.conf           # nameserver 192.168.100.10, search au-team.irpo
```

Проверка на HQ-RTR (в отчёт по DHCP):
```bash
cat /var/lib/dhcpd/dhcpd.leases     # запись о выдаче адреса HQ-CLI
```

---

## 8. Общий финал - DNS на всех

На всех машинах со статикой (ISP не трогаем; HQ-RTR, BR-RTR, BR-FW, HQ-SRV, BR-SRV):
nmtui -> поменять DNS со "DNS стенда" на **192.168.100.10**, домен поиска **au-team.irpo** -> OK ->
Активировать подключение -> выключить и включить интерфейс.

---

## Итоговая проверка - с HQ-CLI

```bash
ping -c 2 hq-rtr
ping -c 2 hq-srv
ping -c 2 br-rtr
ping -c 2 br-fw
ping -c 2 br-srv
ping -c 2 ya.ru
nslookup br-srv.au-team.irpo
nslookup 172.30.200.10
```

---

## Быстрая диагностика

| Симптом | Причина | Проверка |
|---|---|---|
| dnf не качает | нет DNS на интерфейсе или нет NAT на ISP/роутере | `cat /etc/resolv.conf`, `ping 8.8.8.8` |
| sudoers/sshd ругается на синтаксис | кавычки или символы из Word | `visudo -c`, `sshd -t`, `cat -A файл` |
| sshd не стартует на 2027 | нет semanage port | `semanage port -l \| grep ssh` |
| ping через tun0 есть, OSPF-соседа нет | ttl туннеля не 64 / не совпал MD5-ключ | `ip -d link show tun0`, `show ip ospf neighbor` |
| HQ не видит BR-SRV | ens19 на BR-RTR пассивный или нет network 172.30.100.0/30 | `vtysh -c "show ip route"` на HQ-RTR |
| BR-SRV без интернета | нет шлюза 172.30.100.1 на BR-FW | `ip route` на BR-FW |
| NAT не работает | опечатка в имени интерфейса, firewalld не выключен | `iptables -t nat -L -n -v`, `systemctl is-active firewalld` |
| named не стартует | ошибка в зоне / дубль include / SELinux | `named-checkconf -z`, `journalctl -u named` |
| HQ-CLI без адреса | dhcpd не запущен или VLAN200 не поднят | `systemctl status dhcpd`, `ip a` |
