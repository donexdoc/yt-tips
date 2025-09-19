# Параметры для уменьшения шума HPE ProLiant DL360 Gen10

Все параметры, которые я выставлял в своем ролике

---

## 🧾 Содержание

1. [iLO Настройки](#1-iLO-Настройки)
2. [BIOS](#2-BIOS)
3. [Proxmox](#3-Proxmox)
4. [Рекомендации по параметрам ВМ](#4-Рекомендации-по-параметрам-ВМ)

---

## 1. iLO Настройки

- **Power Profile:** `Static Low Power`
- **Thermal Mode:** `Optimal Cooling`
- **Collaborative Power Control:** `Enabled`

---

## 2. BIOS

- **Power Profile:** `Static Low Power`
- **Energy Efficient Turbo:** `Enabled`
- **Minimum Processor Idle Power Core C-State:** `C6 Retention`
- **Minimum Processor Idle Power Package C-Stat:** `Package C6`
- **Intel Turbo Boost Technology:** `Disabled`
- **Intel Hyperthreading:** `Enabled`
- **Memory Operating Speed:** `2133MHz`
- **Fan and Thermal Options:** `Optimal Cooling`

---

## 3. Proxmox

### Управление CPU

```bash

# Проверить текущий governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Установить powersave режим
echo 'powersave' | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Сделать постоянным - добавить в /etc/rc.local
echo 'echo powersave | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor' >> /etc/rc.local

```

### Установка cpufrequtils

```bash

apt update && apt install cpufrequtils
echo 'GOVERNOR="powersave"' > /etc/default/cpufrequtils
systemctl enable cpufrequtils

```

### Настройка C-states

В `/etc/default/grub` добавить к `GRUB_CMDLINE_LINUX_DEFAULT` чтобы выглядело так:

```bash

GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_idle.max_cstate=6 processor.max_cstate=6 intel_pstate=passive"

```

Применяем настройки grub и перезагружаем систему:

```bash
update-grub && reboot
```

Устанавливаем инструменты мониторинга

```bash
apt install lm-sensors fancontrol
sensors-detect
sensors
```

## 4. Рекомендации по параметрам ВМ

- **CPU Type:** `host`
- **Enable NUMA:** `Yes`

Используйте динамическое выделение ОЗУ (`ballooning device`) если это допустимо для ВМ.
