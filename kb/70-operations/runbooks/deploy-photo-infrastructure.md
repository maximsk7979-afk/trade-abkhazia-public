---
title: "Runbook: деплой фото-инфраструктуры (Блок 1 GAP-018)"
status: active
last_updated: 2026-05-30
references:
  - ../../40-system/photo-infrastructure.md
  - ../../90-decisions/ADR-016-photo-infrastructure.md
---

# Runbook: деплой фото-инфраструктуры (Блок 1 GAP-018)

Минимальный деплой первой базы фото — `client-locations` (фото витрин клиентов). Закрывает [GAP-018](../../00-meta/gaps.md#gap-018). После прохождения этого runbook поле 9 онбординга («фото места доставки») становится рабочим.

## Что задеплоить

| Компонент | Где | Источник |
|---|---|---|
| `server.js` расширен `/api/photos` | `/var/www/trade/server.js` | `code/trade-api/server.js` (после слияния с текущим) |
| `photo-bases.js` + `photos-router.js` | `/var/www/trade/` | `code/trade-api/photo-bases.js`, `code/trade-api/photos-router.js` |
| Зависимости trade-api | `npm install multer image-size` | в `/var/www/trade/` |
| Папка хранения фото | `/var/www/trade/photos/` | `mkdir` + права |
| Nginx `location /photos/` | `/etc/nginx/sites-enabled/trade-abkhazia.conf` | runbook ниже |
| Фото-модуль Евы | `/var/www/eva/src/photo/` + `index.js`, `session-state.js`, `system-prompt.js`, `tools/create-client.js`, `package.json` | `code/eva/src/...` |
| Зависимости Евы | `npm install form-data` | в `/var/www/eva/` |
| Bundle `trade_app_v3.jsx` | `/var/www/trade/public/assets/` через `vite build` | `code/trade_app_v3.jsx` |

## Шаги (по порядку)

### 1. Подготовка папки фото

```sh
mkdir -p /var/www/trade/photos
chown -R www-data:www-data /var/www/trade/photos
chmod 755 /var/www/trade/photos
```

### 2. trade-api: код + зависимости

```sh
cd /var/www/trade
# 2.1. Скопировать с локалки:
#   - server.js (после слияния — см. ниже)
#   - photo-bases.js
#   - photos-router.js
# 2.2. Установить зависимости
npm install multer image-size --save
# 2.3. Перезапустить
pm2 restart trade-api --update-env
pm2 logs trade-api --lines 30   # проверить, что стартует без ошибок
```

**Слияние `server.js`:** в существующий `server.js` (~50 строк, прокси к `app_storage`) добавить **только** одну строку подключения роутера сразу после создания `app`:

```js
const photosRouter = require("./photos-router");
app.use("/api/photos", photosRouter(pool));
```

Это всё. Существующие endpoint-ы `/api/storage/*` не трогаются.

### 3. Nginx `location /photos/`

В существующем server-блоке `trade-abkhazia.com` (HTTPS, port 443) добавить **до** блока `location /` (так как `/photos/` более специфичный):

```nginx
location /photos/ {
    alias /var/www/trade/photos/;
    expires 1y;
    add_header Cache-Control "public, max-age=31536000, immutable";
    access_log off;
    try_files $uri =404;
}
```

Применить:
```sh
nginx -t          # проверить синтаксис
systemctl reload nginx
curl -I https://trade-abkhazia.com/photos/test.jpg   # должно вернуть 404 (папка пустая, но Nginx отвечает)
```

### 4. Eva: фото-модуль

```sh
cd /var/www/eva
# 4.1. Скопировать с локалки (через scp):
#   - src/photo/raw-photo.js
#   - src/photo/album-buffer.js
#   - src/photo/dispatcher.js
#   - src/photo/upload-client.js
#   - src/photo/handlers/client-locations.js
#   - src/photo/handlers/index.js
# 4.2. Заменить (с локалки):
#   - src/index.js
#   - src/session-state.js
#   - src/system-prompt.js
#   - src/tools/create-client.js
#   - package.json
# 4.3. Установить form-data
npm install form-data --save
# 4.4. Перезапустить
pm2 restart eva --update-env
pm2 logs eva --lines 30
```

### 5. Bundle trade_app

```sh
# Из корня репозитория: тесты → бэкап → scp (jsx + calc.mjs + ядра ADR-025) → vite build
# → проверка бандла. Имя нового бандла скрипт печатает сам.
./scripts/deploy.sh trade-app
```

> Ручная scp-цепочка отменена: вместе с `trade_app_v3.jsx` обязаны уезжать `calc.mjs` и
> три общих ядра `.cjs` — см. [deploy-frontend.md](deploy-frontend.md).

### 6. Cron — чистка `/tmp/eva` и `/tmp/api-uploads`

```sh
crontab -e
# Добавить:
0 4 * * * find /tmp/eva -mtime +1 -delete 2>/dev/null
0 4 * * * find /tmp/api-uploads -mtime +1 -delete 2>/dev/null
```

## Smoke-test (после деплоя)

### 7.1. Round-trip `/api/photos` через curl

Подготовить тестовый клиент в `trade-cat-clients` (любой существующий, например К-001). Тестовый JPG локально:

```sh
TESTFILE=/tmp/test-photo.jpg
# можно взять любой jpg, или скачать placeholder
curl -o $TESTFILE https://placehold.co/640x480.jpg

curl -X POST https://trade-abkhazia.com/api/photos \
  -F "base=client-locations" \
  -F "entityId=К-001" \
  -F "uploadedBy=smoke-test" \
  -F "fileUniqueId=smoke-$(date +%s)" \
  -F "caption=Smoke test" \
  -F "file=@$TESTFILE"

# Ожидаемый ответ:
# {"ok":true,"url":"https://trade-abkhazia.com/photos/client-locations/К-001/...","photoMeta":{...},"duplicate":false}
```

Проверить, что URL открывается в браузере и что фото попало в KV:

```sh
curl https://trade-abkhazia.com/api/storage/trade-cat-clients | python3 -m json.tool | grep -A 12 'К-001'
```

### 7.2. Идемпотентность (дубль)

Повторить тот же POST с **тем же** `fileUniqueId` → должен вернуться `200 {duplicate: true}` без перезаписи.

### 7.3. Валидация `BAD_PAYLOAD` (фикс GAP-019)

```sh
curl -X POST https://trade-abkhazia.com/api/photos \
  -F "base=client-locations" \
  -F "entityId=К-001"
# Ожидаемый: 400 {"ok":false,"error":"...","code":"BAD_PAYLOAD"}
```

### 7.4. `ENTITY_NOT_FOUND`

```sh
curl -X POST https://trade-abkhazia.com/api/photos \
  -F "base=client-locations" \
  -F "entityId=К-99999" \
  -F "uploadedBy=test" \
  -F "fileUniqueId=test-404" \
  -F "file=@$TESTFILE"
# Ожидаемый: 404 ENTITY_NOT_FOUND
```

### 7.5. DELETE

```sh
# fileUniqueId из шага 7.1
curl -X DELETE https://trade-abkhazia.com/api/photos \
  -H "Content-Type: application/json" \
  -d '{"base":"client-locations","entityId":"К-001","fileUniqueId":"smoke-XXXXX"}'
# Ожидаемый: {"ok":true}
```

## Приёмочный тест: онбординг с фото

1. В Telegram написать Еве: «новый клиент» (тестовый менеджер).
2. Пройти онбординг до конца.
3. На финальном отчёте — Ева должна попросить фото места доставки.
4. Прислать одно фото и одно фото альбомом (несколько).
5. Ева подтверждает: «Принял ✅ Витрина / вход клиента — N фото».
6. Открыть карточку клиента в trade_app → секция «Фото места доставки» → миниатюры есть, lightbox работает.
7. Удалить одно фото из UI → миниатюра пропадает, файл удалён с диска.

## Откат (если что-то сломалось)

- Если `/api/photos` падает: убрать `app.use("/api/photos", ...)` из `server.js`, `pm2 restart trade-api`. Остальное API продолжает работать.
- Если Eva падает при приёме фото: вернуть `code/eva/src/index.js` к предыдущей версии (коммит `bc5cf87` — Ит.3), `pm2 restart eva`. Текстовая часть онбординга работает.
- Если фронт сломался: откатить бандл (предыдущий `index-XXX.js` остаётся в `/var/www/trade/public/assets/` до пересборки).

## После успешного теста

- Закрыть [GAP-018](../../00-meta/gaps.md#gap-018) (`status: closed`).
- Обновить [CHANGELOG-2026-Q2](../../95-changelog/2026-Q2.md) — запись о реализации.
- Обновить [roadmap](../../65-roadmap/current.md) — Шаг 3 онбординга закрыт по 9/9.
