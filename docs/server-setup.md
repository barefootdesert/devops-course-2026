# Конфигурация виртуальной машины devops-vm

Порядок восстановления после отката на снимок `01-clean-install`: разделы 3 → 4 → 5 → 6 → 8.
Каждое изменение SSH и межсетевого экрана проверяется новым подключением во **втором** терминале,
действующая сессия до проверки не закрывается.

## 1. Параметры машины

| Параметр | Значение |
|---|---|
| Гипервизор | VirtualBox 7.2 (Hyper-V на хосте отключён: `bcdedit /set hypervisorlaunchtype off`) |
| Имя | `devops-vm` |
| ОС | Ubuntu Server 26.04.1 LTS (64-bit), без графической подсистемы |
| Оперативная память | 2048 МБ |
| Процессор | 2 ядра |
| Диск | 25 ГБ, VDI, динамически расширяемый |

## 2. Сетевые интерфейсы

| Адаптер VirtualBox | Тип | Интерфейс в ОС | Адрес | Назначение |
|---|---|---|---|---|
| Адаптер 1 | NAT | `enp0s3` | `10.0.2.15/24` (DHCP) | выход в Интернет, apt; вход только через проброс портов |
| Адаптер 2 | Host-only (`VirtualBox Host-Only Ethernet Adapter`) | `enp0s8` | `192.168.56.101/24` (DHCP) | прямой доступ с хоста (браузер, `devops.local`) |

Адрес хоста в сети Host-only — `192.168.56.1/24`. Если после перезагрузки Windows он сменился на `169.254.x.x`:

```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" hostonlyif ipconfig "VirtualBox Host-Only Ethernet Adapter" --ip 192.168.56.1 --netmask 255.255.255.0
```

VPN-клиенты на хосте (AmneziaVPN, Outline) перехватывают маршрут `192.168.56.0/24` и должны быть отключены.

## 3. Правило проброса портов

| Имя | Протокол | Адрес хоста | Порт хоста | Адрес гостя | Порт гостя |
|---|---|---|---|---|---|
| `ssh` | TCP | `127.0.0.1` | `2222` | — | `2222` |

После отката на `01-clean-install` порт гостя равен `22`. Его меняют на `2222` **после** перезапуска sshd на новом порту (раздел 5):

```powershell
$vb = "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe"
& $vb controlvm devops-vm natpf1 delete ssh
& $vb controlvm devops-vm natpf1 "ssh,tcp,127.0.0.1,2222,,2222"
```

## 4. Учётные записи

| Имя | Группы | Аутентификация по SSH |
|---|---|---|
| `vboxuser` | `vboxuser`, `adm`, `sudo`, `users`, `lxd` и др. | запрещена (`AllowUsers devops`); создан автоматической установкой VirtualBox |
| `student` | `student`, `sudo`, `users` | запрещена (`AllowUsers devops`); только локальная консоль |
| `devops` | `devops`, `sudo`, `users` | только по ключу ed25519 |
| `root` | — | запрещена (`PermitRootLogin no`) |

На сервере (вход `ssh student@127.0.0.1 -p 2222` по паролю, пока парольный вход ещё разрешён):

```bash
sudo adduser devops
sudo usermod -aG sudo devops
groups devops
```

На хосте (PowerShell). Отдельный ключ, не ключ для GitHub:

```powershell
ssh-keygen -t ed25519 -C "devops-vm-key" -f $env:USERPROFILE\.ssh\devops_vm
type $env:USERPROFILE\.ssh\devops_vm.pub | ssh devops@127.0.0.1 -p 2222 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Права на сервере: `~/.ssh` — `700`, `~/.ssh/authorized_keys` — `600`.

Файл `%USERPROFILE%\.ssh\config` на хосте:

```
Host devops
    HostName 127.0.0.1
    Port 2222
    User devops
    IdentityFile ~/.ssh/devops_vm
    IdentitiesOnly yes

Host devops.local
    Port 2222
    User devops
    IdentityFile ~/.ssh/devops_vm
    IdentitiesOnly yes
