# Команды для выполнения заданий 1–2 на ALT Linux

Файл подготовлен под стенд, где **все машины работают на ALT Linux**.

Используемые узлы:

| Машина | FQDN | IP-адрес | Назначение |
|---|---|---:|---|
| `isp` | `isp.lab.local` | `172.16.0.1` | шлюз, NAT, DHCP, SSH |
| `dc` | `dc.lab.local` | `172.16.0.10` | сервер, получает IP по DHCP-привязке |
| `srv` | `srv.lab.local` | `172.16.0.20` | сервер, получает IP по DHCP-привязке |
| `cli` | `cli.lab.local` | DHCP | рабочая станция |

Пароль для пользователей по заданию:

```text
P@ssw0rd
```

> Все команды ниже вводятся от `root`. Если работаешь не под `root`, перед командами добавляй `sudo`.

---

# Задание 1. Подготовка виртуального стенда и базовая маршрутизация

## 0. Что сделать в Proxmox до настройки ОС

В настройках сетевых адаптеров ВМ:

| Машина | Сетевые адаптеры |
|---|---|
| `isp` | 2 адаптера: один во внешний мост/WAN, второй во внутренний мост/LAN |
| `dc` | 1 адаптер во внутренний мост |
| `srv` | 1 адаптер во внутренний мост |
| `cli` | 1 адаптер во внутренний мост |

На `isp` будет две сетевые карты:

- `WAN` — получает адрес от внешнего DHCP;
- `LAN` — статический адрес `172.16.0.1/24`.

---

## 1.1. Узнать имена сетевых интерфейсов

### Вводить на каждой машине

```bash
ip -br a
```

Или:

```bash
ip link
```

На `isp` должно быть два интерфейса, например:

```text
enp0s3
enp0s8
```

Дальше в примерах будет так:

```text
WAN = enp0s3
LAN = enp0s8
```

Если у тебя интерфейсы называются иначе, например `ens18`, `ens19`, `eth0`, `eth1`, то в командах меняй имена на свои.

---

## 1.2. Настроить hostname

## На `isp`

```bash
hostnamectl set-hostname isp.lab.local
```

Открыть файл:

```bash
mcedit /etc/hosts
```

Если `mcedit` не установлен, используй `nano` или `vim`:

```bash
nano /etc/hosts
```

Добавить или исправить строки:

```text
127.0.0.1 localhost
172.16.0.1 isp.lab.local isp
```

Проверить:

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
mcedit /etc/hosts
```

Добавить:

```text
127.0.0.1 localhost
172.16.0.10 dc.lab.local dc
```

Проверить:

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
mcedit /etc/hosts
```

Добавить:

```text
127.0.0.1 localhost
172.16.0.20 srv.lab.local srv
```

