# Linux для AppSec-инженера

> Полное руководство: см. остальные файлы в папке SDLC


### 20.1 Базовые команды

```bash
# Навигация
ls -la, cd, pwd, find / -name "*.conf", tree

# Процессы и сеть
ps aux, top/htop, netstat -tulpn, ss -tulpn, lsof -i :8080

# Файлы и права
chmod 750 file, chown user:group, umask

# Текстовые редакторы
vim, nano, sed, awk, grep, cut, sort, uniq

# SSH
ssh -i key.pem user@host, ssh-keygen -t ed25519, scp, rsync
> **SSH** – Secure Shell (защищённая оболочка (протокол удалённого доступа))


# Логи
journalctl -u service, tail -f /var/log/syslog, dmesg

# Пакеты
apt update && apt install, yum/dnf install, dpkg -l
```

### 20.2 Firewall: iptables / nftables / ufw

**iptables** – классический фаервол Linux (Netfilter). Таблицы: filter, nat, mangle. Цепочки: INPUT, OUTPUT, FORWARD.

**Пример (разрешить SSH, HTTP, HTTPS, закрыть остальное):**
```bash
# политика по умолчанию – запрет
iptables -P INPUT DROP
iptables -P FORWARD DROP
# разрешить loopback и установленные соединения
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
# разрешить порты
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
```

**nftables** – современная замена iptables (ядро 3.13+):
```bash
nft add table inet filter
nft add chain inet filter input { type filter hook input priority 0\; policy drop\; }
nft add rule inet filter input tcp dport 22 accept
```

**ufw (Uncomplicated Firewall)** – обёртка над iptables:
```bash
ufw default deny incoming
ufw allow 22/tcp
ufw allow 80,443/tcp
ufw enable
```

**Другие инструменты безопасности Linux:**
- **Fail2ban** – блокировка перебора паролей по логам
- **SELinux / AppArmor** – мандатный контроль доступа
- **auditd** – аудит событий
- **Lynis** – аудит безопасности системы
- **ClamAV** – антивирус
- **Unattended-upgrades** – автоматические обновления безопасности


---

## 📎 Пример: настройка firewall (iptables / nftables / ufw)

### iptables (классика)

```bash
# Сбросить правила
iptables -F
iptables -X

# Политика по умолчанию: ЗАПРЕТИТЬ всё
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Разрешить loopback
iptables -A INPUT -i lo -j ACCEPT

# Разрешить установленные соединения
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Разрешить SSH (только с доверенных IP!)
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT

# Разрешить HTTP/HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Защита от сканирования (пример)
iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP

# Сохранить правила
iptables-save > /etc/iptables/rules.v4
```

### nftables (современный стандарт)

```bash
# /etc/nftables.conf
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;
        iif "lo" accept
        ct state established,related accept
        tcp dport 22 ip saddr 10.0.0.0/8 accept
        tcp dport 80 accept
        tcp dport 443 accept
    }
}
```

### ufw (простая обёртка)

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
ufw status verbose
```

---

## 📎 Пример: fail2ban (защита от брутфорса SSH)

```bash
# Установка
apt install fail2ban

# /etc/fail2ban/jail.local
[sshd]
enabled = true
port = ssh
maxretry = 5          # 5 попыток
bantime = 3600        # бан на 1 час
findtime = 600        # за 10 минут

[nginx-http-auth]
enabled = true
maxretry = 5
bantime = 3600

# Проверка статуса
fail2ban-client status sshd
fail2ban-client set sshd unbanip 1.2.3.4
```

---

## 📎 Пример: базовый аудит Linux (Lynis)

```bash
# Установка и запуск аудита
apt install lynis
lynis audit system

# Ключевые проверки (фрагмент вывода):
# - SSH: PermitRootLogin запрещён?
# - Пароли: используются ли хэши sha512?
# - Обновления: установлены ли security-патчи?
# - Firewall: активен ли?
# - Права на /etc/shadow
```

---

## 📎 Пример: полезные команды для AppSec на Linux

```bash
# Что слушает порты
ss -tulpn
netstat -tulpn

# Открытые файлы процесса
lsof -p <pid>
lsof -i :8080

# Поиск файлов с SUID (потенциальная эскалация)
find / -perm -4000 -type f 2>/dev/null

# История команд (может содержать секреты!)
cat ~/.bash_history

# Проверка целостности пакетов
dpkg -V
rpm -Va

# Журнал
journalctl -u nginx --since today
dmesg | tail

# Сетевые соединения процесса
lsof -i -n -P | grep <pid>
```
