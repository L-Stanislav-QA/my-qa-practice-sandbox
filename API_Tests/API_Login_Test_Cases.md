# Ручное тестирование API (Project: QA Sandbox)

В данном документе представлены тест-кейсы для ручного тестирования API эндпоинтa `/api/auth/login`.
Объект тестирования: [QA Automation Sandbox](https://github.com/manikosto/qa-automation-sandbox)

# Примечание: 
Все тесты выполняются в инструменте Postman. Запросы организованы в коллекцию "QA Sandbox API". 
Для всех запросов используется базовый URL: http://localhost:8000

# Список тестовых данных

| Username | Email | Password | Role | Notes |
|:---|:---|:---:|:---|:---|
| admin | admin@buzzhive.com | admin123 | Admin | Full access, manage users, moderate content |
| moderator | mod@buzzhive.com | mod123 | Moderator | Can delete posts/comments, no user mgmt |
| alice_dev | alice@buzzhive.com | alice123 | User (active) | Active, 8 posts, many followers, verified |
| bob_photo | bob@buzzhive.com | bob123 | User | Photography posts with image URLs |
| carol_writes | carol@buzzhive.com | carol123 | User | Long-form content, technical writer |
| dave_quiet | dave@buzzhive.com | dave123 | User (private) | PRIVATE — follow request required |
| eve_new | eve@buzzhive.com | eve123 | User (new) | New user, zero posts (empty states) |
| frank_banned | frank@buzzhive.com | frank123 | User (banned) | BANNED — login fails |




## План тестирования эндпоинта POST /api/auth/register

### 1. Блок: Happy Path (P0)
* **[LOG-01](#tc-log-01-успешная-авторизация-happy-path) Успешная авторизация (Happy Path).** Проверка входа с полностью валидными данными. (Ожидаемый результат: 200 OK).
### 2. Блок: Бизнес-логика и Безопасность (P1)
* **[LOG-02](#tc-log-02-вход-заблокированного-пользователя) Вход заблокированного пользователя.** Проверка запрета доступа для пользователей со статусом `banned`.
* **[LOG-03](#tc-log-03-несуществующий-пользователь)  Несуществующий пользователь.** Попытка входа с данными, которых нет в БД.
* **[LOG-04](#tc-log-04-неверный-пароль)  Неверный пароль.** Проверка системы при вводе ошибочного пароля.
* **[LOG-05](#tc-log-05-sql-инъекция-в-поле-email) SQL-инъекция в поле Email.** Проверка устойчивости к попытке обхода авторизации через внедрение SQL-кода.
### 3. Блок: Валидация (P2)
* **[LOG-06](#tc-log-06-пустые-значения) Пустые значения.** Проверка обязательности полей логина и пароля.
* **[LOG-07](#tc-log-07-чувствительность-к-регистру-в-email) Чувствительность к регистру в Email.** Проверка, что система игнорирует регистр символов в Email при авторизации (Case Insensitivity).
* **[LOG-08](#tc-log-08-обработка-экстремально-длинных-значений-stress-validation) Обработка экстремально длинных значений.** Проверка устойчивости сервера и корректности валидации при передаче данных большого объема (Stress Validation).

---
## 2. Авторизация (LOG)
* **Инструмент:** Postman + pgweb (Database UI) + DBeaver
---

### TC-LOG-01. Успешная авторизация (Happy Path)
* **Описание:** Проверка входа с полностью валидными данными.
* **Предусловие:** Пользователь `eve@buzzhive.com` зарегистрирован и активен.
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Login**
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/login`
    3. На вкладке **Body** (тип raw/JSON) ввести данные: `{"email": "eve@buzzhive.com", "password": "eve123"}`
    4. Нажать кнопку **Send**.

* **Ожидаемый результат:** 
    * Статус: `200 OK`.
    * Ответ содержит: `access_token`, `refresh_token` и `token_type`.

--- 

### TC-LOG-02. Вход заблокированного пользователя
* **Описание:** Проверка запрета доступа для пользователей со статусом `banned`.
* **Предусловие:** 
    * Пользователь `frank_banned` имеет статус `is_active=false`. 
    * Проверить в БД: `SELECT is_active FROM users WHERE username='frank_banned';` 

* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Login**
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/login`
    3. На вкладке **Body** (тип raw/JSON) ввести подготовленные данные пользователя `frank_banned`.
    4. Нажать кнопку **Send**.
* **Ожидаемый результат:** 
    * Статус: `400 Bab Request`
    * Тело ответа:
    ```json
    {
    "detail": "Account is deactivated",
    "error_code": "BAD_REQUEST",
    "status_code": 400
    }

--- 

### TC-LOG-03. Несуществующий пользователь
* **Описание:** Попытка входа с данными, которых нет в БД.
* **Предусловие:** 
    * Данные пользователя отсутствуют в БД. 

* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Login**
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/login`
    3. На вкладке **Body** (тип raw/JSON) ввести данные: `{"email": "random@buzzhive.com", "password": "random123"}`.
    4. Нажать кнопку **Send**.
* **Ожидаемый результат:** 
    * Статус: `400 Bab Request`
    * Тело ответа:
    ```json
    {
    "detail": "Account is deactivated",
    "error_code": "BAD_REQUEST",
    "status_code": 400
    }

--- 

### TC-LOG-04. Неверный пароль
* **Описание:** Проверка системы при вводе ошибочного пароля.
* **Предусловие:** В системе существует активный пользователь с Email: `eve@buzzhive.com`.
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Login**
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/login`
    3. На вкладке **Body** (тип raw/JSON)  ввести данные: `{"email": "eve@buzzhive.com", "password": "wrong_pass_123"}`.
    4. Нажать кнопку **Send**.
* **Ожидаемый результат:** 
    * Статус: `401 Unauthorized`.
    * Тело ответа:
    ```json
    {
    "detail": "Invalid email or password",
    "error_code": "UNAUTHORIZED",
    "status_code": 401
    }
--- 

### TC-LOG-05. SQL-инъекция в поле Email
* **Описание:** Проверка устойчивости к попытке обхода авторизации через внедрение SQL-кода.
* **Предусловие:** Сервер и база данных запущены и доступны.
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Login**.
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/login`.
    3. На вкладке **Body** (тип raw/JSON) отправить данные:
       ```json
       {
         "email": "' OR 1=1 --",
         "password": "any_password"
       }
    4. Нажать кнопку **Send**.
* **Ожидаемый результат:** 
    * Статус: `422 Unprocessable Entity` (ошибка валидации формата).
    * **Критично:** Отсутствие ошибки `500 Internal Server Error` (которая означала бы, что база данных "проглотила" и попыталась исполнить этот код).

--- 

### TC-LOG-06. Пустые значения
* **Описание:** Проверка обязательности полей логина и пароля.
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Login**.
    2. На вкладке **Body** отправить пустой объект `{}`.
    3. Нажать кнопку **Send**.
* **Ожидаемый результат:** 
    * Статус: `422 Unprocessable Entity`.
    * Тело ответа:
    ```json
    {
    "detail": [
        {
            "type": "missing",
            "loc": [
                "body",
                "email"
            ],
            "msg": "Field required",
            "input": {}
        },
        {
            "type": "missing",
            "loc": [
                "body",
                "password"
            ],
            "msg": "Field required",
            "input": {}
        }
    ]
    }

--- 

### TC-LOG-07. Чувствительность к регистру в Email
* **Описание:** Проверка, что система игнорирует регистр символов в Email при авторизации (Case Insensitivity).
* **Предусловие:** В системе существует активный пользователь с Email `eve@buzzhive.com` (записан в нижнем регистре).
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Login**.
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/login`.
    3. На вкладке **Body** отправить данные: `{"email": "EVE@buzzhive.com", "password": "eve123"}`.
    4. Нажать кнопку **Send**.
* **Ожидаемый результат:** 
    * Статус: `200 OK`. 
    * Авторизация успешна (выдан токен), так как для Email-адресов регистр не должен иметь значения.

--- 

### TC-LOG-08. Обработка экстремально длинных значений (Stress Validation)
* **Описание:** Проверка устойчивости сервера и корректности валидации при передаче данных большого объема.
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Login**.
    2. Подготовить строку длиной 1000+ символов.
    3. На вкладке **Body** (тип raw/JSON) в поле `password` вставить эту строку.
    4. Нажать кнопку **Send**.
* **Ожидаемый результат:** 
    * Статус: `422 Unprocessable Entity`, `413 Payload Too Large` или `401 Unauthorized`.
    * **Критично:** Отсутствие ошибки `500 Internal Server Error`, что означало бы падение сервера или необработанное исключение.