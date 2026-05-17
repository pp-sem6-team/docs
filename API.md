# Cпецификация API

## Общая информация

Бэкенд реализован на Go с использованием фреймворка Gin и предоставляет REST API для взаимодействия фронтенда с системой анализа кожи.

### Base URL

В среде разработки сервер доступен по адресу:

http://localhost:8080

Все эндпоинты указываются относительно этого адреса.

### Формат данных

Основной формат передачи данных - JSON.

Content-Type запросов и ответов:

application/json

Исключение составляет загрузка изображений, для которой используется формат:

multipart/form-data

### Коды ответов

API использует стандартные HTTP-коды ответа:

- 200 OK - запрос выполнен успешно  
- 201 Created - ресурс создан  
- 202 Accepted - запрос принят в обработку  
- 204 No Content - успешное выполнение без тела ответа  
- 400 Bad Request - некорректный запрос  
- 401 Unauthorized - требуется авторизация
- 403 Forbidden - доступ запрещён
- 404 Not Found - ресурс не найден
- 409 Conflict - конфликт (ресурс уже существует)
- 500 Internal Server Error - внутренняя ошибка сервера
- 503 Service Unavailable - недоступны инфраструктурные зависимости (DB / Storage)

### Взаимодействие сервисов

Архитектура сервиса включает три основных компонента:

Frontend -> Backend (Gin) -> ML-сервис

Frontend взаимодействует только с backend API.  
Backend обрабатывает запросы пользователей, управляет данными и при необходимости отправляет изображения в ML-сервис для анализа.  
ML-сервис выполняет обработку изображения и возвращает результаты анализа бэкенду.


## Структуры данных

### User
    {
        "id": "UUID",
        "email": "user@example.com",
        "name": "string",
        "birth_date": "YYYY-MM-DD" | null,
        "gender": "male" | "female" | null,
        "created_at": "2026-01-01T12:00:00Z",
        "updated_at": "2026-01-01T12:00:00Z"
    }

### Photo
    {
        "id": "UUID",
        "user_id": "UUID",
        "file_url": "string",
        "uploaded_at": "2026-01-01T12:00:00Z"
    }

### Analysis
    {
        "id": "UUID",
        "photo_id": "UUID",
        "status": "processing" | "completed" | "failed",
        "skin_type": "oily" | "dry" | "normal" | "combination" | null, // null пока анализ не завершён
        "analysis_data": { ... } | null,                               // null пока анализ не завершён
        "created_at": "2026-01-01T12:00:00Z",
        "updated_at": "2026-01-01T12:00:00Z"
    }

### Recommendation
    {
        "id": "UUID",
        "skin_type": "oily" | "dry" | "normal" | "combination",
        "title": "string",
        "description": "string"
    }

### Ingredient
    {
        "id": "UUID",
        "name": "string",
        "description": "string"
    }


## Эндпоинты

### Список всех эндпоинтов
    POST   /auth/register
    POST   /auth/login
    POST   /auth/logout
    POST   /auth/refresh

    GET    /users/me
    PATCH  /users/me
    PATCH  /users/me/password
    DELETE /users/me

    POST   /analyses
    GET    /analyses
    GET    /analyses/{id}
    DELETE /analyses/{id}

    GET    /health
    GET    /health/db
    GET    /health/storage
    GET    /health/ml


### POST /auth/register
Регистрация нового пользователя.

#### Request
    {
        "email": "user@example.com",
        "password": "string",
        "name": "John Doe",
        "birth_date": "2000-01-01",  // необязательное
        "gender": "male" | "female" | null // необязательное
    }

#### Response
    201 Created
    {
        "access_token": "string",
        "refresh_token": "string",
        "token_type": "Bearer"
    }


### POST /auth/login
Авторизация пользователя.

#### Request
    {
        "email": "user@example.com",
        "password": "string"
    }

#### Response
    200 OK
    {
        "access_token": "string",
        "refresh_token": "string",
        "token_type": "Bearer"
    }


### POST /auth/logout
Выход пользователя из системы.

#### Request
    Authorization: Bearer access_token
    {
        "refresh_token": "string"
    }

#### Response
    200 OK
    {
        "message": "successfully logged out"
    }


### POST /auth/refresh
Получение новой пары токенов.

#### Request
    {
        "refresh_token": "string"
    }

#### Response
    200 OK
    {
        "access_token": "string",
        "refresh_token": "string",
        "token_type": "Bearer"
    }


