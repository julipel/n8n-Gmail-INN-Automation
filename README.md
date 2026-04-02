# n8n Gmail INN Automation

Автоматизация в `n8n` для обработки входящих писем Gmail с заявками и проверкой дублей по ИНН в Google Sheets.

## Что делает workflow

1. Отслеживает входящие письма Gmail по фильтру темы (`SUBJECT_FILTER_QUERY`)
2. Получает письмо целиком
3. Извлекает данные из тела письма:
   - `sender_email`
   - `inn`
   - `org_name`
   - `org_address`
4. Читает строки из Google Sheets
5. Проверяет, существует ли ИНН
6. Если ИНН новый:
   - добавляет строку в Google Sheets
   - отправляет ответное письмо с вложением `.docx`
7. Если ИНН уже есть:
   - отправляет ответное письмо без вложения

---

## Стек

- `n8n` (self-hosted / cloud)
- `Gmail Trigger` + `Gmail`
- `Google Sheets`
- `Code` nodes (JavaScript, без внешних библиотек)
- `Google Drive Download` **или** `Read Binary File` (для вложения)

---

## Структура workflow

`Gmail Trigger` → `Gmail Get Message` → `Code - Parse Email` → `Google Sheets - Read All Rows` → `Code - Check INN Exists` → `IF - INN Exists?`

- `TRUE` → `Gmail Send - Duplicate INN`
- `FALSE` → `Google Sheets - Append Row` → `Google Drive Download / Read Binary File` → `Gmail Send - Success With Attachment`

---

## Требуемые credentials

Создать в n8n:

- `Gmail OAuth2`
- `Google Sheets OAuth2`
- (опционально) `Google Drive OAuth2` — если вложение берется из Drive

---

## Placeholders (обязательные для замены)

- `GOOGLE_SHEET_ID`
- `SHEET_NAME_MAIN` (например: `requests`)
- `SUBJECT_FILTER_QUERY` (например: `subject:"Новый проект Пардус-р" in:inbox -from:me`)
- `DOCX_FILE_PATH` (только для варианта с `Read Binary File`)
- `GOOGLE_DRIVE_FILE_ID` (только для варианта с `Google Drive Download`)

---

## Формат таблицы Google Sheets

На листе должна быть строка заголовков:

- `created_at`
- `sender_email`
- `inn`
- `org_name`
- `org_address`
- `message_date`
- `subject`
- `message_id`

---

## Рекомендованный формат дат

- `created_at`:
  - `{{$now.setZone('Europe/Moscow').toFormat('yyyy-LL-dd HH:mm:ss')}}`
- `message_date`:
  - нормализовать к такому же формату (в Code node или expression в Append Row)

---

## Извлечение данных из письма

Парсер поддерживает:

- ИНН по метке `ИНН:` (приоритет) или regex `10/12` цифр
- Название по меткам:
  - `Название организации:`
  - `Наименование:`
  - `Организация:`
  - `Компания:`
  - `Название:`
- Адрес по меткам:
  - `Адрес:`
  - `Юридический адрес:`
  - `Фактический адрес:`

HTML преобразуется в plain text (минимальная очистка).

---

## Важные замечания

### 1) Ветки IF
Проверь соединения:

- `TRUE (inn_exists = true)` → `Gmail Send - Duplicate INN`
- `FALSE (inn_exists = false)` → `Append Row` + вложение + success email

### 2) Тип `inn_exists`
В `Code - Check INN Exists` возвращай строго boolean:

```js
inn_exists: Boolean(exists)
```

### 3) Локальные файлы в self-hosted
Если используешь `Read Binary File`, в n8n может быть ограничение на allowed path.  
Для n8n 2.4.8 обычно разрешен путь:

- `/home/node/.n8n-files`

---

## Запуск self-hosted (docker compose)

Пример `docker-compose.yml`:

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n_compose
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - TZ=Europe/Moscow
      - N8N_ENCRYPTION_KEY=change_me_32_chars_min
    volumes:
      - n8n_data:/home/node/.n8n
      - C:/n8n/files:/home/node/.n8n-files

volumes:
  n8n_data:
```

Запуск:

```bash
docker compose -p n8nlocal up -d
```

---

## Импорт workflow

- В UI: `Workflows` → `Import from file`
- Выбрать `gmail-inn-workflow.json`
- Подключить credentials
- Обновить placeholders
- `Activate`

---

## Тест-план

1. Отправить письмо с новым ИНН:
   - запись добавляется в Sheets
   - приходит success-письмо с вложением
2. Отправить письмо с тем же ИНН:
   - запись не добавляется
   - приходит duplicate-письмо без вложения
3. Проверить, что фильтр темы не цепляет исходящие (`-from:me`, `in:inbox`)

---

## Безопасность

- Не хранить OAuth токены в репозитории
- Не коммитить `.env` и чувствительные данные
- Для публичного репо использовать placeholders вместо реальных ID

---

## Лицензия

Укажите нужную лицензию (например MIT) в отдельном файле `LICENSE`.
