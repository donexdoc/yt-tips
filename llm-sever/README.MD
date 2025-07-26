# 🧠 Домашний сервер для нейросетей с Ollama, GPU и WebUI

Это пошаговое руководство по запуску локального сервера нейросетей на базе Proxmox с использованием Ollama, проброса GPU и Open WebUI.

---

## 🧾 Содержание

1. [Требования к железу](#1-требования-к-железу)
2. [Создание ВМ для Ollama CPU](#2-создание-вм-для-ollama-cpu)
3. [Установка Ubuntu Server](#3-установка-ubuntu-server)
4. [Установка Ollama (CPU)](#4-установка-ollama-cpu)
5. [Проверка работы и API Ollama](#5-проверка-работы-и-api-ollama)
6. [Настройка WebUI на отдельной ВМ](#6-настройка-webui-на-отдельной-вм)
7. [Проброс GPU в Ollama GPU ВМ](#7-проброс-gpu-в-ollama-gpu-вм)
8. [Установка драйверов NVIDIA](#8-установка-драйверов-nvidia)
9. [Установка Ollama (GPU)](#9-установка-ollama-gpu)
10. [Исправление проблемы запуска с CPU после перезагрузки](#10-исправление-проблемы-gpu-после-перезагрузки)
11. [Дополнительно: Web-прокси через NGINX](#11-web-прокси-для-api-ollama)

---

## 1. Требования к железу

- CPU: от 4 ядер (лучше 6–8)
- RAM: от 12–16 ГБ
- GPU: NVIDIA (например, RTX 3060 12 ГБ)
- Proxmox установлен с поддержкой UEFI

---

## 2. Создание ВМ для Ollama CPU

- **BIOS:** OVMF (UEFI)
- **Machine:** q35
- **CPU:** 6 ядер, Type: `host`
- **RAM:** 12 ГБ
- **Disk:** 40–50 ГБ SSD, Cache: `Write back`
- **Ballooning:** отключить
- **QEMU Agent:** включить
- **OS:** Ubuntu 22.04 LTS

---

## 3. Установка Ubuntu Server

- Выбираем минимальную установку
- Расширяем раздел `ubuntu-lv` вручную
- Устанавливаем SSH и `qemu-guest-agent`
- После установки ставим:
  ```bash
  sudo apt install nano htop qemu-guest-agent -y
  ```

## 4. Установка Ollama (CPU)

```bash

curl -fsSL https://ollama.com/install.sh | sh

#  проверяем установленную версию
ollama --version

```

## 5. Проверка работы и API Ollama

```bash

ollama run gemma3:4b

#  проверяем работу API
curl http://localhost:11434/api/tags

```

## 6. Настройка WebUI на отдельной ВМ

Параметры ВМ:

- 2 ядра / 2 ГБ / Ubuntu 22.040
- Установка Docker [Актуальная и официальная инструкция](https://docs.docker.com/engine/install/ubuntu/)

мой `docker-compose.yml` но я рекомендую ставить по инструкциям из [официального репозитория](https://github.com/open-webui/open-webui)

```yml
version: "3.8"
services:
  webui:
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - 3000:8080
    volumes:
      - openwebui-data:/app/backend/data
    environment:
      - OLLAMA_API_BASE_URL=http://ollama-cpu
volumes:
  openwebui-data:
```

Запускаем через compose

```bash
docker compose up -d
```

## 7. Проброс GPU в Ollama GPU ВМ

### 1. В вашем BIOS нужно включить виртуализацию и 2 параметра виртуализации, если они есть. Это `VT-d` и `VT-x`.

### 2. Настроить proxmox

модифицировать grub:

```bash
sudo nano /etc/default/grub
```

найти первую строчку и заменить на вторую

```ini
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
```

Перезагрузить grub

```bash
sudo update-grub
```

---

включить модули VFIO

```bash
sudo nano /etc/modules-load.d/vfio.conf
```

Вписать:

```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

---

Добавить в игнор карту для Proxmox:

```bash
sudo nano /etc/modprobe.d/blacklist-nouveau.conf
```

содержимое:

```sh
blacklist nouveau
options nouveau modeset=0
```

---

Определеить id карты

```bash

lspci -nn | grep -i nvidia
```

у меня вывод был вот такой:

```text
04:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA106 [GeForce RTX 3060 Lite Hash Rate] [10de:2504] (rev a1)
04:00.1 Audio device [0403]: NVIDIA Corporation GA106 High Definition Audio Controller [10de:228e] (rev a1)

```

запоминаем `[10de:2504]` и `[10de:228e]` это id PCI-E для проброса устройства

создаём VFIO binding:

```bash
sudo nano /etc/modprobe.d/vfio-pci.conf
```

вставляем

```text
options vfio-pci ids=10de:2504,10de:228e
```

Перегенерируем initramfs и перегружаем сервак

```bash
sudo update-initramfs -u -k all

sudo reboot
```

Проверяем карту

```bash
lspci -nnk | grep -A 2 -i nvidia
```

вывод должен содержать

```text
Kernel driver in use: vfio-pci
```

### 3. Пробросить карту

В интерфейсе Proxmox:

1. Останавливаем ВМ с ollama gpu

2. Переходим в Hardware → Add → PCI Device

ищем:

NVIDIA RTX 3060 (ну или какая у вас)

Ставим галки :

- ✅ All Functions

- ✅ PCI-Express

- ✅ Primary GPU

запускаем VM

## 8. Установка драйверов NVIDIA

```sh
sudo apt update
sudo ubuntu-drivers autoinstall
sudo reboot

# проверяем, есть ли вывод инфы о карте
nvidia-smi
```

## 9. Установка Ollama (GPU)

почти так же, но желательно проверить libcudа

```bash
ldconfig -p | grep libcuda
```

Если библиотеки на месте, то ставим ollama через ту же команду. Инсталлятор сам определить GPU если она есть и настроена корректно.

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

смотреть статистику по GPU в лайве можно так

```bash
watch -n 1 nvidia-smi
```

## 10. Исправление проблемы GPU после перезагрузки

После перезагрузки машины с GPU ollama, я словил баг. Видеокарта работает, отображается в системе, но вычисления продолжают происходить на cpu и занимается оперативка, а не видеопамять.
Это происходит потому что GPU не успевает загрузиться и стартует после сервиса ollama. Я не знаю, как правильно это фиксить, но для себя сделал костыль. Поставил задержку в конфиге сервиса ollama.

```bash
sudo systemctl edit ollama
```

над строкой `### Edits below this comment will be discarded` вставляем задержку. получится что-то типа такого:

```ini
### Editing /etc/systemd/system/ollama.service.d/override.conf
### Anything between here and the comment below will become the contents of the drop-in file

[Service]
ExecStartPre=/bin/sleep 5

### Edits below this comment will be discarded
```

После этого презагружаем демон и конфиг

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload

sudo systemctl restart ollama

sudo systemctl status ollama
```

## 11. Web-прокси для API Ollama

Ставим nginx и создаем конфиг

```bash

sudo apt install nginx -y

sudo nano /etc/nginx/sites-available/ollama


```

в конфиге

```cfg
server {
    listen 80;
    server_name <ТВОЙ_IP_ИЛИ_ДОМЕН>;

    location / {
        proxy_pass http://localhost:11434;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

запускаем наш конфиг

```bash
sudo ln -s /etc/nginx/sites-available/ollama /etc/nginx/sites-enabled/
sudo nginx -t  # проверить синтаксис
sudo systemctl reload nginx
```

на другом устройстве проверяем работу nginx

```bash
curl http://<ip_или_домен>/api/tags
```
