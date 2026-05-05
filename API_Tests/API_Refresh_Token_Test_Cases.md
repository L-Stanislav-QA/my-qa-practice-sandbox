# Ручное тестирование API (Project: QA Sandbox)

В данном документе представлены тест-кейсы для ручного тестирования API эндпоинтa `/api/auth/refresh`.
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

---

## План тестирования эндпоинта POST /api/auth/refresh

### 1. Блок: Happy Path (P0)
* **[REF-01](#tc-ref-01-успешное-обновление-токена-happy-path) Успешное обновление токена (Happy Path).** Проверка обновления access_token с валидным refresh_token. (Ожидаемый результат: 200 OK).

### 2. Блок: Бизнес-логика и Безопасность (P1)
* **[REF-02](#tc-ref-02-использование-невалидного-refresh-token) Использование невалидного Refresh Token.** Попытка обновить токен с несуществующим refresh_token.
* **[REF-03](#tc-ref-03-использование-истёкшего-refresh-token) Использование истёкшего Refresh Token.** Проверка поведения системы при передаче протухшего токена.
* **[REF-04](#tc-ref-04-повторное-использование-refresh-token) Повторное использование Refresh Token.** Проверка, что после успешного обновления старый refresh_token инвалидируется и не может быть использован повторно (защита от token replay attack).
* **[REF-05](#tc-ref-05-использование-access-token-вместо-refresh-token) Использование Access Token вместо Refresh Token.** Проверка, что система различает типы токенов и не принимает access_token в качестве refresh_token.

### 3. Блок: Валидация (P2)
* **[REF-06](#tc-ref-06-пустое-тело-запроса) Пустое тело запроса.** Проверка обязательности поля refresh_token.
* **[REF-07](#tc-ref-07-отсутствие-поля-refresh-token) Отсутствие поля refresh_token.** Проверка валидации обязательного поля.
* **[REF-08](#tc-ref-08-некорректный-формат-refresh-token) Некорректный формат Refresh Token.** Передача строки, не соответствующей формату JWT (например, просто "invalid_token" или случайная строка).
* **[REF-09](#tc-ref-09-экстремально-длинная-строка-в-refresh-token) Экстремально длинная строка в refresh_token.** Проверка устойчивости к Stress Validation (строка 5000+ символов).

---

## Тест-кейсы (TC): Обновление токена (REF)

* **Инструмент:** Postman + pgweb (Database UI) + DBeaver

---

### TC-REF-01. Успешное обновление токена (Happy Path)
* **Описание:** Проверка обновления access_token с валидным refresh_token. Делаем эту проверку, потому что это основной сценарий использования endpoint'а — пользователь должен иметь возможность получить новую пару токенов без повторной авторизации, что критично для UX и безопасности (короткоживущие access_token снижают риски при компрометации).
* **Предусловие:** 
    * Пользователь `eve@buzzhive.com` авторизован в системе.
    * Получены `access_token` и `refresh_token` через эндпоинт `POST /api/auth/login`.
    * Сохранить полученный `refresh_token` для использования в тесте.

* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Refresh Token**
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/refresh`
    3. На вкладке **Body** (тип raw/JSON) ввести данные: 
    ```json
    {
        "refresh_token": "<полученный_ранее_refresh_token>"
    }
    ```
    4. Нажать кнопку **Send**.

* **Ожидаемый результат:** 
    * Статус: `200 OK`.
    * Ответ содержит JSON-объект с обязательными полями:
        * `"access_token"` (новая строка JWT, отличная от предыдущей).
        * `"refresh_token"` (новая строка JWT, отличная от предыдущей).
        * `"token_type": "bearer"`.
    * Оба токена имеют валидный формат JWT (три части, разделённые точками).

---

### TC-REF-02. Использование невалидного Refresh Token
* **Описание:** Попытка обновить токен с несуществующим refresh_token. Делаем эту проверку, потому что система должна отклонять токены, которые не были выданы сервером, предотвращая несанкционированный доступ через поддельные токены.
* **Предусловие:** Сервер и база данных запущены и доступны.

* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Refresh Token**
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/refresh`
    3. На вкладке **Body** (тип raw/JSON) ввести данные с фальшивым токеном:
    ```json
    {
        "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c"
    }
    ```
    4. Нажать кнопку **Send**.

* **Ожидаемый результат:** 
    * Статус: `401 Unauthorized`.
    * Тело ответа содержит информацию об ошибке:
    ```json
    {
        "detail": "Invalid or expired refresh token",
        "error_code": "UNAUTHORIZED",
        "status_code": 401
    }
    ```

---

### TC-REF-03. Использование истёкшего Refresh Token
* **Описание:** Проверка поведения системы при передаче протухшего токена. Делаем эту проверку, потому что система должна корректно обрабатывать expired токены, заставляя пользователя авторизоваться заново, что является важным элементом безопасности (предотвращение использования старых токенов после их lifetime).
* **Предусловие:** 
    * Получить валидный `refresh_token` через `/api/auth/login`.
    * Дождаться истечения срока действия токена (согласно настройкам JWT в приложении) ИЛИ использовать специально подготовленный истёкший токен из тестовых данных (если поддерживается в тестовой среде).

* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Refresh Token**
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/refresh`
    3. На вкладке **Body** (тип raw/JSON) ввести данные с истёкшим токеном:
    ```json
    {
        "refresh_token": "<expired_refresh_token>"
    }
    ```
    4. Нажать кнопку **Send**.

* **Ожидаемый результат:** 
    * Статус: `401 Unauthorized`.
    * Тело ответа:
    ```json
    {
        "detail": "Invalid or expired refresh token",
        "error_code": "UNAUTHORIZED",
        "status_code": 401
    }
    ```

---

### TC-REF-04. Повторное использование Refresh Token
* **Описание:** Проверка, что после успешного обновления старый refresh_token инвалидируется и не может быть использован повторно (защита от token replay attack). Делаем эту проверку, потому что это критический механизм безопасности — если злоумышленник перехватит refresh_token, он не сможет использовать его повторно после того, как легитимный пользователь уже обновил свою пару токенов.
* **Предусловие:** 
    * Пользователь `eve@buzzhive.com` авторизован.
    * Получены `access_token` и `refresh_token` через `/api/auth/login`.

* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Refresh Token**
    2. На вкладке **Body** отправить первый раз валидный `refresh_token`:
    ```json
    {
        "refresh_token": "<валидный_refresh_token>"
    }
    ```
    3. Нажать **Send**. Сохранить ответ (должен быть 200 OK с новыми токенами).
    4. Немедленно отправить тот же самый `refresh_token` повторно (не используя новый, полученный в п.3).
    5. Нажать **Send**.

* **Ожидаемый результат:** 
    * Первый запрос: Статус `200 OK`, получены новые токены.
    * Второй запрос (с тем же старым токеном): Статус `401 Unauthorized`.
    * Тело ответа второго запроса:
    ```json
    {
        "detail": "Invalid or expired refresh token",
        "error_code": "UNAUTHORIZED",
        "status_code": 401
    }
    ```
    * **Критично:** Система не должна выдавать новую пару токенов при повторном использовании старого refresh_token.

---

### TC-REF-05. Использование Access Token вместо Refresh Token
* **Описание:** Проверка, что система различает типы токенов и не принимает access_token в качестве refresh_token. Делаем эту проверку, потому что путаница токенов может привести к серьёзным уязвимостям — access_token имеет другой lifecycle и права, и его использование для refresh операций нарушает архитектуру безопасности.
* **Предусловие:** 
    * Пользователь `eve@buzzhive.com` авторизован.
    * Получены `access_token` и `refresh_token` через `/api/auth/login`.

* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Refresh Token**
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/refresh`
    3. На вкладке **Body** (тип raw/JSON) ввести `access_token` в поле `refresh_token`:
    ```json
    {
        "refresh_token": "<полученный_access_token>"
    }
    ```
    4. Нажать кнопку **Send**.

* **Ожидаемый результат:** 
    * Статус: `401 Unauthorized`.
    * Тело ответа:
    ```json
    {
        "detail": "Invalid or expired refresh token",
        "error_code": "UNAUTHORIZED",
        "status_code": 401
    }
    ```

---

### TC-REF-06. Пустое тело запроса
* **Описание:** Проверка обязательности поля refresh_token. Делаем эту проверку, потому что endpoint должен корректно валидировать входящие данные и отклонять запросы без обязательных полей, предотвращая некорректные обращения к серверу.
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Refresh Token**.
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
                    "refresh_token"
                ],
                "msg": "Field required",
                "input": {}
            }
        ]
    }
    ```

---

### TC-REF-07. Отсутствие поля refresh_token
* **Описание:** Проверка валидации обязательного поля. Делаем эту проверку, потому что при отправке JSON с другими полями, но без refresh_token, система должна чётко указать на отсутствие обязательного параметра.
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Refresh Token**.
    2. На вкладке **Body** (тип raw/JSON) отправить:
    ```json
    {
        "some_field": "some_value"
    }
    ```
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
                    "refresh_token"
                ],
                "msg": "Field required",
                "input": {
                    "some_field": "some_value"
                }
            }
        ]
    }
    ```

---

### TC-REF-08. Некорректный формат Refresh Token
* **Описание:** Передача строки, не соответствующей формату JWT (например, просто "invalid_token" или случайная строка). Делаем эту проверку, потому что система должна валидировать структуру токена перед его обработкой, предотвращая ошибки парсинга и потенциальные уязвимости.
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Refresh Token**.
    2. На вкладке **Body** (тип raw/JSON) отправить:
    ```json
    {
        "refresh_token": "this_is_not_a_valid_jwt_token"
    }
    ```
    3. Нажать кнопку **Send**.

* **Ожидаемый результат:** 
    * Статус: `401 Unauthorized` или `422 Unprocessable Entity`.
    * Тело ответа (для 401):
    ```json
    {
        "detail": "Invalid or expired refresh token",
        "error_code": "UNAUTHORIZED",
        "status_code": 401
    }
    ```
    * **Критично:** Отсутствие ошибки `500 Internal Server Error`.

---

### TC-REF-09. Экстремально длинная строка в refresh_token
* **Описание:** Проверка устойчивости к Stress Validation (строка 5000+ символов). Делаем эту проверку, потому что система должна корректно обрабатывать аномально большие данные без падения сервера, защищаясь от потенциальных DoS-атак и переполнения буфера.
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Refresh Token**.
    2. Подготовить строку длиной 5000+ символов (например, повторение "a" 5000 раз).
    3. На вкладке **Body** (тип raw/JSON) в поле `refresh_token` вставить эту строку:
    ```json
    {
        "refresh_token": "aaaaaaaaaaa...(5000+ символов)"
    }
    ```
    4. Нажать кнопку **Send**.

* **Ожидаемый результат:** 
    * Статус: `401 Unauthorized`, `422 Unprocessable Entity` или `413 Payload Too Large`.
    * **Критично:** Отсутствие ошибки `500 Internal Server Error`, что означало бы падение сервера или необработанное исключение.