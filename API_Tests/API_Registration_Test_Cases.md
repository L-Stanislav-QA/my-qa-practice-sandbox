# Ручное тестирование API (Project: QA Sandbox)

В данном документе представлены тест-кейсы для ручного тестирования API эндпоинтa `/api/auth/register`.
Объект тестирования: [QA Automation Sandbox](https://github.com/manikosto/qa-automation-sandbox)

# Примечание: 
Все тесты выполняются в инструменте Postman. Запросы организованы в коллекцию "QA Sandbox API". 
Для всех запросов используется базовый URL: http://localhost:8000

---

## 1. Регистрация (REG)
* **Инструмент:** Postman + pgweb (Database UI) + DBeaver

## План тестирования эндпоинта POST /api/auth/register
**Проверка Happy Path**

* **[REG-01](#tc-reg-01-успешная-регистрация-нового-пользователя-happy-path) Успешная регистрация нового пользователя (Happy Path).** Регистрация с полным набором валидных данных. (Ожидаемый результат: 201 Created).

**Проверки на уникальность (Uniqueness)**

* **[REG-02](#tc-reg-02-регистрация-на-уже-существующий-email) Регистрация на уже существующий Email.** Регистрация на почту, которая уже есть в БД. (Ожидаемый результат: 409 Conflict).

* **[REG-03](#tc-reg-03-регистрация-на-уже-существующий-username) Регистрация на уже существующий Username.** Регистрация с ником, который уже занят. (Ожидаемый результат: 409 Conflict).

**Валидация обязательных полей (Required Fields)**

* **[REG-04](#tc-reg-04-отсутствие-email-при-регистрации) Отсутствие Email при регистрации.** Удалить поле email из JSON. (Ожидаемый результат: 422 Unprocessable Entity).

* **[REG-05](#tc-reg-05-отсутствие-password-при-регистрации) Отсутствие Password при регистрации.** Удалить поле password из JSON. (Ожидаемый результат: 422 Unprocessable Entity).

* **[REG-06](#tc-reg-06-пустое-тело-запроса-при-регистрации) Пустое тело запроса при регистрации.** Отправить просто {}. (Ожидаемый результат: 422 Unprocessable Entity).

**Валидация форматов (Format/Data Integrity)**

* **[REG-07](#tc-reg-07-некорректный-формат-email-при-регистрации) Некорректный формат Email при регистрации.** Отправить "test_user_001example.com" (без @). (Ожидаемый результат: 422 Unprocessable Entity).

* **[REG-08](#tc-reg-08-короткий-пароль-при-регистрации) Короткий пароль при регистрации.** Отправить пароль из 1 символа. (Ожидаемый результат: 422 Unprocessable Entity).

* **[REG-09](#tc-reg-09-username-на-нижней-границе-3-символа) Минимально допустимая длина Username (3 символа).********** Проверка нижней границы.

* **[REG-10](#tc-reg-10-username-короче-минимально-допустимого-2-символа) Username короче минимального (2 символа).** Негативный тест на нижнюю границу.

* **[REG-11](#tc-reg-11-username-длиннее-максимального-31-символ) Username длиннее максимального (31 символ).** Негативный тест на верхнюю границу (лимит 30).

* **[REG-12](#tc-reg-12-использование-недопустимых-символов-в-username-кириллица) Недопустимые символы в Username.** Попытка использовать кириллицу или спецсимволы (согласно regex в Swagger).

* **[REG-13](#tc-reg-13-пароль-на-нижней-границе-6-символов) Пароль на нижней границе (6 символов).** Позитивный тест.

* **[REG-14](#tc-reg-14-превышение-максимальной-длины-display-name-101-символ) Слишком длинный Display Name (более 100 символов).** Проверка устойчивости БД.

**Проверки безопасности (Security Tests)**
* [REG-15](#tc-reg-15-проверка-на-sql-инъекцию-в-поле-email) Проверка на SQL-инъекцию в поле Email

* [REG-16](#tc-reg-16-проверка-на-xss-уязвимость-в-поле-display-name) Проверка на XSS-уязвимость в поле Display Name
---



## 1. Тест-кейсы (TC): Регистрация (REG)

### TC-REG-01. Успешная регистрация нового пользователя (Happy Path)

* **Метод:** `POST /api/auth/register`
* **Предусловие:** Email `test_user_001@example.com` отсутствует в таблице `users`.
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Register**.
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/register`.
    3. На вкладке **Body** (тип raw/JSON) ввести данные:
    ```json
        {
            "email": "test_user_001@example.com",
            "username": "test_user_001",
            "password": "test_psw_001",
            "display_name": "test_user_name"
        }
    ```
    4. Нажать кнопку **Send**.
    5. Открыть **pgweb** (или другой клиент БД) и выполнить запрос:
        `SELECT * FROM users WHERE email = 'test_user_001@example.com';`

* **Ожидаемый результат:**
* **В Postman:** Статус-код: `201 Created`.
* В теле ответа вернулся JSON-объект пользователя со следующими обязательными полями:
    * `"id"` (строка в формате UUID).
    * `"email"`, `"username"`, `"display_name"` (соответствуют отправленным).
    * `"is_active": true`.
    * `"created_at"` и `"updated_at"` (содержат актуальную дату).
    * `"followers_count": 0`, `"posts_count": 0`.
    * **Важно:** Поле `"password"` отсутствует в теле ответа.

* **В Базе Данных:**
    * Запрос `SELECT` вернул ровно одну запись.
    * Значения в колонках `email`, `username`, `display_name` совпадают с отправленными.
    * Пароль в колонке `hashed_password` хранится в зашифрованном виде.



---



### TC-REG-02. Регистрация на уже существующий Email.

* **Метод:** `POST /api/auth/register`
* **Предусловие:** Выполнить `SELECT * FROM users WHERE email = 'test_user_001@example.com';` и зафиксировать значения полей.
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Register**.
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/register`.
    3. На вкладке **Body** (тип raw/JSON) ввести данные:
    ```json
        {
            "email": "test_user_001@example.com",
            "username": "test_user_001",
            "password": "test_psw_001",
            "display_name": "test_user_name1"
        }
    ```
    4. Нажать кнопку **Send**.
    5. Открыть **pgweb** (или другой клиент БД) и выполнить запрос:
        `SELECT * FROM users WHERE email = 'test_user_001@example.com';`  

* **Ожидаемый результат:**
* **В Postman:**
    * Статус-код: `409 Conflict`.
    * В теле ответа вернулся JSON-объект: 
    ```json
    {
    "detail": "User with this email or username already exists",
    "error_code": "CONFLICT",
    "status_code": 409
    }
    ```
* **В Базе Данных:**
    * Новая запись не создана (количество строк по данному email по-прежнему = 1).
    * Значения полей остались идентичны тем, что были до выполнения запроса (защита от несанкционированного UPDATE).



 ---



  ## TC-REG-03. Регистрация на уже существующий Username.
* **Метод:** `POST /api/auth/register`
* **Предусловие:** Выполнить `SELECT * FROM users WHERE username = 'test_user_001';` и зафиксировать значения полей.
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Register**.
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/register`.
    3. На вкладке **Body** (тип raw/JSON) ввести данные:
    ```json
        {
            "email": "test_user_002@example.com",
            "username": "test_user_001",
            "password": "test_psw_002",
            "display_name": "test_user_name2"
        }
    ```
    4. Нажать кнопку **Send**.
    5. Открыть **pgweb** (или другой клиент БД) и выполнить запрос:
        `SELECT * FROM users WHERE username = 'test_user_001';`  

* **Ожидаемый результат:**
* **В Postman:**
    * Статус-код: `409 Conflict`.
    * В теле ответа вернулся JSON-объект: 
    ```json
    {
    "detail": "User with this email or username already exists",
    "error_code": "CONFLICT",
    "status_code": 409
    }
    ```
* **В Базе Данных:**
    * Новая запись не создана (количество строк по данному username по-прежнему = 1).
    * Значения полей остались идентичны тем, что были до выполнения запроса (защита от несанкционированного UPDATE).



---


## TC-REG-04. Отсутствие Email при регистрации..
* **Метод:** `POST /api/auth/register`
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Register**.
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/register`.
    3. На вкладке **Body** (тип raw/JSON) ввести данные:
    ```json
        {
            "username": "test_user_001",
            "password": "test_psw_001",
            "display_name": "test_user_name1"
        }
    ```
    4. Нажать кнопку **Send**.
   

* **Ожидаемый результат:**
* **В Postman:**
    * Статус-код: `422 Unprocessable Entity`.
    * В теле ответа вернулся JSON-объект: 
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
                    "input": {
                        "username": "test_user_001",
                        "password": "test_psw_001",
                        "display_name": "test_user_name1"
                    }
                }
            ]
        } 



---




* ## TC-REG-05. Отсутствие Password при регистрации.
* **Метод:** `POST /api/auth/register`
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Register**.
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/register`.
    3. На вкладке **Body** (тип raw/JSON) ввести данные:
    ```json
        {
            "email": "test_user_002@example.com",
            "username": "test_user_001",
            "display_name": "test_user_name2"
        }
    ```
    4. Нажать кнопку **Send**. 

* **Ожидаемый результат:**
* **В Postman:**
    * Статус-код: `422 Unprocessable Entity`.
    * В теле ответа вернулся JSON-объект: 
    ```json
        {
    "detail": [
        {
            "type": "missing",
            "loc": [
                "body",
                "password"
            ],
            "msg": "Field required",
            "input": {
                "email": "test_user_002@example.com",
                "username": "test_user_001",
                "display_name": "test_user_name2"
            }
        }
    ]
        }    



---



* ## TC-REG-06. Пустое тело запроса при регистрации.
* **Метод:** `POST /api/auth/register`
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Register**.
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/register`.
    3. На вкладке **Body** (тип raw/JSON) ввести данные:
    ```json
        {}
    ```
    4. Нажать кнопку **Send**. 

* **Ожидаемый результат:**
* **В Postman:**
    * Статус-код: `422 Unprocessable Entity`.
    * В теле ответа вернулся JSON-объект: 
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
                "username"
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
        },
        {
            "type": "missing",
            "loc": [
                "body",
                "display_name"
            ],
            "msg": "Field required",
            "input": {}
        }
    ]
    }



---




* ## TC-REG-07. Некорректный формат Email при регистрации.
* **Метод:** `POST /api/auth/register`
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Register**.
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/register`.
    3. На вкладке **Body** (тип raw/JSON) ввести данные:
    ```json
        {
    "email": "test_user_001example.com",
    "username": "test_user_001",
    "password": "test_psw_001",
    "display_name": "test_user_name1"
        }
    ```
    4. Нажать кнопку **Send**. 

* **Ожидаемый результат:**
* **В Postman:**
    * Статус-код: `422 Unprocessable Entity`.
    * В теле ответа вернулся JSON-объект: 
    ```json
    {
    "detail": [
        {
            "type": "value_error",
            "loc": [
                "body",
                "email"
            ],
            "msg": "value is not a valid email address: An email address must have an @-sign.",
            "input": "test_user_001example.com",
            "ctx": {
                "reason": "An email address must have an @-sign."
            }
        }
    ]
    }   



---



* ## TC-REG-08. Короткий пароль при регистрации.
* **Метод:** `POST /api/auth/register`
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Register**.
    2. Проверить, что URL запроса: `http://localhost:8000/api/auth/register`.
    3. На вкладке **Body** (тип raw/JSON) ввести данные:
    ```json
        {
    "email": "test_user_001example.com",
    "username": "test_user_001",
    "password": "1",
    "display_name": "test_user_name1"
        }
    ```
    4. Нажать кнопку **Send**. 

* **Ожидаемый результат:**
* **В Postman:**
    * Статус-код: `422 Unprocessable Entity`.
    * В теле ответа вернулся JSON-объект: 
    ```json
    {
        "detail": [
            {
                "type": "string_too_short",
                "loc": [
                    "body",
                    "password"
                ],
                "msg": "String should have at least 6 characters",
                "input": "1",
                "ctx": {
                    "min_length": 6
                }
            }
        ]
    }




---



## TC-REG-09. Username на нижней границе (3 символа).
* **Метод:** `POST /api/auth/register`
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Register**.
    2. На вкладке **Body** ввести данные:
    ```json
    {
    "email": "border_min@example.com",
    "username": "abc",
    "password": "password123",
    "display_name": "Min User"
    }
  ```
    3. Нажать кнопку **Send**.
* **Ожидаемый результат:**
    * Статус-код: `201 Created`.
    * Пользователь успешно создан, данные в БД соответствуют отправленным.



---



## TC-REG-10. Username короче минимально допустимого (2 символа).
* **Метод:** `POST /api/auth/register`
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Register**.
    2. На вкладке **Body** ввести данные:
    ```json 
    {
    "email": "too_short@example.com",
    "username": "ab",
    "password": "password123",
    "display_name": "Short User"
    }
    ```
    3. Нажать кнопку **Send**.
* **Ожидаемый результат:**
    * Статус-код: `422 Unprocessable Entity`.
    * Response Body: Содержит ошибку валидации поля `username` с типом `string_too_short` и сообщением о минимальной длине `String should have at least 3 characters`.



---



## TC-REG-11. Username длиннее максимального (31 символ).
* **Метод:** `POST /api/auth/register`
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Register**.
    2. На вкладке **Body** ввести данные:
    ```json 
    {
    "email": "too_long@example.com",
    "username": "abcdefghijklmnopqrstuvwxyz123456",
    "password": "password123",
    "display_name": "Long User"
    }
    ```
    3. Нажать кнопку **Send**.
* **Ожидаемый результат:**
    * Статус-код: `422 Unprocessable Entity`.
    * Response Body: Содержит ошибку валидации поля `username` с типом `string_too_long` и сообщением о минимальной длине `String should have at most 30 characters`.



---



## TC-REG-12. Использование недопустимых символов в Username (Кириллица).
* **Метод:** `POST /api/auth/register`
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Register**.
    2. На вкладке **Body** ввести данные:
    ```json
    {
    "email": "cyrillic@example.com",
    "username": "тестер_001",
    "password": "password123",
    "display_name": "Cyrillic User"
    }
    ```
    3. Нажать кнопку **Send**.
* **Ожидаемый результат:**
    * Статус-код: `422 Unprocessable Entity`.
    * Response Body: Содержит ошибку валидации поля `username` с типом `string_pattern_mismatch` и сообщением о соответствии паттерну `String should match pattern '^[a-zA-Z0-9_]+$'`.



---



## TC-REG-13. Пароль на нижней границе (6 символов).
* **Метод:** `POST /api/auth/register`
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Register**.
    2. На вкладке **Body** ввести данные:
    ```json
    {
    "email": "border_password_min@example.com",
    "username": "BorderPasswordUser",
    "password": "123456",
    "display_name": "Border Password User"
    }
  ```
    3. Нажать кнопку **Send**.
* **Ожидаемый результат:**
    * Статус-код: `201 Created`.
    * Пользователь успешно создан, данные в БД соответствуют отправленным.


---



## TC-REG-14. Превышение максимальной длины Display Name (101 символ).
* **Метод:** `POST /api/auth/register`
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Register**.
    2. В поле `display_name` ввести строку из 101 символа.
    3. Нажать кнопку **Send**.
* **Ожидаемый результат:**
    * Статус-код: `422 Unprocessable Entity`.
    * Response Body: Содержит ошибку валидации поля `display_name` с типом `string_too_long` и сообщением о минимальной длине `String should have at most 100 characters`.




---



## TC-REG-15. Проверка на SQL-инъекцию в поле Email.
* **Метод:** POST /api/auth/register
* **Описание:** Попытка внедрения SQL-кода для проверки устойчивости слоя доступа к данным.
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Register**.
    2. На вкладке **Body** (тип raw/JSON) ввести данные, содержащие SQL-инъекцию в поле email:
      ```json
       {
       "email": "' OR 1=1 --",
       "username": "sql_hacker",
       "password": "password123",
       "display_name": "Hacker"
       }

    3. Нажать кнопку **Send**.
* **Ожидаемый результат:**
    * **В Postman:** 
        * Статус-код: `422 Unprocessable Entity` (ошибка формата email).
        * **Критический показатель:** Сервер НЕ должен возвращать статус `500 Internal Server Error`. Ошибка 500 указывала бы на то, что база данных попыталась исполнить вредоносный код.
    * **В Базе Данных:** 
        * Запрос `SELECT * FROM users WHERE email = '' OR 1=1 --''` не должен возвращать данных или создавать новую запись.



---



## TC-REG-16. Проверка на XSS-уязвимость в поле Display Name.
* **Метод:** POST /api/auth/register
* **Описание:** Проверка того, как сервер обрабатывает вставку исполняемого JavaScript-кода.
* **Шаги:**
    1. В коллекции **QA Sandbox API** выбрать запрос **POST Register**.
    2. На вкладке **Body** (тип raw/JSON) ввести данные, содержащие скрипт в поле `display_name`:
       ```json
       {
       "email": "xss_test@example.com",
       "username": "xss_warrior",
       "password": "password123",
       "display_name": "<script>alert('xss')</script>"
       }
    3. Нажать кнопку **Send**.
* **Ожидаемый результат:**
    * **Вариант А (Строгая валидация):** Статус-код 422 Unprocessable Entity, если бэкенд запрещает использование символов < >.
    * **Вариант Б (Санитизация/Экранирование):** Статус-код 201 Created. В этом случае при проверке через БД (SELECT) символы должны быть преобразованы в безопасные сущности (например, &lt; &gt;) или сохранены так, чтобы фронтенд не смог их исполнить.