---
title: Runbook — поднять рабочую станцию с нуля
status: active
last_updated: 2026-08-29
references:
  - ./work-from-laptop.md
  - ../../35-security/access-keys.md
  - ../infrastructure.md
referenced_by: []
---

# Поднять рабочую станцию с нуля

> Читается, когда **рабочий VPS недоступен или потерян**. Сессия открывается с ноутбука
> (см. [work-from-laptop.md](work-from-laptop.md)), и по этому файлу станция
> воссоздаётся примерно за 25 минут.

## Что такое рабочая станция

Отдельный VPS, на котором живёт Claude Code и рабочая копия репозитория. **Это не боевой
сервер.** Боевой (`trade-server`, 108.61.167.168) держит магазин, Еву и БД — он не
затрагивается ни при потере станции, ни при её пересоздании.

| | Рабочая станция | Боевой |
|---|---|---|
| Роль | среда разработки, Claude Code | магазин, Ева, Postgres, nginx |
| Потеря означает | неудобство, восстанавливается по этому файлу | остановку бизнеса |
| Данные | копия репозитория (эталон в GitHub) | боевая БД |

## Шаг 1. Создать VPS

Vultr → Compute → Deploy → **Cloud Compute, Shared CPU** · **Amsterdam** (там же боевой) ·
**Ubuntu 24.04 LTS** · **2 vCPU / 4 GB** (Claude Code требует от 4 ГБ; на приёмке
поднимается Chrome) · Auto Backups включить · hostname `trade-workstation`.

Записать IP и root-пароль.

## Шаг 2. Ключ вместо пароля

С ноутбука:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/trade_workstation -N '' -C "mac->trade-workstation"
export SSHPASS='<root-пароль>'
sshpass -e ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no root@<IP> \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && echo '$(cat ~/.ssh/trade_workstation.pub)' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
ssh -i ~/.ssh/trade_workstation root@<IP> 'echo ok'
```

## Шаг 3. Защита

```bash
ufw default deny incoming; ufw default allow outgoing; ufw allow 22/tcp; ufw --force enable
printf 'PasswordAuthentication no\nPermitRootLogin prohibit-password\nKbdInteractiveAuthentication no\n' \
  > /etc/ssh/sshd_config.d/99-hardening.conf
sshd -t && systemctl reload ssh
apt-get update -qq && apt-get -y upgrade
apt-get -y install fail2ban unattended-upgrades
systemctl enable --now fail2ban
dpkg-reconfigure -f noninteractive unattended-upgrades
timedatectl set-timezone Europe/Moscow      # иначе записи в changelog датируются вчера
```

## Шаг 4. Окружение

```bash
apt-get -y install git tmux curl ripgrep sshpass jq python3-pip python3-venv unzip
curl -fsSL https://deb.nodesource.com/setup_22.x | bash - && apt-get -y install nodejs
pip3 install --break-system-packages openpyxl pillow

# Chrome для headless-приёмки. Именно Google Chrome, НЕ snap-chromium:
# скрипты приёмки запускаются с channel:"chrome", и snap-версия им не подходит.
install -d -m 0755 /etc/apt/keyrings
curl -fsSL https://dl.google.com/linux/linux_signing_key.pub | gpg --dearmor -o /etc/apt/keyrings/google-chrome.gpg
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/google-chrome.gpg] https://dl.google.com/linux/chrome/deb/ stable main" \
  > /etc/apt/sources.list.d/google-chrome.list
apt-get update -qq && apt-get -y install google-chrome-stable

curl -fsSL https://claude.ai/install.sh | bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
```

⚠️ Под root у Chrome нет песочницы — в харнессах приёмки запуск идёт с
`args:["--no-sandbox","--disable-dev-shm-usage"]`.

## Шаг 5. Ключ для GitHub

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" -C "trade-workstation"
cat ~/.ssh/id_ed25519.pub
```

Публичный ключ добавить: **github.com → Settings → SSH and GPG keys → New SSH key**.
Старый ключ потерянной станции оттуда **удалить** — см. [access-keys.md](../../35-security/access-keys.md).

## Шаг 6. Репозиторий, память, секреты

```bash
git clone git@github.com:maximsk7979-afk/trade-abkhazia.git /root/trade_app_repo
cd /root/trade_app_repo
git config --global --add safe.directory /root/trade_app_repo
git config user.name "Maxim Skorokhodov"; git config user.email "maximsk7979@gmail.com"

./scripts/sync-memory.sh restore     # память агента из agent-memory/ → ~/.claude/...
```

**Секреты** (`~/secrets-trade/credentials.md`) в git НЕ хранятся. Взять из менеджера
паролей или скопировать с ноутбука:

```bash
scp -i ~/.ssh/trade_workstation ~/secrets-trade/credentials.md root@<IP>:~/secrets-trade/
ssh -i ~/.ssh/trade_workstation root@<IP> 'chmod 700 ~/secrets-trade; chmod 600 ~/secrets-trade/*'
```

**Ключ станции на боевой** (чтобы деплой шёл без пароля):

```bash
export SSHPASS='<пароль боевого из credentials.md>'
sshpass -e ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no root@108.61.167.168 \
  "echo '<содержимое ~/.ssh/id_ed25519.pub со станции>' >> ~/.ssh/authorized_keys"
```

## Шаг 7. Хуки и права Claude Code

```bash
mkdir -p /root/trade_app_repo/.claude
sed 's|/Users/skorokhodovmaxim/Documents/trade_app|/root/trade_app_repo|g' \
  /root/trade_app_repo/hooks/settings-hooks.json.sample > /root/trade_app_repo/.claude/settings.json
chmod +x /root/trade_app_repo/hooks/*.sh
mkdir -p ~/.claude && printf '{"permissions":{"defaultMode":"bypassPermissions"}}\n' > ~/.claude/settings.json
```

⚠️ Без хуков всё работает, но **молча пропадает протокол сессии** — CC перестаёт читать
roadmap, gaps и обязательства. Проверить: в начале сессии должно появиться напоминание.

## Шаг 8. Claude Code под systemd

```bash
claude    # первый запуск: выбрать тему, войти в аккаунт по ссылке, вставить код
```

Затем автозапуск (файл юнита — в шаге ниже), проверка `systemctl status claude-rc`.

## Шаг 9. Приёмка переезда

- [ ] `cd /root/trade_app_repo && git log --oneline -1` — последний коммит на месте
- [ ] `git rev-parse HEAD^{tree}` совпадает с GitHub
- [ ] `ls code/retail-app/img/*.jpg | wc -l` = 46
- [ ] `./scripts/deploy.sh retail-app` проходит
- [ ] тестовый коммит уходит в GitHub авто-пушем
- [ ] сессия открывается с телефона
- [ ] в начале сессии появляется напоминание протокола (хуки живы)

## Что НЕ нужно делать

- Не поднимать станцию на боевом сервере: 1 ядро, 2 ГБ, и рядом магазин.
- Не класть секреты в git — см. [GAP-006](../../00-meta/gaps.md#gap-006).
- Не работать одновременно с ноутбука и станции: писатель один (станция).
