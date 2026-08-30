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
sudo adduser --disabled-password --gecos '' max && sudo usermod -aG sudo max
echo 'max ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/90-max
# дальше — от имени max:
git clone git@github.com:maximsk7979-afk/trade-abkhazia.git ~/trade_app_repo
cd ~/trade_app_repo
git config --global --add safe.directory /home/max/trade_app_repo
git config user.name "Maxim Skorokhodov"; git config user.email "maximsk7979@gmail.com"

./scripts/sync-memory.sh restore     # память агента из agent-memory/ → ~/.claude/...
ln -sf ../../hooks/pre-commit-writer-guard.sh .git/hooks/pre-commit   # защита от расхождения веток

# проверить, что память легла ТУДА, где Claude Code ведёт транскрипты (*.jsonl):
ls -d ~/.claude/projects/*trade*/          # ожидается ОДИН каталог
```

⚠️ **Каталог памяти именуется слугификацией рабочего пути**: `/` и `_` превращаются в `-`,
то есть `/home/max/trade_app_repo` → `-home-max-trade-app-repo`. Каталог с подчёркиванием
в имени — признак ошибки переноса. Два каталога опасны: `sync-memory.sh save` может
затолкать в репозиторий устаревшую память и молча откатить правки. Скрипт теперь вычисляет
путь сам и при неоднозначности останавливается, но лишний каталог всё равно убрать.

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
REPO=/home/max/trade_app_repo
mkdir -p $REPO/.claude
# В образце стоит плейсхолдер __REPO__ — подставляем путь этой машины.
sed "s|__REPO__|$REPO|g" $REPO/hooks/settings-hooks.json.sample > $REPO/.claude/settings.json

# проверить, что ОБА пути ведут к существующим файлам (иначе хук молча не запустится):
python3 -c "import json,os;h=json.load(open('$REPO/.claude/settings.json'))['hooks'];\
print({k:os.path.exists(v[0]['hooks'][0]['command'].split()[1]) for k,v in h.items()})"
# ожидается {'SessionStart': True, 'UserPromptSubmit': True}
chmod +x $REPO/hooks/*.sh
mkdir -p ~/.claude && printf '{"permissions":{"defaultMode":"bypassPermissions"}}\n' > ~/.claude/settings.json
```

⚠️ Без хуков всё работает, но **молча пропадает протокол сессии** — CC перестаёт читать
roadmap, gaps и обязательства. Проверить: в начале сессии должно появиться напоминание.

## Шаг 8. Claude Code под systemd

```bash
claude    # первый запуск: выбрать тему, войти в аккаунт по ссылке, вставить код
```

Вход интерактивный: Claude Code печатает ссылку, её открывают на телефоне, код
возвращают в терминал. На headless-сервере удобно вести через tmux:

```bash
tmux new-session -d -s claude -c /home/max/trade_app_repo -x 200 -y 50
tmux send-keys -t claude "claude" Enter
tmux capture-pane -pt claude | tail -30     # прочитать ссылку
tmux send-keys -t claude "<код>" Enter
```

Затем автозапуск (файл юнита — в шаге ниже), проверка `systemctl status claude-rc`.

## Шаг 9. Приёмка переезда

- [ ] `cd /home/max/trade_app_repo && git log --oneline -1` — последний коммит на месте
- [ ] `git rev-parse HEAD^{tree}` совпадает с GitHub
- [ ] `ls code/retail-app/img/*.jpg | wc -l` = 46
- [ ] `./scripts/deploy.sh retail-app` проходит
- [ ] тестовый коммит уходит в GitHub авто-пушем
- [ ] сессия открывается с телефона
- [ ] в начале сессии появляется напоминание протокола (хуки живы)

## Грабли, на которые уже наступали

- **`fail2ban` банит за серию неудачных входов** — в том числе автоматизацию, если
  сломалась подстановка ключа и ssh свалился на пароль. Бан 10 минут, порт отвечает
  «Connection refused». Адрес ноутбука вынесен в `/etc/fail2ban/jail.d/ignore-owner.conf`;
  при смене адреса файл нужно обновить. Всегда ходить с `-o BatchMode=yes`.
- **Путь к хукам** — см. предупреждение в шаге 7.
- **Старый rsync в macOS** (2.6.9) не понимает `--info`; перенос репозитория надёжнее
  делать потоком: `tar czf - . | ssh ... 'tar xzf - -C /home/max/trade_app_repo'`.
- **Права после переноса с macOS**: файлы приезжают с чужим uid, git ругается
  «dubious ownership». Лечится `chown -R root:root` + `git config --global --add safe.directory`.
- **macOS-мусор** `._*` в архиве — удалить: `find . -name '._*' -not -path './.git/*' -delete`.

## Эксплуатация: сессии Claude Code на станции

Главное отличие от работы в локальном терминале: **закрыть приложение ≠ завершить сессию**.
Раньше процесс жил в терминале ноутбука и умирал вместе с ним. Здесь он живёт на станции:

| Процесс | Что это |
|---|---|
| `claude remote-control --spawn=same-dir` | слушатель, поднимается службой, работает всегда |
| `… --sdk-url … --resume=…` | **сама сессия** (один процесс на разговор, ~370 МБ) |

Закрытие вкладки или приложения на телефоне отключает только клиента — процесс сессии
продолжает жить со всем контекстом, поэтому при возврате открывается тот же разговор.

**Начать с чистого листа:** `/clear` в сессии, затем первой фразой «продолжаем работу».
Это аналог нового терминала: контекст стёрт, процесс тот же, ничего не накапливается.
Фраза нужна потому, что перезапуск `SessionStart`-хука на `/clear` не гарантирован, а
она заставляет CC перечитать протокол в любом случае.

**Если реально подвисло:** `sudo systemctl restart claude-rc` — убивает все сессии, служба
поднимается заново. Репозиторий и git не затрагиваются.

**Реальный риск — не зависание, а накопление.** Каждая новая сессия из приложения поднимает
отдельный процесс, а старый остаётся жить. При ~370 МБ на сессию и ~3 ГБ свободных станция
вывозит **примерно 8** сессий; в юните при этом стоит `--capacity 32` (умолчание), вчетверо
больше физической вместимости. Посмотреть живые сессии:

```bash
ps -eo pid,etime,rss,args --no-headers | grep -F -- '--sdk-url' | grep -vF bash \
  | awk '{printf "сессия PID %s  живёт %s  память %d МБ\n",$1,$2,$3/1024}'
```

Накопилось 5–6 — перезапустить службу или снять лишние по PID. **Открытый вопрос:** снизить
`--capacity` до 4–6 и добавить `MemoryMax=` в юнит, чтобы лишняя сессия не поднималась
вместо того, чтобы ронять машину по памяти (решение владельца, требует перезапуска службы).

## Что НЕ нужно делать

- Не поднимать станцию на боевом сервере: 1 ядро, 2 ГБ, и рядом магазин.
- Не класть секреты в git — см. [GAP-006](../../00-meta/gaps.md#gap-006).
- Не работать одновременно с ноутбука и станции: писатель один (станция).
  Защиту ставит хук `pre-commit` — см. [work-from-laptop.md](work-from-laptop.md).
- **Работать под root нельзя**: Claude Code отказывается стартовать в режиме без
  подтверждений от имени root. Рабочее место живёт под пользователем `max`
  (`/home/max/trade_app_repo`), у него sudo без пароля.