Проверить:

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
mcedit /etc/hosts
```

Добавить:

```text
127.0.0.1 localhost
172.16.0.200 cli.lab.local cli
```

Проверить:

```bash
hostname
hostname -f
```

---

## 1.3. Настроить сеть на `isp` через etcnet

В ALT Linux обычно используется настройка сети через каталог:

```text
/etc/net/ifaces/
```

Сначала посмотри интерфейсы:

```bash
ip -br a
```

Дальше пример:

```text
WAN = enp0s3
LAN = enp0s8
```

### Настройка WAN-интерфейса на DHCP

### Вводить на `isp`

Создать каталог для WAN-интерфейса:

```bash
mkdir -p /etc/net/ifaces/enp0s3
```

Открыть файл настроек:

```bash
mcedit /etc/net/ifaces/enp0s3/options
```

Вставить:

```text
TYPE=eth
BOOTPROTO=dhcp
ONBOOT=yes
CONFIG_IPV4=yes
DISABLED=no
```

---

### Настройка LAN-интерфейса со статическим IP

Создать каталог для LAN-интерфейса:

```bash
mkdir -p /etc/net/ifaces/enp0s8
```

Открыть файл:

```bash
mcedit /etc/net/ifaces/enp0s8/options
```

Вставить:

```text
TYPE=eth
BOOTPROTO=static
ONBOOT=yes
CONFIG_IPV4=yes
DISABLED=no
```

Создать файл IPv4-адреса:

```bash
mcedit /etc/net/ifaces/enp0s8/ipv4address
```

Вставить:

```text
172.16.0.1/24
```

Перезапустить сеть:

```bash
systemctl restart network
```

Если служба называется иначе, проверь:

```bash
systemctl list-units --type=service | grep -E 'network|etcnet'
```

Проверить адреса:

```bash
ip -br a
ip route
```

Проверить интернет с `isp`:

```bash
ping -c 4 8.8.8.8
```

---

## 1.4. Включить IP-forwarding на `isp`

### Вводить на `isp`

Создать файл:

```bash
mcedit /etc/sysctl.d/99-ip-forward.conf
```

Вставить:

```text
net.ipv4.ip_forward = 1
```

Применить:

```bash
sysctl -p /etc/sysctl.d/99-ip-forward.conf
```

Проверить:

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

Установить нужные пакеты:

```bash
apt-get update
apt-get install -y iptables iptables-save
```

Если пакет `iptables-save` отдельно не находится, установи только `iptables`:

```bash
apt-get install -y iptables
```

Добавить правила NAT.

В примере:

```text
WAN = enp0s3
LAN = enp0s8
```

Команды:

```bash
iptables -t nat -A POSTROUTING -s 172.16.0.0/24 -o enp0s3 -j MASQUERADE
iptables -A FORWARD -i enp0s8 -o enp0s3 -s 172.16.0.0/24 -j ACCEPT
iptables -A FORWARD -i enp0s3 -o enp0s8 -m state --state ESTABLISHED,RELATED -j ACCEPT
```

Проверить:

```bash
iptables -t nat -L -n -v
iptables -L FORWARD -n -v
```

Сохранить правила:

```bash
mkdir -p /etc/sysconfig
iptables-save > /etc/sysconfig/iptables
```

Создать systemd-службу для автозагрузки правил:

```bash
mcedit /etc/systemd/system/iptables-restore.service
```

Вставить:

```ini
[Unit]
Description=Restore iptables rules
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/iptables-restore /etc/sysconfig/iptables
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Включить автозагрузку:

```bash
systemctl daemon-reload
systemctl enable iptables-restore.service
systemctl start iptables-restore.service
```

Проверить статус:

```bash
systemctl status iptables-restore.service
```

---

## 1.6. Настроить `dc`, `srv`, `cli` на получение адреса по DHCP

Для задания 2 сервер `isp` будет выдавать адреса:

- `dc` — `172.16.0.10` по MAC-привязке;
- `srv` — `172.16.0.20` по MAC-привязке;
- `cli` — из диапазона `172.16.0.200-172.16.0.250`.

Сначала можно настроить все три машины на DHCP.

---

## На `dc`

Посмотреть имя интерфейса:

```bash
ip -br a
```

Допустим интерфейс называется `enp0s3`.

Создать каталог:

```bash
mkdir -p /etc/net/ifaces/enp0s3
```

Открыть файл:

```bash
mcedit /etc/net/ifaces/enp0s3/options
```

Вставить:

```text
TYPE=eth
BOOTPROTO=dhcp
ONBOOT=yes
CONFIG_IPV4=yes
DISABLED=no
```

Перезапустить сеть:

```bash
systemctl restart network
```

Проверить:

```bash
ip -br a
ip route
```

---

## На `srv`

Посмотреть имя интерфейса:

```bash
ip -br a
```

Создать каталог:

```bash
mkdir -p /etc/net/ifaces/enp0s3
```

Открыть файл:

```bash
mcedit /etc/net/ifaces/enp0s3/options
```

Вставить:

```text
TYPE=eth
BOOTPROTO=dhcp
ONBOOT=yes
CONFIG_IPV4=yes
DISABLED=no
```

Перезапустить сеть:

```bash
systemctl restart network
```

Проверить:

```bash
ip -br a
ip route
```

---

## На `cli`

Посмотреть имя интерфейса:

```bash
ip -br a
```

Создать каталог:

```bash
mkdir -p /etc/net/ifaces/enp0s3
```

Открыть файл:

```bash
mcedit /etc/net/ifaces/enp0s3/options
```

Вставить:

```text
TYPE=eth
BOOTPROTO=dhcp
ONBOOT=yes
CONFIG_IPV4=yes
DISABLED=no
```

