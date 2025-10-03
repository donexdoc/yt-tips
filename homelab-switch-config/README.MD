# Настройка коммутатора cisco nexus

## 1️⃣ Первичная настройка: SSH и Management

Все конфиги вводятся через console порт

### Поиск USB-Serial устройства (macOS/Linux)

```bash
ls /dev/tty.*

# выведет что-то типа
/dev/tty.usbserial-1337
```

подключение

```bash
screen /dev/tty.usbserial-1337 9600
```

---

### Переход в конфигурационный режим

```bash
configure terminal
```

### Устанавливаем имя хоста

```bash
hostname sw-box
```

### Настраиваем mgmt0 порт

```bash
interface mgmt0
ip address 192.168.1.2/24
no shutdown
exit
```

### Настраиваем шлюз для доступа из других подсетей

```bash
ip default-gateway 192.168.1.1
```

### Создаем локального пользователя

```bash
username admin password <strong_password> role network-admin
```

### Включаем SSH и задаем версии

```bash
feature ssh
ssh key rsa 2048
ssh server v2
ssh server algorithm mac hmac-sha1
ssh server algorithm publickey rsa
```

### Сохраняем изменения

```bash
copy running-config startup-config
```

## 2️⃣ Подключение iLO в отдельную подсеть (ZOO, VLAN100)

Эта конфигурация задавалась через порт management 0

### Подключаемся по ssh

```bash
ssh admin@192.168.1.2
```

Если возникают ошибки по ключам и алгоритмам на новых macOS/Linux:

```bash
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa admin@192.168.1.2
```

Если требуется добавить поддержку старых MAC-алгоритмов:

```bash
ssh -o HostKeyAlgorithms=+ssh-rsa \
    -o PubkeyAcceptedKeyTypes=+ssh-rsa \
    -o MACs=hmac-sha1 \
    admin@192.168.1.2
```

---

### Входим в режим конфигурации

```bash
configure terminal
```

### Создаем VLAN100 для серверного сегмента (ZOO)

```bash
vlan 100
  name ZOO
exit
```

### Включаем SVI для VLAN100

```bash
interface vlan 100
  ip address 192.168.100.2/24
  no shutdown
exit
```

### Настраиваем порт для iLO (например Eth1/13)

```bash
interface Ethernet1/13
  switchport
  switchport mode access
  switchport access vlan 100
  no shutdown
exit
```

### Проверяем VLAN и порты

```bash
show vlan brief
show interface status
```

### Сохраняем изменения

```bash
copy running-config startup-config
```
