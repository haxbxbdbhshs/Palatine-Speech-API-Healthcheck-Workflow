# Palatine Speech API — Healthcheck Workflow

### Начало работы:
1. Установить Desctop Docker или другую среду для работы с n8n
2. Установить n8n версии 1.118 и выше
```bash
docker run -it --rm ^
  -p 5678:5678 ^
  -v %USERPROFILE%\.n8n:/home/node/.n8n ^
  -e N8N_USER_FOLDER=/home/node/.n8n ^
  n8nio/n8n:latest
 ```
3. Запустить среду

### Как использовать
1.Импортировать файл Palatine Speech API.json как воркфлоу

2.Создать credentials (palatine_bearer, telegram_bot).

3.Указать реальный telegramChatId.

4.При необходимости заменить fileUrl на ваш тестовый аудиофайл.

5.Запустить workflow вручную — проверить Execution.

6.Включить активацию.

### Процесс работы 
Периодически проверяет работоспособность эндпоинтов Palatine Speech (транскрибация, TTS и др.).
При ошибке — отправляет уведомление в Telegram.
Логи пишутся в Execution.
Требуемые Credentials
Создайте в n8n два credentials:

|ИМЯ            |     ТИП          |            ПОЛЯ             |
------------------------------------------------------------------
|palatine_bearer | Bearer Auth     |    ваш API-ключ Palatine|
|telegram_bot    |  Telegram API    |         Access Token    |


### Общая структура workflow
Schedule Trigger → Config → Download audio → Prepare Request → HTTP Request → Evaluate Result → IF → Telegram / Log
1. Schedule Trigger
Назначение: запуск workflow по расписанию.
Настройка:
Cron expression: каждые 5 минут + 1-30 сек 
Опционально: добавьте рандомный сдвиг через Expression.

2. Config (Set node)
Назначение: центральное хранилище настроек — легко править 1 ноду, а не 10.

3. Download audio
Назначение: скачивание файла для его дальнейшей отправки в palatine speech

4. Prepare Request (Code node)
Назначение: формирует параметры запроса для каждого check’а.
Что делает:
Разбирает checks[] из Config (поддерживает и объект, и массив).
Готовит:
url (база + endpoint),
fileUrl — ссылка на аудиофайл,
formDataFields — доп. параметры (language, format),
checkName, successCriteria — для последующей оценки.
Чистит apiBaseUrl от пробелов.

5. HTTP Request
Назначение: отправка запроса к Palatine Speech API.
Ключевые настройки:
Authentication: Bearer Auth → palatine_bearer
Send Body (+)
Body Content Type: Form Data
Specify Fields(опционально, можно добавить в конфиг):
language = {{ $json.formDataFields.language }}
format = {{ $json.formDataFields.format }}
model = {{ $json.formDataFields.model }}
file = (имя поля, которое ожидает API)

6. Evaluate Result (Code node)
Назначение: анализ ответа — успешно или ошибка?
Что проверяет:
HTTP-статус (statusCode == 200),
структуру JSON (status == "done"),
наличие результата (text не пустой),
сетевые ошибки (404, таймаут и т.д.).
Возвращает:

7. IF (isOk?)
Назначение: ветвление по результату.
Условие - ={{ $json.isOk === false }}
true → идёт в Telegram Notify + Log to console
false → идёт в Log to Console

8. Telegram Notify
Назначение: отправка алерта при ошибке.
9. Final log
Назначение: записывает данные о каждой проверке
