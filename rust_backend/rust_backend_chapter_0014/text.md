# Глава 1.4. Хендлеры и ответы

Хендлер в Axum - это обычная асинхронная функция.
```rust
async fn hello() -> &'static str {
    "Hello, World!"
}
```

Фреймворк принимает такую функцию, если выполняются два условия: все её аргументы умеют извлекаться из запроса, а возвращаемое значение умеет превращаться в HTTP-ответ. Первое обеспечивают трейты `FromRequest` и `FromRequestParts` (глава 1.5), а второе трейтом `IntoResponse`. 
## Что можно вернуть из хендлера

`IntoResponse` уже реализован для большинства типов, которые могут понадобиться:

| Возвращаем               | Статус                  | content-type                |
| ------------------------ | ----------------------- | --------------------------- |
| `&'static str`, `String` | 200                     | `text/plain; charset=utf-8` |
| `Json<T>`                | 200                     | `application/json`          |
| `Html<T>`                | 200                     | `text/html; charset=utf-8`  |
| `Vec<u8>`, `Bytes`       | 200                     | `application/octet-stream`  |
| `()`                     | 200                     | тела нет                    |
| `StatusCode`             | он сам                  | тела нет                    |
| `Redirect`               | зависит от конструктора | `Location`                  |
| `Response`               | как собрали             | как собрали                 |
`content-type` выставляется самим типом возвращаемого значения, поэтому отдельно указывать его обычно не нужно. Именно так в 1.1 у `&'static str` появились заголовки в ответе `curl`, которые мы нигде не писали.
## Статус и заголовки

Всё остальное собирается кортежами. Последний элемент тело ответа, а всё перед ним статус и заголовки:
```rust
use axum::{http::{header, StatusCode}, response::IntoResponse, Json};

async fn create_user() -> (StatusCode, Json<UserResponse>) {
    (StatusCode::CREATED, Json(user))
}

async fn delete_user() -> StatusCode {
    StatusCode::NO_CONTENT
}

async fn export() -> impl IntoResponse {
    (
        [(header::CONTENT_TYPE, "text/csv")],
        "id,name\n1,Alice",
    )
}
```

Статус и заголовки необязательны, но порядок фиксированный: сначала статус, потом заголовки, последним тело. Заголовки из кортежа заменяют те, что тип тела выставил сам, поэтому в `export` вместо `text/plain` уйдёт `text/csv`.
## `impl IntoResponse`

Иногда хендлер может вернуть несколько разных вариантов ответа. В таком случае удобно указать `impl IntoResponse`:
```rust
async fn get_user() -> impl IntoResponse {
    if let Some(user) = find_user() {
        Json(user).into_response()
    } else {
        StatusCode::NOT_FOUND.into_response()
    }
}
```

Здесь разные значения приводятся к одному типу `Response`, поэтому хендлер может вернуть и JSON, и статус ошибки.
## Свой тип ответа

Трейт `IntoResponse` можно реализовать для собственного типа. Нужна одна функция, которая превращает его в `Response`:
```rust
struct Created<T>(T);

impl<T: Serialize> IntoResponse for Created<T> {
    fn into_response(self) -> Response {
        (StatusCode::CREATED, Json(self.0)).into_response()
    }
}

async fn create_user() -> Created<UserResponse> {
    Created(user)
}
```

Такой подход полезен, когда один и тот же формат ответа используется в нескольких хендлерах.