Перезапустить сеть:

```bash
systemctl restart network
```

Проверить:

```bash
ip -br a
ip route
```

---

# Задание 2. DHCP, SSH, sudo и локальные пользователи

---

## 2.1. Узнать MAC-адреса `dc` и `srv`

MAC-адреса нужны для DHCP-привязок.

---

## На `dc`

```bash
ip link
```

Найди строку вида:

```text
link/ether 52:54:00:aa:bb:cc
```

Запиши MAC-адрес `dc`.

---

## На `srv`

```bash
ip link
```

Найди строку вида:

```text
link/ether 52:54:00:dd:ee:ff
```

Запиши MAC-адрес `srv`.

---

## 2.2. Установить DHCP-сервер на `isp`

### Вводить на `isp`

Установить пакет DHCP-сервера:

```bash
apt-get update
apt-get install -y dhcp-server
```

Если пакет не найден, попробуй:

```bash
apt-cache search dhcp | grep server
```

Чаще всего в ALT Linux используется пакет `dhcp-server`.

---

## 2.3. Указать интерфейс DHCP-сервера

### Вводить на `isp`

Открыть файл:

```bash
mcedit /etc/sysconfig/dhcpd
```

Если файла нет, создай его этой же командой.

Вставить:

```text
DHCPDARGS=enp0s8
```

Где `enp0s8` — это LAN-интерфейс `isp`, который смотрит во внутреннюю сеть.

Если у тебя LAN-интерфейс называется иначе, например `ens19`, пиши его:

```text
DHCPDARGS=ens19
```

---

## 2.4. Настроить DHCP на `isp`

### Вводить на `isp`

Открыть конфиг:

```bash
mcedit /etc/dhcp/dhcpd.conf
```

Если каталога `/etc/dhcp` нет, проверь расположение файла:

```bash
find /etc -name 'dhcpd.conf'
```

В файл `/etc/dhcp/dhcpd.conf` вставить конфиг.

Внимание: замени `MAC_DC` и `MAC_SRV` на реальные MAC-адреса.

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

Пример с настоящим MAC:

```conf
host dc {
    hardware ethernet 52:54:00:aa:bb:cc;
    fixed-address 172.16.0.10;
    option host-name "dc";
}
```

Проверить синтаксис:

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

Запустить DHCP-сервер:

```bash
systemctl enable dhcpd
systemctl restart dhcpd
```

Проверить статус:

```bash
systemctl status dhcpd
```

Если служба называется не `dhcpd`, найди её:

```bash
systemctl list-unit-files | grep -i dhcp
```

---

## 2.5. Получить адреса по DHCP на `dc`, `srv`, `cli`

---

## На `dc`

```bash
systemctl restart network
```

Проверить адрес:

```bash
ip -br a
ip route
```

Должен быть адрес:

```text
172.16.0.10/24
```

Проверить связь:

```bash
ping -c 4 172.16.0.1
ping -c 4 8.8.8.8
```

---

## На `srv`

```bash
systemctl restart network
```

Проверить адрес:

```bash
ip -br a
ip route
```

Должен быть адрес:

```text
172.16.0.20/24
```

Проверить связь:

```bash
ping -c 4 172.16.0.1
ping -c 4 8.8.8.8
```

---

## На `cli`

```bash
systemctl restart network
```

Проверить адрес:

```bash
ip -br a
ip route
```

Должен быть адрес из диапазона:

```text
172.16.0.200-172.16.0.250
```

Проверить связь:

```bash
ping -c 4 172.16.0.1
ping -c 4 172.16.0.10
ping -c 4 172.16.0.20
ping -c 4 8.8.8.8
```

---

## 2.6. Создать пользователей `admin` и `monitor`

Это нужно сделать на всех четырёх машинах:

```text
isp
dc
srv
cli
```

### Вводить на каждой машине

```bash
useradd -m -s /bin/bash admin
echo 'admin:P@ssw0rd' | chpasswd

useradd -m -s /bin/bash monitor
echo 'monitor:P@ssw0rd' | chpasswd
```

Проверить:

```bash
id admin
id monitor
```

---

## 2.7. Установить sudo, htop и SSH-сервер

### Вводить на каждой машине: `isp`, `dc`, `srv`, `cli`

