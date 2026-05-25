# Задания 1–2: команды поэтапно

Источник задания: файл `задание.md`.

Ниже команды **только для заданий 1 и 2**:

- `isp` — `172.16.0.1`
- `dc` — `172.16.0.10`
- `srv` — `172.16.0.20`
- `cli` — адрес из DHCP
- сеть — `172.16.0.0/24`
- шлюз — `172.16.0.1`
- DNS-суффикс — `lab.local`
- пароль для пользователей — `P@ssw0rd`

Команды рассчитаны на **Debian/Ubuntu Server**. Все команды вводить **от root** или через `sudo`.

---

# Задание 1. Подготовка виртуального стенда и базовая маршрутизация

## 0. Настройка ВМ в Proxmox

Перед вводом команд в Proxmox нужно настроить сетевые адаптеры:

| Машина | Сетевые адаптеры |
|---|---|
| `isp` | 2 адаптера: `WAN` во внешний мост, `LAN` во внутренний мост |
| `dc` | 1 адаптер во внутренний мост |
| `srv` | 1 адаптер во внутренний мост |
| `cli` | 1 адаптер во внутренний мост |

На `isp` один интерфейс должен смотреть в интернет, второй — в локальную сеть `172.16.0.0/24`.

---

## 1.1. Посмотреть имена сетевых интерфейсов

### Вводить на каждой машине

```bash
ip -br a
```

На `isp` ты увидишь примерно два интерфейса:

```text
ens18
ens19
```

Допустим:

```text
ens18 = WAN
ens19 = LAN
```

У тебя имена могут быть другие, например `enp0s3`, `enp0s8`.

---

## 1.2. Настроить hostname

## На `isp`

```bash
hostnamectl set-hostname isp.lab.local
```

Открыть файл:

```bash
nano /etc/hosts
```

Добавить или исправить строки:

```text
127.0.0.1 localhost
172.16.0.1 isp.lab.local isp
```

Проверка:

```bash
hostname
hostname -f
```

---

## На `dc`

```bash
hostnamectl set-hostname dc.lab.local
```

```bash
nano /etc/hosts
```

Добавить:

```text
127.0.0.1 localhost
172.16.0.10 dc.lab.local dc
```

Проверка:

```bash
hostname
hostname -f
```

---

## На `srv`

```bash
hostnamectl set-hostname srv.lab.local
```

```bash
nano /etc/hosts
```

Добавить:

```text
127.0.0.1 localhost
172.16.0.20 srv.lab.local srv
```

Проверка:

```bash
hostname
hostname -f
```

---

## На `cli`

```bash
hostnamectl set-hostname cli.lab.local
```

```bash
nano /etc/hosts
```

Добавить:

```text
127.0.0.1 localhost
172.16.0.200 cli.lab.local cli
```

Проверка:

```bash
hostname
hostname -f
```

---

## 1.3. Настроить сеть на `isp`

### Вводить на `isp`

Сначала узнать имена интерфейсов:

```bash
ip -br a
```

Дальше в примере используется:

```text
WAN = ens18
LAN = ens19
```

Если у тебя другие имена — замени их в командах.

Открыть netplan:

```bash
ls /etc/netplan
```

Обычно файл называется примерно так:

```text
00-installer-config.yaml
```

Открыть его:

```bash
nano /etc/netplan/00-installer-config.yaml
```

Вставить конфиг, заменив `ens18` и `ens19` на свои интерфейсы:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      dhcp4: true
    ens19:
      dhcp4: false
      addresses:
        - 172.16.0.1/24
```

Применить:

```bash
netplan apply
```

Проверить:

```bash
ip -br a
ip route
```

У `isp` должно быть:

```text
LAN: 172.16.0.1/24
WAN: адрес от внешнего DHCP
```

Проверить интернет с `isp`:

```bash
ping -c 4 8.8.8.8
```

---

## 1.4. Включить IP-forwarding на `isp`

### Вводить на `isp`

```bash
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-ip-forward.conf
sysctl -p /etc/sysctl.d/99-ip-forward.conf
```

Проверка:

```bash
sysctl net.ipv4.ip_forward
```

Должно быть:

```text
net.ipv4.ip_forward = 1
```

---

## 1.5. Настроить NAT на `isp`

### Вводить на `isp`

Установить пакеты:

```bash
apt update
apt install -y iptables iptables-persistent netfilter-persistent
```

Добавить NAT.

Здесь:

- `ens18` — WAN
- `ens19` — LAN

Замени на свои интерфейсы, если они называются иначе.

```bash
iptables -t nat -A POSTROUTING -s 172.16.0.0/24 -o ens18 -j MASQUERADE
iptables -A FORWARD -i ens19 -o ens18 -s 172.16.0.0/24 -j ACCEPT
iptables -A FORWARD -i ens18 -o ens19 -m state --state ESTABLISHED,RELATED -j ACCEPT
```

Сохранить правила:

```bash
netfilter-persistent save
systemctl enable netfilter-persistent
```

Проверка NAT:

```bash
iptables -t nat -L -n -v
iptables -L FORWARD -n -v
```

---

## 1.6. Временно настроить `dc`, `srv`, `cli` на DHCP

Потом во 2 задании `isp` будет выдавать им адреса через DHCP.

---

## На `dc`

Посмотреть имя интерфейса:

```bash
ip -br a
```

Открыть netplan:

```bash
nano /etc/netplan/00-installer-config.yaml
```

Пример, если интерфейс `ens18`:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      dhcp4: true
```