```

`Port 2222` здесь — порт хоста, поэтому файл не меняется при смене порта гостя в разделе 5.

## 5. Служба SSH

Основной файл `/etc/ssh/sshd_config` не изменяется, эталонная копия — `/etc/ssh/sshd_config.backup`.
Изменения вынесены в `/etc/ssh/sshd_config.d/99-hardening.conf`:

```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
LoginGraceTime 30
AllowUsers devops
X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
```

Команды:

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
sudo nano /etc/ssh/sshd_config.d/99-hardening.conf      # содержимое выше
grep -r "PasswordAuthentication" /etc/ssh/sshd_config.d/ /etc/ssh/sshd_config   # конфликтующих файлов (50-cloud-init.conf) нет
sudo sshd -t && echo "Конфигурация корректна"
sudo sshd -T | grep -Ei "^(port|permitrootlogin|passwordauthentication|maxauthtries|allowusers)"
sudo systemctl disable --now ssh.socket                 # иначе сокет держит порт 22
sudo systemctl enable --now ssh.service
sudo systemctl restart ssh
sudo ss -tlnp | grep sshd                               # ожидается 0.0.0.0:2222
```

Затем — изменение проброса портов (раздел 3) и проверки во втором терминале:

```powershell
ssh devops                                                   # успех
ssh -p 2222 root@127.0.0.1                                   # Permission denied (publickey)
ssh -p 2222 -o PubkeyAuthentication=no devops@127.0.0.1      # Permission denied (publickey)
```

Ожидаемые значения `sshd -T`: `port 2222`, `permitrootlogin no`, `passwordauthentication no`, `maxauthtries 3`, `allowusers devops`.
Принимаемые методы аутентификации: до настройки `publickey,password`, после — `publickey`.

## 6. Правила межсетевого экрана

Политики по умолчанию: `deny (incoming)`, `allow (outgoing)`, `disabled (routed)`. Журналирование: `medium`.

| Порт | Действие | Комментарий |
|---|---|---|
| `2222/tcp` | `LIMIT IN` | `SSH rate-limited` (блокировка источника при 6+ подключениях за 30 с) |
| `80/tcp` | `ALLOW IN` | `HTTP` (работа № 6) |
| `443/tcp` | `ALLOW IN` | `HTTPS` |

Правила действуют для IPv4 и IPv6. Разрешающее правило для 2222 создаётся **до** `ufw enable`.
Команду `ufw allow ssh` не использовать: она открывает порт 22 по `/etc/services`.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp comment 'SSH'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
sudo ufw enable
sudo ufw delete allow 2222/tcp
sudo ufw limit 2222/tcp comment 'SSH rate-limited'
sudo ufw logging medium
sudo ufw status verbose
```

Журнал блокировок: `/var/log/ufw.log` (`sudo grep 'UFW BLOCK' /var/log/ufw.log | tail -5`).

## 7. Снимки состояния

| Имя | Момент создания | Состояние |
|---|---|---|
| `01-clean-install` | после установки ОС и `apt upgrade`, машина выключена | чистая система, пользователи `vboxuser`, `student`, проброс `2222 → 22` |
| `02-keys-configured` | после задания 1, машина выключена | пользователь `devops`, вход по ключу, `~/.ssh/config` на хосте |
| `03-ssh-hardened` | после задания 2, машина выключена | `99-hardening.conf`, sshd на 2222 как `ssh.service`, проброс `2222 → 2222` |

Снимки делаются на выключенной машине: живой снимок при включённом Hyper-V зависал.

## 8. Доменное имя

На хосте, в `C:\Windows\System32\drivers\etc\hosts` (редактор от имени администратора):

```
192.168.56.101 devops.local
```

```powershell
ipconfig /flushdns
ping devops.local
ssh devops.local                # через Host-only напрямую, ключ из ~/.ssh/config
```

На сервере:

```bash
sudo hostnamectl set-hostname devops-vm
sudo sed -i 's/^127.0.1.1.*/127.0.1.1 devops-vm.devops.local devops-vm/' /etc/hosts
hostname -f                     # devops-vm.devops.local
```

## 9. Проверка

```bash
sudo ./scripts/audit.sh         # все проверки [OK], код возврата 0
```
