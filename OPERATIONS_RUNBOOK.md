# Руководство по эксплуатации (Runbook)

Документ фиксирует проверенные команды и процедуры для продакшн-обслуживания бота.

## 1. Базовые команды управления ботом

Основной сервис:
- `bot.service`
- Рабочая директория: `/root/ASCO/current`

Проверка статуса:
```bash
systemctl status bot --no-pager -l
systemctl is-active bot
```

Перезапуск/остановка/запуск:
```bash
systemctl restart bot
systemctl stop bot
systemctl start bot
```

Логи в реальном времени:
```bash
journalctl -fu bot
```

Последние 200 строк:
```bash
journalctl -u bot -n 200 --no-pager -l
```

Проверка webhook-строк и ошибок:
```bash
journalctl -u bot -n 400 --no-pager -l | grep -E "URL вебхука|bad webhook|Ошибка при запуске|POST /webhook"
```

Проверка текущих webhook-переменных:
```bash
grep -nE "^WEBHOOK_HOST=|^WEBHOOK_URL=" /root/ASCO/current/.env
```

## 2. Миграция webhook на новый домен

Пример: перенос с `bot.ascorobot.ru` на `ascorobot.org`.

### 2.1 Проверка DNS
На авторитативных NS домен должен уже указывать на сервер (`31.128.32.169`):
```bash
dig +short A ascorobot.org @ns1.dyna-ns.net
dig +short A ascorobot.org @ns2.dyna-ns.net
dig +short A www.ascorobot.org @ns1.dyna-ns.net
dig +short A www.ascorobot.org @ns2.dyna-ns.net
```

### 2.2 Выпуск сертификата
```bash
certbot certonly --nginx -d ascorobot.org -d www.ascorobot.org --non-interactive --agree-tos --register-unsafely-without-email
```

### 2.3 Nginx
Проверить конфиг и перечитать:
```bash
nginx -t
systemctl reload nginx
```

Проверка HTTPS endpoint:
```bash
curl -I https://ascorobot.org/webhook/
```
Нормальный ответ приложения: `204`.

### 2.4 Переключение `.env`
В файле `/root/ASCO/current/.env`:
```env
WEBHOOK_HOST="https://ascorobot.org"
WEBHOOK_URL="https://ascorobot.org/webhook/"
```

Перезапуск:
```bash
systemctl restart bot
```

### 2.5 Финальная проверка в Telegram
```bash
TOKEN=$(grep '^API_TOKEN=' /root/ASCO/current/.env | cut -d= -f2- | sed 's/^"//; s/"$//')
curl -s "https://api.telegram.org/bot${TOKEN}/getWebhookInfo"
```
Ожидаемо:
- `url`: новый домен
- `last_error_message`: `null`
- `pending_update_count`: не растет

## 3. Платежи и выдача подписок: как диагностировать

Важно: успешная оплата и успешная выдача ключа - разные этапы.

Наблюдавшиеся кейсы:
- YooKassa: `cancelled` с причиной `expired_on_confirmation` (пользователь не завершил оплату вовремя).
- Ошибка выдачи ключа: `Ключ не найден после создания: <id>` (проблема кластера/панели, не webhook).

### 3.1 SQL-проверка платежей пользователя
```sql
SELECT id, tg_id, amount, payment_system, status, payment_id, created_at, metadata
FROM payments
WHERE tg_id = <USER_ID>
ORDER BY id DESC;
```

### 3.2 SQL-проверка ключей пользователя
```sql
SELECT client_id, tg_id, tariff_id, created_at, expiry_time, server_id, key
FROM keys
WHERE tg_id = <USER_ID>
ORDER BY created_at DESC;
```

### 3.3 Логи по конкретному пользователю
```bash
journalctl -u bot -n 1200 --no-pager -l | grep "<USER_ID>"
```

### 3.4 Логи ошибок выдачи
```bash
journalctl -u bot -n 1200 --no-pager -l | grep -E "Ключ не найден после создания|create_key|update_payment_status|yookassa|cryptobot"
```

## 4. Чистка логов и контроль заполнения диска

### 4.1 Быстрая диагностика
```bash
df -h
du -xhd1 /var | sort -h
du -xhd1 /var/log | sort -h
journalctl --disk-usage
```

### 4.2 Разовая чистка journald
```bash
journalctl --vacuum-size=200M
journalctl --disk-usage
```

### 4.3 Постоянные лимиты journald
Файл `/etc/systemd/journald.conf.d/99-size-limit.conf`:
```ini
[Journal]
SystemMaxUse=200M
SystemKeepFree=500M
MaxRetentionSec=2week
```

Применить:
```bash
systemctl restart systemd-journald
journalctl --disk-usage
```

## 5. Nginx и сертификаты

Проверка синтаксиса и перезагрузка:
```bash
nginx -t
systemctl reload nginx
```

Проверка установленных сертификатов:
```bash
certbot certificates
```

Ручное обновление (dry-run):
```bash
certbot renew --dry-run
```

## 6. Управление DS Discord ботом (`/root/ASCO/DS_bot`)

Запуск:
```bash
cd /root/ASCO/DS_bot
nohup ./venv/bin/python ./main.py >> ./nohup.out 2>&1 < /dev/null &
```

Проверка процесса:
```bash
pgrep -af "DS_bot/main.py"
```

Просмотр последних логов:
```bash
tail -n 100 /root/ASCO/DS_bot/nohup.out
```

Остановка:
```bash
pkill -f "/root/ASCO/DS_bot/venv/bin/python /root/ASCO/DS_bot/main.py"
```

Типовая ошибка:
- `disnake.errors.LoginFailure: Improper token has been passed.`
- Причина: неверный `TOKEN` в `/root/ASCO/DS_bot/.env`.

## 7. Полезные one-liners

Топ крупных лог-файлов:
```bash
find /var/log -type f -printf "%s %p\n" | sort -nr | head -n 30
```

Проверка, что бот слушает webhook локально:
```bash
curl -I http://127.0.0.1:3001/webhook/
```

Проверка webhook через внешний домен:
```bash
curl -I https://ascorobot.org/webhook/
```

Проверка активных сервисов бота:
```bash
systemctl list-units --type=service | grep -E "bot|cloudflared"
```

## 8. Рекомендации по улучшению наблюдаемости

Чтобы точнее разделять «оплачено» и «выдано», добавить:
1. `keys.issue_source` (`payment|trial|gift|admin|referral|manual`)
2. `keys.payment_id` (связь с `payments.payment_id`)
3. запись причины отмены провайдера в `payments.metadata`