Применить:

```bash
netplan apply
```

---

## На `srv`

```bash
ip -br a
nano /etc/netplan/00-installer-config.yaml
```

Пример:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      dhcp4: true
```

```bash
netplan apply
```

---

## На `cli`

```bash
ip -br a
nano /etc/netplan/00-installer-config.yaml
```

Пример:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      dhcp4: true
```

```bash
netplan apply
```

---

# Задание 2. DHCP, SSH, sudo и локальные пользователи

## 2.1. Узнать MAC-адреса `dc` и `srv`

Они нужны для статической DHCP-привязки.

---

## На `dc`

```bash
ip link
```

Найти строку вида:

```text
link/ether 52:54:00:aa:bb:cc
```

Записать MAC-адрес.

---

## На `srv`

```bash
ip link
```

Записать MAC-адрес.

---

## 2.2. Установить DHCP-сервер на `isp`

### Вводить на `isp`

```bash
apt update
apt install -y isc-dhcp-server
```

Указать, на каком интерфейсе DHCP должен работать.

Открыть файл:

```bash
nano /etc/default/isc-dhcp-server
```

Найти строку:

```text
INTERFACESv4=""
```

Заменить на LAN-интерфейс `isp`.

Например, если LAN — `ens19`:

```text
INTERFACESv4="ens19"
```

---

## 2.3. Настроить DHCP на `isp`

### Вводить на `isp`

Открыть конфиг:

```bash
nano /etc/dhcp/dhcpd.conf
```

Лучше очистить файл и вставить полностью такой конфиг.

Замени:

```text
MAC_DC
MAC_SRV
```

на реальные MAC-адреса `dc` и `srv`.

```conf
authoritative;

default-lease-time 600;
max-lease-time 7200;

option domain-name "lab.local";
option domain-name-servers 172.16.0.10, 8.8.8.8;

subnet 172.16.0.0 netmask 255.255.255.0 {
    range 172.16.0.200 172.16.0.250;
    option routers 172.16.0.1;
    option subnet-mask 255.255.255.0;
    option broadcast-address 172.16.0.255;
    option domain-name "lab.local";
}

host dc {
    hardware ethernet MAC_DC;
    fixed-address 172.16.0.10;
    option host-name "dc";
}

host srv {
    hardware ethernet MAC_SRV;
    fixed-address 172.16.0.20;
    option host-name "srv";
}
```

Пример с MAC:

```conf
host dc {
    hardware ethernet 52:54:00:aa:bb:cc;
    fixed-address 172.16.0.10;
    option host-name "dc";
}
```

Проверить конфиг:

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

Перезапустить DHCP:

```bash
systemctl restart isc-dhcp-server
systemctl enable isc-dhcp-server
```

Проверить статус:

```bash
systemctl status isc-dhcp-server
```

---

## 2.4. Получить адреса по DHCP на `dc`, `srv`, `cli`

## На `dc`

```bash
netplan apply
dhclient -r
dhclient -v
ip -br a
ip route
```

Должен получить:

```text
172.16.0.10/24
```

Проверка:

```bash
ping -c 4 172.16.0.1
ping -c 4 8.8.8.8
```

---

## На `srv`

```bash
netplan apply
dhclient -r
dhclient -v
ip -br a
ip route
```

Должен получить:

```text
172.16.0.20/24
```

Проверка:

```bash
ping -c 4 172.16.0.1
ping -c 4 8.8.8.8
```

---

## На `cli`

```bash
netplan apply
dhclient -r
dhclient -v
ip -br a
ip route
```

Должен получить адрес из диапазона:

```text
172.16.0.200-172.16.0.250
```

Проверка:

```bash
ping -c 4 172.16.0.1
ping -c 4 172.16.0.10
ping -c 4 172.16.0.20
ping -c 4 8.8.8.8
```

---

## 2.5. Создать пользователей `admin` и `monitor`

Это надо сделать **на всех четырёх узлах**:

- `isp`
- `dc`
- `srv`
- `cli`

### Вводить на каждой машине