```bash
apt-get update
apt-get install -y sudo htop openssh-server
```

Включить SSH-сервер:

```bash
systemctl enable sshd
systemctl start sshd
```

Проверить:

```bash
systemctl status sshd
```

---

## 2.8. Настроить sudo

### Вводить на каждой машине: `isp`, `dc`, `srv`, `cli`

Создать файл sudoers:

```bash
mcedit /etc/sudoers.d/lab-users
```

Вставить:

```sudoers
admin ALL=(ALL) NOPASSWD: ALL

Cmnd_Alias MONITOR_CMDS = /usr/bin/htop, /bin/df, /usr/bin/df, /bin/free, /usr/bin/free, /bin/journalctl, /usr/bin/journalctl, /bin/systemctl status *, /usr/bin/systemctl status *

monitor ALL=(ALL) NOPASSWD: MONITOR_CMDS
```

Выставить правильные права:

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
sudo systemctl status sshd
exit
```

Проверка, что лишние команды запрещены:

```bash
su - monitor
sudo apt-get update
exit
```

Эта команда для `monitor` должна быть запрещена.

---

## 2.9. Захарднить SSH

Нужно сделать на всех четырёх машинах:

```text
isp
dc
srv
cli
```

---

## На каждой машине

Создать баннер:

```bash
echo 'Authorized access only' > /etc/issue.net
```

Открыть основной конфиг SSH:

```bash
mcedit /etc/openssh/sshd_config
```

Если такого файла нет, проверь:

```bash
find /etc -name 'sshd_config'
```

В конец файла добавить:

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
systemctl restart sshd
```

Проверить, что SSH слушает порт `2222`:

```bash
ss -tulpn | grep 2222
```

Должна быть строка с `sshd` и портом `2222`.

---

## 2.10. Проверить SSH с `cli`

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

Проверка, что `root` не может войти по SSH:

```bash
ssh -p 2222 root@172.16.0.1
```

Доступ должен быть запрещён.

---

# Итоговые проверки для отчёта

## На `isp`

```bash
hostname -f
ip -br a
ip route
sysctl net.ipv4.ip_forward
iptables -t nat -L -n -v
iptables -L FORWARD -n -v
systemctl status dhcpd
systemctl status sshd
ss -tulpn | grep 2222
```

---

## На `dc`

```bash
hostname -f
ip -br a
ip route
ping -c 4 172.16.0.1
ping -c 4 8.8.8.8
systemctl status sshd
ss -tulpn | grep 2222
```

---

## На `srv`

```bash
hostname -f
ip -br a
ip route
ping -c 4 172.16.0.1
ping -c 4 8.8.8.8
systemctl status sshd
ss -tulpn | grep 2222
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

# Что заскринить для отчёта

1. На `isp`: `ip -br a`, где видно LAN-адрес `172.16.0.1`.
2. На `isp`: `sysctl net.ipv4.ip_forward`.
3. На `isp`: `iptables -t nat -L -n -v`.
4. На `isp`: `systemctl status dhcpd`.
5. На `dc`: `ip -br a`, где видно `172.16.0.10`.
6. На `srv`: `ip -br a`, где видно `172.16.0.20`.
7. На `cli`: адрес из диапазона `172.16.0.200-172.16.0.250`.
8. На `cli`: успешный `ping` до `isp`, `dc`, `srv` и `8.8.8.8`.
9. На `cli`: успешный вход по SSH на порт `2222`.
10. Проверку, что вход по SSH под `root` запрещён.
11. Проверку `sudo` для `admin`.
12. Проверку разрешённых и запрещённых команд для `monitor`.

---

# Важные замечания под ALT Linux

1. В ALT Linux обычно используется `apt-get`, а не `apt`.
2. Настройка сети обычно находится в `/etc/net/ifaces/`, а не в Netplan.
3. SSH-служба обычно называется `sshd`.
4. Конфиг SSH часто расположен здесь:

```text
/etc/openssh/sshd_config
```

5. DHCP-служба обычно называется `dhcpd`.
6. Если команда или служба не находится, проверяй название так:

```bash
systemctl list-unit-files | grep -i имя
apt-cache search имя
find /etc -name 'имя_файла'
```
