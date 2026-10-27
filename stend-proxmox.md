# Стенд на Proxmox

## Хост
- PVE 9.2, адрес `192.168.1.204`, вход: `ssh root@192.168.1.204` (по ключу с мака)
- CPU: Ryzen 7 7800X3D, 16 потоков; RAM: 30 ГБ
- Хранилища:

| Хранилище | Тип | Диск | Размер | Для чего |
|---|---|---|---|---|
| `name-disk` | lvmthin | NVMe | 136 ГБ | диски ВМ стенда |
| `local-lvm` | lvmthin | SSD | 137 ГБ | LXC 100 samba, запас |
| `local` | dir | SSD | ~22 ГБ свободно | ISO, бэкапы |

- ISO: `redos-8-20260717.0-Everything-x86_64-DVD1.iso`

## Мосты

| Мост | Сеть | Кто подключён |
|---|---|---|
| vmbr0 | интернет (домашняя сеть) | ISP |
| vmbr1 | ISP - HQ-RTR | ISP, HQ-RTR |
| vmbr2 | ISP - BR-RTR | ISP, BR-RTR |
| vmbr3 | HQ-SW (VLAN 100/200/999) | HQ-RTR, HQ-SRV, HQ-CLI |
| vmbr4 | FW-NET | BR-RTR, BR-FW |
| vmbr5 | BR-NET | BR-FW, BR-SRV |

vmbr10-13 - старые, не используются.

## ВМ

| ID | ВМ | vCPU | RAM | Диск | ens18 | ens19 | ens20 | ОС |
|---|---|---|---|---|---|---|---|---|
| 101 | ISP | 1 | 2 ГБ | 15 ГБ | vmbr0 | vmbr1 | vmbr2 | минимальная |
| 102 | HQ-RTR | 1 | 2 ГБ | 15 ГБ | vmbr1 | vmbr3 | - | минимальная |
| 103 | BR-RTR | 1 | 2 ГБ | 15 ГБ | vmbr2 | vmbr4 | - | минимальная |
| 104 | BR-FW | 1 | 2 ГБ | 15 ГБ | vmbr4 | vmbr5 | - | минимальная |
| 105 | HQ-SRV | 2 | 3 ГБ | 20 ГБ + 2×1 ГБ | vmbr3 | - | - | минимальная |
| 106 | BR-SRV | 2 | 4 ГБ | 20 ГБ | vmbr5 | - | - | минимальная |
| 107 | HQ-CLI | 4 | 4 ГБ | 40 ГБ | vmbr3 | - | - | с графикой |
| | **Итого** | **12/16** | **19/30 ГБ** | | | | | |

Общее для всех: CPU type host, VirtIO SCSI single, discard=on, сетевые карты virtio, загрузка: диск, потом CD.

- HQ-SRV: два диска по 1 ГБ (sdb, sdc) - под RAID0 в модуле 2. ОС ставить только на sda.
- Роутерам после установки можно урезать RAM до 1 ГБ.

## Установка ОС
1. Включить ВМ -> Console -> установщик RED OS.
2. Роутеры и серверы - минимальная установка, HQ-CLI - с графикой.
3. Задать пароль root, сеть в установщике не настраивать (это часть задания).
4. После установки выключить ВМ.

## Снапшоты (тренировка)

Точки: `clean` - чистая ОС; `m1done` - сделан модуль 1; `m2done` - сделан модуль 2.

Команды на хосте (Shell в Proxmox или по ssh):
```bash
IDS="101 102 103 104 105 106 107"

# снять снапшот со всех (лучше при выключенных ВМ)
for id in $IDS; do qm snapshot $id clean; done

# откатить все к снапшоту
for id in $IDS; do qm rollback $id clean; done

# список снапшотов одной ВМ
qm listsnapshot 101

# удалить снапшот
qm delsnapshot 101 m1done
```
В веб-интерфейсе: ВМ -> Snapshots -> Take Snapshot / Rollback.

Следить, чтобы пул `name-disk` не заполнялся больше 80%:
```bash
lvs vmdata/thin    # колонка Data%
```

## Бэкап (один раз, страховка)
```bash
vzdump $IDS --mode stop --compress zstd --storage local
# восстановить одну ВМ
qmrestore /var/lib/vz/dump/vzdump-qemu-101-*.vma.zst 101 --force
```
В `local` мало места (~22 ГБ). В `/var/lib/vz/images` лежат 22 ГБ старых vmdk Windows Server 2016 - можно удалить, если нужно место.