```bash
useradd -m -s /bin/bash admin
echo 'admin:P@ssw0rd' | chpasswd

useradd -m -s /bin/bash monitor
echo 'monitor:P@ssw0rd' | chpasswd
```

Проверка:

```bash
id admin
id monitor
```

---

## 2.6. Настроить sudo

### Вводить на каждой машине: `isp`, `dc`, `srv`, `cli`

Установить sudo и htop:

```bash
apt install -y sudo htop
```

Создать отдельный sudoers-файл:

```bash
nano /etc/sudoers.d/lab-users
```

Вставить туда:

```sudoers
admin ALL=(ALL) NOPASSWD: ALL

Cmnd_Alias MONITOR_CMDS = /usr/bin/htop, /usr/bin/df, /usr/bin/free, /usr/bin/journalctl, /bin/systemctl status *, /usr/bin/systemctl status *

monitor ALL=(ALL) NOPASSWD: MONITOR_CMDS
```

Выставить права:

```bash
chmod 440 /etc/sudoers.d/lab-users
```

Проверить синтаксис:

```bash
visudo -cf /etc/sudoers.d/lab-users
```

Проверка для `admin`:

```bash
su - admin
sudo whoami
exit
```

Должно вывести:

```text
root
```

Проверка для `monitor`:

```bash
su - monitor
sudo df -h
sudo free -h
sudo systemctl status ssh
exit
```

Проверка, что лишнее запрещено:

```bash
su - monitor
sudo apt update
exit
```

Команда должна быть запрещена.

---

## 2.7. Настроить SSH на порт `2222`

Это делается **на всех четырёх узлах**.

### Вводить на каждой машине: `isp`, `dc`, `srv`, `cli`

Установить SSH-сервер:

```bash
apt install -y openssh-server
```

Создать баннер:

```bash
echo "Authorized access only" > /etc/issue.net
```

Создать файл настроек SSH:

```bash
nano /etc/ssh/sshd_config.d/99-lab-hardening.conf
```

Вставить:

```sshconfig
Port 2222
Banner /etc/issue.net
MaxAuthTries 2
PermitRootLogin no
AllowUsers admin monitor
PasswordAuthentication yes
```

Проверить конфиг:

```bash
sshd -t
```

Перезапустить SSH:

```bash
systemctl restart ssh
```

Если команда не сработала, попробовать:

```bash
systemctl restart sshd
```

Проверить порт:

```bash
ss -tulpn | grep 2222
```

Должно быть видно, что SSH слушает порт `2222`.

---

## 2.8. Проверить SSH с `cli`

### Вводить на `cli`

Подключение к `isp`:

```bash
ssh -p 2222 admin@172.16.0.1
```

Подключение к `dc`:

```bash
ssh -p 2222 admin@172.16.0.10
```

Подключение к `srv`:

```bash
ssh -p 2222 admin@172.16.0.20
```

Проверка пользователя `monitor`:

```bash
ssh -p 2222 monitor@172.16.0.1
ssh -p 2222 monitor@172.16.0.10
ssh -p 2222 monitor@172.16.0.20
```

Проверка, что `root` запрещён:

```bash
ssh -p 2222 root@172.16.0.1
```

Доступ должен быть запрещён.

---

## 2.9. Итоговые проверки для отчёта

## На `isp`

```bash
hostname -f
ip -br a
sysctl net.ipv4.ip_forward
iptables -t nat -L -n -v
systemctl status isc-dhcp-server
```

---

## На `dc`

```bash
hostname -f
ip -br a
ip route
ping -c 4 172.16.0.1
ping -c 4 8.8.8.8
```

---

## На `srv`

```bash
hostname -f
ip -br a
ip route
ping -c 4 172.16.0.1
ping -c 4 8.8.8.8
```

---

## На `cli`

```bash
hostname -f
ip -br a
ip route
ping -c 4 172.16.0.1
ping -c 4 172.16.0.10
ping -c 4 172.16.0.20
ping -c 4 8.8.8.8
ssh -p 2222 admin@172.16.0.1
ssh -p 2222 admin@172.16.0.10
ssh -p 2222 admin@172.16.0.20
```

---

# Что обязательно заскринить для отчёта

1. На `isp`: `ip -br a`, где видно `172.16.0.1`.
2. На `isp`: `sysctl net.ipv4.ip_forward`.
3. На `isp`: `iptables -t nat -L -n -v`.
4. На `isp`: `systemctl status isc-dhcp-server`.
5. На `dc`: `ip -br a`, где видно `172.16.0.10`.
6. На `srv`: `ip -br a`, где видно `172.16.0.20`.
7. На `cli`: адрес из диапазона `172.16.0.200-172.16.0.250`.
8. На `cli`: успешный `ping` до `isp`, `dc`, `srv` и `8.8.8.8`.
9. На `cli`: успешный вход по SSH на порт `2222`.
10. Проверку, что `root` по SSH не входит.