### GET /users/me
Получение информации о текущем пользователе.

#### Request
    Authorization: Bearer access_token

#### Response
    200 OK
    {
        "id": "UUID",
        "email": "user@example.com",
        "name": "John Doe",
        "birth_date": "2000-01-01" | null,
        "gender": "male" | "female" | null,
        "created_at": "2026-01-01T12:00:00Z",
        "updated_at": "2026-01-01T12:00:00Z"
    }


### PATCH /users/me
Обновление данных пользователя.

#### Request
    Authorization: Bearer access_token
    {
        "email": "user@example.com",        // необязательное
        "name": "John Doe",                 // необязательное
        "birth_date": "2000-01-01",         // необязательное
        "gender": "male" | "female" | null  // необязательное
    }

#### Response
    200 OK
    {
        "id": "UUID",
        "email": "user@example.com",
        "name": "John Doe",
        "birth_date": "2000-01-01" | null,
        "gender": "male" | "female" | null,
        "created_at": "2026-01-01T12:00:00Z",
        "updated_at": "2026-01-01T12:00:00Z"
    }


### PATCH /users/me/password
Обновление пароля пользователя.

#### Request
    Authorization: Bearer access_token
    {
        "current_password": "string",
        "new_password": "string"
    }

#### Response
    204 No Content


### DELETE /users/me
Удаление аккаунта пользователя.

#### Request
    Authorization: Bearer access_token

#### Response
    204 No Content


### POST /analyses
Загрузка фотографии для анализа кожи.

#### Request
    Authorization: Bearer access_token

    multipart/form-data:

    photo: file

#### Response
    202 Accepted
    {
        "id": "UUID",        // id анализа
        "photo_id": "UUID",  // id фото
        "file_url": "https://example.com/photos/uuid.jpg", // ссылка на фото
        "status": "processing",
        "uploaded_at": "2026-01-01T12:00:00Z"
    }


### GET /analyses
Получение истории анализов пользователя.

#### Request
    Authorization: Bearer access_token
    Query:
        offset (int, default: 0)
        limit  (int, default: 10, max: 100)

#### Response
    200 OK
    [
        {
            "id": "UUID",
            "photo_id": "UUID",
            "file_url": "https://example.com/photos/uuid.jpg",
            "status": "processing" | "completed" | "failed",
            "skin_type": "oily" | "dry" | "normal" | "combination" | null,
            "created_at": "2026-01-01T12:00:00Z",
            "updated_at": "2026-01-01T12:30:00Z"
        }
    ]


### GET /analyses/{id}
Получение результата конкретного анализа.

#### Request
    Authorization: Bearer access_token
    Query:
        recs_limit (int, default: 10)
        ings_limit (int, default: 10)

#### Response
    200 OK
    {
        "id": "UUID",
        "photo_id": "UUID",
        "file_url": "https://example.com/photos/uuid.jpg",
        "status": "processing" | "completed" | "failed",
        "skin_type": "oily" | "dry" | "normal" | "combination" | null,
        "analysis_data": { ... },          // содержимое JSONB из таблицы analyses
        "recommendations": [               // присутствует только если status = completed
            {
                "id": "UUID",
                "title": "Use gentle cleanser",
                "description": "Avoid aggressive cleansing products."
            }
        ],
        "ingredients": [                   // присутствует только если status = completed
            {
                "id": "UUID",
                "name": "Salicylic Acid",
                "description": "Helps unclog pores and reduce acne."
            }
        ],
        "created_at": "2026-01-01T12:00:00Z",
        "updated_at": "2026-01-01T12:30:00Z"
    }


### DELETE /analyses/{id}
Удаление анализа пользователя.

#### Request
    Authorization: Bearer access_token

#### Response
    204 No Content


### GET /health
Проверка состояния сервиса.

#### Request
    none

#### Response
    200 OK
    {
        "status": "ok"
    }

### GET /health/db
Проверка состояния базы данных.

#### Request
    none

#### Response
    200 OK
    {
        "status": "db ok"
    }

### GET /health/storage
Проверка состояния объектного хранилища.

#### Request
    none

#### Response
    200 OK
    {
        "status": "storage ok"
    }


### GET /health/ml
Проверка состояния ML-сервиса.

#### Request
    none

#### Response
    200 OK
    {
        "status": "ml ok"
    }
